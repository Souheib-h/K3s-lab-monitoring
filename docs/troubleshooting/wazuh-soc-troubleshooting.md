# Wazuh SOC: Troubleshooting Reference

This file documents every issue encountered while building the SOC layer (see [08-soc-configuration.md](../phase-4-soc/README.md)). Each entry follows the same structure: symptom → diagnosis → root cause → fix. The `TS-x` identifiers are referenced from the main procedure.

Most of these issues share a nasty property: **no component ever reports an error.** Feeds say "completed", scans say "finished", connectors say "initialized", the indexer answers "errors: false", and the dashboard stays empty. Diagnosis was a process of eliminating layers one by one.

---

## TS-1: Vulnerability Detection returns zero results on an all-in-one deployment

**Symptom.** Vulnerability Detection dashboard empty ("No results"). `wazuh-states-vulnerabilities-*` index exists but holds 0 documents. Zero CVE alerts in `alerts.json`. The feed downloads correctly, every scan reports success, no error appears in any log.

**Diagnosis.** With `wazuh_modules.debug=2`, the scanner's startup config dump revealed:

```
"managerDisabledScan": 1
...
DEBUG: Vulnerability scanner in manager still disabled
```

**Root cause.** The internal option `vulnerability-detection.disable_scan_manager` defaults to `1`: Wazuh does not scan the manager host itself. On an all-in-one deployment, the manager (agent 000) is the *only* monitored host, so the scanner had an empty work list. Every re-scan evaluated nobody, matched nothing, wrote nothing, and reported success.

**Fix.**

```bash
echo "vulnerability-detection.disable_scan_manager=0" | sudo tee -a /var/ossec/etc/local_internal_options.conf
sudo systemctl restart wazuh-manager
```

The manager immediately logs `Policy changed. Performing re-scan over the manager.` and the first real evaluation runs, 1,875 findings on this host.

**Lesson.** This single default cost ~20 hours of investigation across three reinstalls and an OS upgrade, because its effect (empty results) is indistinguishable from half a dozen pipeline failures. When a scanner reports success but produces nothing, check *what it is scanning* before checking *how it writes results*.

---

## TS-2: First-boot race: indexer connectors fail during the initial inventory sync

**Symptom.** All `wazuh-states-inventory-*` indices permanently at 0 documents, even though syscollector collects correctly (verified in the local SQLite delta DB: 761 packages in `dbsync_packages`). Debug logs later showed the connector only ever issuing DELETE operations, never INSERT.

**Diagnosis.** First-boot logs from the quickstart install:

```
11:15:32  syscollector: Starting evaluation          ← first scan (the full sync)
11:15:33  Evaluation finished
11:15:33  indexer-connector: WARNING: initialization failed ... retrying   ← ×14
11:16:47  indexer-connector: INFO: initialized successfully   ← 74 seconds too late
```

**Root cause.** The installation assistant starts the manager *before* configuring the indexer's internal users. During the manager's first syscollector scan, the connectors cannot authenticate; the initial full sync is emitted into the void and is not replayed. All subsequent scans only push *deltas*, and since nothing changes on a fresh host, the gap never closes. The manager-side sync state (`queue/db/000.db`) believes everything is synced; the indexer never received anything.

**Fix (procedure).** After install, wait for the indexer cluster to report green, then restart the manager once. The connectors initialize in seconds and the scan_on_start full sync lands correctly.

**Fix (recovery on an already-affected system).** Deleting the delta databases alone is not sufficient (the comparator DB re-marks everything as known). A full state reset is required, with the manager fully stopped:

```bash
sudo systemctl stop wazuh-manager
ps aux | grep wazuh-modulesd   # must be empty
sudo rm -f  /var/ossec/queue/db/000.db
sudo rm -f  /var/ossec/queue/syscollector/db/local.db
sudo rm -rf /var/ossec/queue/indexer/*      # see TS-6 about globs here
sudo systemctl start wazuh-manager
```

---

## TS-3: Interrupted feed download leaves a "phantom feed"

**Symptom.** `vd_updater/` at ~40 MB when a fresh feed should peak in the gigabytes; subsequent feed cycles "complete" in ~4 minutes instead of ~15–30; re-scans match nothing.

**Diagnosis.** Timeline reconstruction: the manager was stopped at 16:54 while a feed download was at 1.1 GB (mid-ingestion). The next cycle completed suspiciously fast and reported success.

**Root cause.** Stopping the manager during the download/ingestion phase leaves the RocksDB metadata in `vd_updater/` claiming the feed is current. The next cycle performs a delta check against this phantom state, downloads nothing, and the scanner silently matches packages against an incomplete CVE base.

**Fix.**

```bash
sudo systemctl stop wazuh-manager
# verify no wazuh-modulesd process remains
sudo rm -rf /var/ossec/queue/vd_updater/rocksdb /var/ossec/queue/vd_updater/tmp
sudo systemctl start wazuh-manager
# a full ~15-30 min cycle follows: do not touch the manager until "completed"
```

**Recognition markers.**

| Healthy cycle | Phantom feed |
|---|---|
| 15–30 min duration | "completed" in minutes |
| Peak ~1 GB then ~4.5 GB during ingestion | never grows |
| ~40 MB at rest **after a full cycle** | ~40 MB *without* a full cycle ever happening |

Note the trap: 40 MB at rest is **normal** after a successful cycle (compacted database). Size alone does not distinguish healthy from phantom, cycle duration does.

**Rule adopted.** Between `Initiating update feed process` and `Feed update process completed`, the manager is untouchable: no systemctl, no apt, no config edits.

---

