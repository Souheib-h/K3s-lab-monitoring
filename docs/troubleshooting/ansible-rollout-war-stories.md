# Troubleshooting: Ansible rollout & Prometheus↔K3s integration

Every issue hit while building Phase 5 (Ansible automation) and the
Prometheus→K3s scraping pipeline, in the order encountered. Kept together
because most were discovered while chasing a different symptom than the one
that turned out to matter. Format: symptom → diagnosis → root cause → fix.

---

## 1. Ansible hangs silently on specific hosts: asymmetric routing

**Symptom.** `ansible -m ping` succeeded on 11/13 hosts. On `k3s-db` and
`load-srv`, Ansible hung indefinitely, no error, no timeout. Yet manually:

```
nc -zvw3 10.10.0.20 22        # TCP OK
ping -c 2 10.10.0.20          # ICMP OK
ssh user@10.10.0.20 'echo OK' # auth + exec OK
```

**Diagnosis.** A comparative gather script (OS, sshd, python, disk, perms, rc
files, routes) run against one working host and both failing hosts turned up
exactly one difference: the route back to `10.20.0.0/24` was present on the
working host and absent on both failing ones, missed by the Phase 1 netplan
rollout, which had only covered the 6 K3s nodes.

**Root cause.** Forward path: ansible-srv (10.20.0.x) → OPNsense → target.
Works. Return path: with no route to 10.20.0.0/24, the target's reply exits
via its default gateway (the libvirt host, 10.10.0.1) instead of OPNsense.
The libvirt host knows both bridges, so tiny exchanges (ICMP, SSH handshake,
short commands) survive the asymmetric path, but Ansible pushes its module
payload over SFTP (~100 KB), and the sustained flow dies on the asymmetric
path while the session still reports "established". Hence: ping and SSH
succeed, Ansible hangs.

**Fix.**

```yaml
# Ubuntu: /etc/netplan/99-routes.yaml
network:
  version: 2
  ethernets:
    enp1s0:
      routes:
        - { to: 10.20.0.0/24, via: 10.10.0.254, on-link: true }
```
```
sudo chmod 600 /etc/netplan/99-routes.yaml && sudo netplan apply
```

```
# Alpine: immediate + persistent in /etc/network/interfaces under eth0
sudo ip route add 10.20.0.0/24 via 10.10.0.254
#   post-up ip route add 10.20.0.0/24 via 10.10.0.254
```

**Takeaway.** Ping OK + SSH OK + Ansible hang → suspect asymmetric routing or
a bulk transfer, not connectivity. A single comparative gather script across
a working and a failing host finds this kind of diff in one pass.

---

## 2. Fix applied, Ansible still hangs: stale ControlMaster socket

**Symptom.** After adding the missing route, `ansible k3s-db -m ping` still
hung.

**Root cause.** Ansible multiplexes SSH via ControlMaster sockets
(`~/.ansible/cp/`) with `ControlPersist`. The stale master kept the dead
connection alive; the new return path is only used by *new* connections.

**Fix.**
```
rm -f ~/.ansible/cp/*
```

**Takeaway.** A network fix does not revive existing connections; always
purge the control socket cache after a routing change before retesting.

---

## 3. `become` fails with a non-standard sudo prompt

**Symptom.** `ansible all -m command -a whoami --become` failed on several
hosts (k3s-srv-3, the 4 monitoring VMs) with either:
```
sudo: interactive authentication is required
Missing sudo password
Timeout (12s) waiting for privilege escalation prompt
```
even after supplying `-K` with the correct password.

**Diagnosis.** A manual `ssh -t ... 'sudo -v'` on the failing host showed the
prompt text itself: `[sudo: authenticate] Password:` instead of the standard
`[sudo] password for <user>:`. Ansible's become plugin pattern-matches the
prompt text to know when to inject the password, a non-standard prompt (seen
on hosts created later / with a newer sudo build) is never recognized, so
Ansible waits forever regardless of whether the password is correct.

**Fix.** Bypass become detection entirely: deploy the NOPASSWD sudoers file
directly over a plain interactive SSH session (which doesn't depend on
Ansible's prompt matching):
```
ssh -t -i ~/.ssh/ansible <user>@<ip> \
  'echo "<user> ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/<user> \
   && sudo chmod 440 /etc/sudoers.d/<user>'
```
Once NOPASSWD is in place there is no prompt left to mismatch.

**Takeaway.** "Timeout waiting for privilege escalation prompt" is not
necessarily a wrong password; check the literal prompt text first.

---

## 4. Inventory YAML: a host silently dropped from its group

**Symptom.** `[WARNING]: Skipping unexpected key (load-srv) in group
(load_balancer)` and the host missing from `ansible-inventory --graph` and
from every playbook run, twice, independently, for the same host.

**Root cause.** Both times, the host entry was indented at the same level as
`hosts:` instead of one level deeper underneath it:
```yaml
# wrong
    load_balancer:
      hosts:
      load-srv: { ansible_host: 10.10.0.10 }
```
YAML parses `hosts:` as an empty mapping and `load-srv` as an unrelated
sibling key of the group, which Ansible then rejects as "unexpected".

**Fix.**
```yaml
    load_balancer:
      hosts:
        load-srv: { ansible_host: 10.10.0.10 }
```

**Related trap, duplicate `hosts:` key.** Pasting a new group's block
*inside* an existing group's YAML body (rather than as a sibling) produces
two `hosts:` keys in the same mapping; YAML silently keeps only the last one
and drops everything under the first, an entire group (four monitoring VMs)
disappeared from the graph with only a generic
`found a duplicate dict key (hosts)` warning as a clue.

**Takeaway.** After any inventory edit, always run
`ansible-inventory --graph` before the actual playbook; it is the cheapest
way to catch a host or group that silently vanished.

---

## 5. `group_vars/secrets.yml` never loads

**Symptom.** `'zabbix_api_token' is undefined`, despite the file existing
with the right content at `inventory/group_vars/secrets.yml`.

**Root cause.** `group_vars/<name>.yml` is loaded **per matching group name**.
Only `all.yml` matches the built-in `all` group automatically; a file named
anything else at that level (e.g. `secrets.yml`) matches no group and is
silently ignored.

**Fix.** Group multiple files under a directory named after the group instead; every file inside is loaded for that group:
```
inventory/group_vars/all/main.yml       # non-secret vars
inventory/group_vars/all/secrets.yml    # gitignored
```

**Takeaway.** `group_vars/<group>/*.yml` (directory form) is the right
pattern as soon as you want to split committed vars from secrets.

---

## 6. `community.zabbix` v3: two breaking changes from v1/v2 tutorials

**Symptom A.** `AssertionError: socket_path must be a value`, using the
"classic" pattern (`connection: local` + credentials passed via
`environment:`).

**Root cause A.** Since v3, `community.zabbix` modules talk to the API
through the **httpapi connection plugin**, not environment variables. The
play must target the Zabbix host itself with `ansible_connection: httpapi`
and the auth key as a play var:
```yaml
- hosts: zabbix-srv
  gather_facts: false
  vars:
    ansible_network_os: community.zabbix.zabbix
    ansible_connection: httpapi
    ansible_httpapi_port: 80
    ansible_httpapi_use_ssl: false
    ansible_zabbix_url_path: zabbix
    ansible_zabbix_auth_key: "{{ zabbix_api_token }}"
```

**Symptom B.** `Hostgroup not found: Monitoring stack` / `K3s cluster` for
every host in the loop, even though the API call itself was authenticating
fine.

**Root cause B.** Unlike v2, `zabbix_host` in v3 no longer auto-creates
missing host groups.

**Fix B.** Create the groups explicitly first, in a dedicated task:
```yaml
- name: Ensure host groups exist
  community.zabbix.zabbix_group:
    host_groups: ["K3s cluster", "Monitoring stack"]
    state: present
```

**Takeaway.** Community collection major versions can change the connection
model, not just module parameters; check the target version's docs rather
than reusing an older example verbatim.

---

## 7. Alpine's Wazuh package ships no init script: twice

**Symptom.** `Error when trying to add zabbix-agent: rc=1 * rc-update:
service 'zabbix-agent' does not exist`, then, after fixing that, the
identical error for `wazuh-agent`.

**Root cause.** OpenRC service names on Alpine are suffixed `-d`
(`zabbix-agentd`), unlike the systemd unit name used by the Debian package; first gotcha. Second, deeper gotcha: the Wazuh **apk** package installs
`/var/ossec/bin/wazuh-control` but ships **no OpenRC init script at all**
(`ls /etc/init.d/ | grep wazuh` returns nothing), while Ubuntu's `.deb` provides
one, Alpine's `.apk` does not.

**Fix.** A custom OpenRC script wrapping `wazuh-control`, deployed by the
playbook itself (`files/wazuh-agentd.initd` → `ansible.builtin.copy` →
`/etc/init.d/wazuh-agentd`, `mode: "0755"`), then `service: {name:
wazuh-agentd, state: started, enabled: true}`.

**Takeaway.** Never assume a community/vendor package on a minority distro
ships the same integration files as the mainstream one, verify
`/etc/init.d/` (or the systemd equivalent) exists before writing a `service:`
task.

---

## 8. The custom init script breaks idempotence

**Symptom.** Every single run of `install-agents.yml` reported
`changed: [ansible-srv]` / `changed: [load-srv]` on the "Enable and start
wazuh-agent" task, even when the agent was already running and nothing had
changed.

**Root cause.** The initial `status()` function in the custom init script
just proxied `wazuh-control status` verbatim (which prints five lines, one
per daemon) instead of returning a clean 0/1 exit code. OpenRC's service
module couldn't parse that as "running", so it treated the service as
stopped and restarted it on every run.

**Fix.**
```sh
status() {
    if /var/ossec/bin/wazuh-control status | grep -q "wazuh-agentd is running"; then
        einfo "wazuh-agent is running"; return 0
    else
        eerror "wazuh-agent is stopped"; return 1
    fi
}
```

**Takeaway.** A custom init script's `status()` is what idempotence is built
on, if it doesn't return a real exit code, every playbook run "changes"
something that never actually changed.

---

## 9. Host freeze: RAM overcommit across 13 simultaneous VMs

**Symptom.** The Arch host completely froze mid-rollout; no SysRq response
(magic SysRq keys disabled by default on Arch).

**Diagnosis.** `free -h` before the freeze (checked after recovery) showed
the trajectory: `available` dropping from several GiB to near 0 with swap
climbing, classic swap-thrashing on a Btrfs swapfile (slow under sustained
pressure). An audit of every VM's `Max memory` totalled **46 GiB allocated
for 31 GiB physical**, an ~50% overcommit that had been tolerable only as
long as guests were started in small groups; a full-inventory Ansible run
woke every guest's memory pressure at once.

**Fix.** Hard shutdown (safe: Btrfs is copy-on-write and survives a power
cut with `btrfs device stats` reporting zero errors afterward; qcow2 guests
replay their journal on boot). Rebalanced allocations down to ~33 GiB
(agents 4→2 GiB, k3s-db 4→2 GiB, Prometheus 4→2 GiB, Zabbix 4→3 GiB,
OPNsense 4→2 GiB, load-srv 2→1 GiB, Ansible control node 2→1 GiB), then
rebooted the lab in stages (router → k3s infra → control-plane → workers →
monitoring → control node) rather than all at once.

**Takeaway.** KVM overcommit tolerates guests that don't touch their full
allocation *most of the time*; it does not tolerate a coordinated wake-up
(a full-fleet Ansible run) hitting a 50% overcommit simultaneously. Audit
`virsh dominfo` totals against physical RAM before scaling the number of
concurrently-running VMs, not after a freeze.

---

## 10. kube-state-metrics: the standard Service can't become a NodePort

**Symptom.**
```
The Service "kube-state-metrics" is invalid: spec.clusterIPs[0]: Invalid
value: "None": may not be set to 'None' for NodePort services
```
while trying to `kubectl patch` the upstream Service to `type: NodePort`.

**Root cause.** The official manifest creates a **headless** Service
(`clusterIP: None`), designed for in-cluster discovery. A headless Service's
`ClusterIP` field cannot be converted by a patch, it's a structural
constraint, not a permissions issue.

**Fix.** Leave the original headless Service untouched; add a second Service
selecting the same pod labels, `type: NodePort`, for external exposure. Two
doors, one pod behind both.

**Takeaway.** When a Prometheus instance lives outside the cluster it's
scraping, exposing an in-cluster-oriented Service usually means adding an
external Service rather than converting the existing one.

---

## 11. `kubectl create clusterrole` rejects mixed verb types

**Symptom.** `error: invalid verb: 'list' for nonResourceURL` when trying to
create one ClusterRole covering both a non-resource URL (`/metrics`) and a
resource (`nodes/metrics`) in a single imperative command.

**Root cause.** `kubectl create clusterrole --non-resource-url=... --verb=...
--resource=... --verb=...` applies the *last* `--verb` flag set to *both*
rule types; `list`/`watch` are invalid for non-resource URLs, which only
support `get`.

**Fix.** Declare the two rules explicitly in YAML instead of the imperative
shortcut:
```yaml
rules:
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["nodes", "nodes/metrics", "nodes/proxy"]
    verbs: ["get", "list", "watch"]
```
The earlier imperative attempt had also left a ClusterRoleBinding pointing at
a ClusterRole that didn't actually exist with the intended rules; worth
double-checking with `kubectl auth can-i ... --as=system:serviceaccount:...`
after any RBAC change.

**Takeaway.** `kubectl create clusterrole` is fine for a single rule type;
mixed resource / non-resource permissions need a YAML manifest.

---

## 12. Prometheus reload rejected by systemd

**Symptom.** `Failed to reload prometheus.service: Job type reload is not
applicable for unit prometheus.service.`

**Root cause.** The systemd unit has no `ExecReload=` directive, so
`systemctl reload` has nothing to call even though the config itself was
valid (`promtool check config` passed).

**Fix.** `sudo systemctl restart prometheus` instead; a few seconds of
scrape gap, no data loss to the TSDB.

**Takeaway.** `promtool check config` validates syntax, not whether the
running process can pick up the change without a full restart, check the
unit file's reload support before assuming `reload` will work.

---

## 13. A duplicated scrape target doubles every metric

See the dedicated writeup in `docs/10-prometheus-k3s-scraping.md` §5, kept
there rather than duplicated here since it's part of the main scraping
narrative rather than a one-off fix.