## TS-4: Active response config silently inert inside an XML comment

**Symptom.** Active response configured (`firewall-drop`, rules 5712/5720/5763) but never triggering. No error anywhere.

**Root cause.** The default `ossec.conf` ships its active-response section wrapped in an XML comment (`<!-- ... -->`) as a placeholder. Editing the values *inside* the comment produces a syntactically valid file where the entire block is ignored.

**Fix.** Move/rewrite the block outside any comment, then verify:

```bash
sudo grep -n 'active-response\|<!--\|-->' /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-manager
```

Validated afterwards with a real SSH brute force from another VM, detection → `firewall-drop` executed automatically.

---

## TS-5: wazuh-maild cannot send to Gmail (no auth, no TLS)

**Symptom.** Email alerting configured, alerts matching the level threshold, but no email ever sent. Manager log:

```
wazuh-maild: CRITICAL: (1265): Invalid SMTP Server: smtp.example.wazuh.com
```

**Root cause (two layers).** First, the `<smtp_server>` tag still held its placeholder value. Second, the structural problem, `wazuh-maild` is a minimal SMTP client with **no SMTP authentication and no TLS/STARTTLS support**. It cannot talk to Gmail or any modern provider directly, and it does not use the system's sendmail (so an msmtp setup is bypassed entirely).

**Fix.** Local Postfix relay on loopback: maild speaks plain SMTP to `127.0.0.1`, Postfix relays to `smtp.gmail.com:587` with SASL auth (Gmail App Password) and TLS. Full setup in [08-soc-configuration.md](../phase-4-soc/README.md) section 5. Delivery confirmed end-to-end.

---

## TS-6: `rm -rf dir/*` silently deletes nothing on root-owned directories

**Symptom.** During TS-2/TS-3 recovery: `sudo rm -rf /var/ossec/queue/indexer/*` returned without error, but `ls` showed all 15 directories still present.

**Root cause.** The glob `*` is expanded by the **user's shell**, not by root. When the directory is not readable by the invoking user (`drw-rw---- root wazuh`), zsh finds nothing to expand and `rm` receives no arguments (or the literal pattern). No error is raised.

**Fix.** Either name the targets explicitly (`sudo rm -rf /path/dir1 /path/dir2 ...`), or run the expansion as root: `sudo sh -c 'rm -rf /var/ossec/queue/indexer/*'`. Always verify with `sudo ls -la` afterwards, a lesson that applies to every state-purge operation in this file.

**Related, FIM realtime watches only existing directories.** syscheck registers realtime (inotify) watches on monitored directories that exist **at startup**. A directory created after the last manager restart is not watched, and no FIM event fires on it, with no error. If a newly-created monitored path produces no alerts, restart the manager once so the watch is registered.

---

## TS-7: `<integration>` block placement crashes the manager

**Symptom.** After adding the VirusTotal integration, the manager refuses to start:

```
wazuh-manager.service failed
wazuh-testrule: ERROR: (1230): Invalid element in the configuration: 'integration'.
wazuh-testrule: ERROR: (1202): Configuration error at 'etc/ossec.conf'.
```

**Root cause.** `ossec.conf` can contain **multiple** `<ossec_config>` blocks (this install ends with a separate one holding the `<localfile>` entries). The `<integration>` block was placed outside any `<ossec_config>` container, Wazuh does not recognise it there.

**Fix.** `<integration>` must be a direct child of `<ossec_config>`, a sibling of `<syscheck>` / `<global>`, never inside another block. Validate the config **without** starting the service before restarting:

```bash
sudo /var/ossec/bin/wazuh-logtest-legacy 2>&1 | head -5   # must show no configuration error
sudo systemctl restart wazuh-manager
```

**Lesson.** Any edit to `ossec.conf` should be validated with `wazuh-logtest-legacy` before the restart. It parses the config and reports errors without bringing the service down, which matters because a bad config leaves the whole SOC offline.

---

## TS-8: Kernel modules unavailable after a host system upgrade (Arch/KVM host)

**Symptom.** After a routine `pacman -Syu` on the KVM host, no VM would start:

```
error: Unable to open /dev/net/tun, is tun module loaded?: No such device
```

**Root cause.** The kernel upgrade removed the running kernel's modules from `/lib/modules/`; the system was still executing the old kernel, whose `tun` module no longer existed on disk. libvirt could not create TAP interfaces.

**Fix.** Reboot. Rule adopted on the host: after any kernel upgrade, reboot before doing anything that loads kernel modules (tun, kvm, vfio...).

---

## Appendix: What made this investigation hard

Four independent issues (TS-1 through TS-4) were present simultaneously and masked one another: the XML comment hid the active-response config, the interrupted feed hid the scanner's real behavior, the connector race emptied the inventory indices, and underneath it all `disable_scan_manager` guaranteed zero results regardless of everything else. Fixing any single layer changed nothing observable, which repeatedly invalidated correct partial diagnoses.

The methodological takeaways:

1. **One change at a time, timestamp everything.** Entangled restarts/purges/upgrades made cause-and-effect unreadable on day one.
2. **Silent success is a diagnostic dead end, go to debug level.** `wazuh_modules.debug=2` in `local_internal_options.conf` is what surfaced both the connector behavior (DELETE-only) and the final root cause (`managerDisabledScan: 1`).
3. **Validate what a component consumes before debugging what it produces.** The scanner's empty *input* (no hosts to scan) was checked last; it should have been checked first.
4. **Time-box exploratory debugging.** The decision rule "if state purges don't fix it, the state was never the problem" would have saved a reinstall and an OS upgrade.
