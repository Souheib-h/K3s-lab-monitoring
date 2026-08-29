# Architecture Decision Records

This document captures the key architectural decisions made throughout the `K3s-lab-monitoring` project, not _what_ was built, but _why_ it was built that way. Each ADR answers: what was the context, what was decided, what alternatives were considered, and what are the consequences.

---

## ADR-001: Two isolated networks instead of one flat network

**Status:** Decided

### Context

The lab runs a K3s cluster (3 control planes, 3 workers, 1 LB, 1 DB) alongside a dedicated monitoring stack (Wazuh, Prometheus, Grafana, Zabbix). Both sets of VMs could have shared a single libvirt network.

### Decision

Two separate networks:

- `k3s-net` (10.10.0.0/24), cluster traffic
- `monitoring-net` (10.20.0.0/24), monitoring stack

Routed through OPNsense.

### Why

- Security boundary: monitoring VMs can observe the cluster without being part of it. If a monitoring tool is compromised, it doesn't have L2 access to the cluster nodes.
- Realistic architecture: production monitoring infrastructure is always network-segregated from what it monitors.
- Traffic isolation: Prometheus scrape traffic, Wazuh agent communication, and Zabbix polling don't pollute cluster inter-node traffic.

### Alternatives rejected

- Single flat network: simpler, but no security boundary and not representative of real environments.
- VLANs on a single bridge: overkill for a KVM lab, harder to manage without physical switching.

---

## ADR-002: OPNsense as the inter-network router

**Status:** Decided

### Context

With two isolated networks, something needs to route between them. Options ranged from a simple kernel IP forwarding setup to a full firewall VM.

### Decision

OPNsense 26.1 deployed as a dedicated VM with:

- WAN interface on `k3s-net` (10.10.0.254)
- LAN interface on `monitoring-net` (10.20.0.254)

### Why

- Visibility: OPNsense provides packet capture, firewall logs, and traffic graphs. Proved invaluable during debugging (confirmed inter-network routing with live packet captures showing TTL decrements and MAC translation at the L3 boundary).
- Realistic SOC context: Wazuh and Grafana should ideally receive OPNsense syslog as a data source. Having a real firewall in the path makes this possible.
- Learning value: OPNsense config (NAT outbound disable, block private networks off on WAN) is directly transferable to real deployments.

### Alternatives rejected

- iptables forwarding on the host: works, but no visibility, no logs, not a realistic setup.
- libvirt routed mode: attempted, failed due to masquerade chain (`LIBVIRT_PRT`) intercepting traffic before custom routing rules. Required adding `RETURN` rules to exempt inter-network traffic, ultimately replaced by OPNsense for cleaner separation.

### Notable fix

libvirt's nftables/iptables masquerade chain (`LIBVIRT_PRT`) rewrites source IPs before custom routing can work. Fix: switch libvirt firewall backend to `iptables`, add `RETURN` rules to exempt inter-network traffic, disable outbound NAT on OPNsense.


### Update  outbound NAT: Disable is too broad, Hybrid is correct

**Status:** Revised (2026-07-23)

Disabling OPNsense's outbound NAT entirely (to stop it rewriting source addresses for the inter-network routing above) had an unintended side effect: it also stopped NAT for traffic actually leaving the lab. Any VM on `k3s-net`, `monitoring-net`, or `bastion-net` lost internet access  discovered when a fresh `Loki-srv` VM could reach every other lab network but timed out on `apk update`.

**Fix:** switch outbound NAT mode from **Disable** to **Hybrid** (Firewall > NAT > Outbound). Hybrid preserves the manual/system rules needed for inter-network routing while still auto-generating NAT for everything else, including WAN egress.

Verified both directions still work after the change:

- `ping 8.8.8.8` from a lab VM internet reachable
- `ping 10.10.0.254` from monitoring-net , inter-network routing via OPNsense still intact

**Lesson:** a fix scoped to "stop rewriting addresses between our own networks" should be scoped that narrowly disabling a whole subsystem (outbound NAT) to solve a specific symptom (source rewriting) reaches further than intended. Hybrid mode existed for exactly this reason and should have been the first choice.

---

## ADR-003: Wazuh all-in-one for SOC

**Status:** Decided

### Context

Wazuh has three components: indexer (OpenSearch-based), manager, and dashboard. They can be deployed separately (distributed) or co-located (all-in-one).

### Decision

All-in-one deployment on a single `Wazuh-srv` VM (8GB RAM, 4 vCPU).

### Why

- Scale: the lab monitors fewer than 20 endpoints, well within the all-in-one supported range (up to 100 agents).
- Simplicity: distributed deployment adds cert complexity, inter-component networking, and failure domains that aren't justified at this scale.
- Resource efficiency: a single 8GB VM handles all three components comfortably.

### Alternatives rejected

- Distributed deployment: appropriate for production at scale, overkill here.
- Wazuh Cloud: defeats the purpose of a self-hosted SOC lab.

---

## ADR-004: Zabbix agent for host metrics, Prometheus for K8s/application metrics

**Status:** Decided (revised)

### Context

The NOC stack needs metrics at two levels: host-level (CPU, RAM, disk, network) and K8s/application-level (pod status, container metrics, application counters).

### Decision

- Zabbix agent: host-level metrics on all monitored VMs. Already collected and stored in Zabbix, visualized in Grafana via the Zabbix datasource.
- Prometheus: K8s and application-level metrics if/when needed (kube-state-metrics, application exporters). Not deployed at host level since Zabbix agent covers that.

### Why node_exporter was dropped

Originally planned to deploy `node_exporter` on all VMs for Prometheus to scrape host metrics. Dropped after the Grafana-Zabbix plugin was successfully connected, Zabbix agent already collects the same host metrics and Grafana can visualize them directly via the Zabbix datasource. node_exporter would be pure duplication.

### Role split

|Tool|Scope|
|---|---|
|**Zabbix agent**|Host-level metrics: all VMs|
|**Prometheus**|K8s/app metrics: when needed|
|**Grafana**|Unified visualization (both datasources)|

### Alternatives rejected

- node_exporter + Prometheus for host metrics: initially planned, dropped as redundant once Zabbix datasource was working in Grafana.
- Wazuh for metrics: Wazuh is a SIEM, not a metrics platform. No time-series data in Prometheus-compatible format.

---

## ADR-005: Zabbix retained in the stack for infrastructure monitoring

**Status:** Decided (revised, see history below)

### Context

Zabbix was originally planned as part of the monitoring stack. During Phase 3, the Grafana-Zabbix plugin (`alexanderzobnin-zabbix-app`) failed to connect despite correct credentials, reachable API, and valid network path.

### Decision history

1. **Initially:** Zabbix included in the stack for infrastructure monitoring alongside Prometheus.
2. **Phase 3 (mid-debug):** Zabbix temporarily removed from the active stack after exhausting standard troubleshooting (two plugin versions, two Grafana versions, API token vs user/password). Documented in `docs/troubleshooting/zabbix-apache-authorization-header.md`.
3. **Phase 3 (resolution):** Root cause identified and fixed. Zabbix reinstated.

### Root cause of the Phase 3 issue

Apache was stripping the `Authorization` header before it reached PHP-FPM, causing every authenticated Zabbix API call to return `"Not authorized"`. Fix: `sudo a2enconf php8.5-fpm && sudo systemctl reload apache2`.

See full post-mortem: `docs/troubleshooting/zabbix-apache-authorization-header.md`

### Final decision

Zabbix is retained for:

- Host metrics collection via Zabbix agent (see ADR-004)
- Infrastructure monitoring, host groups, triggers, alerting
- Complementing Wazuh for infrastructure-level visibility (process monitoring, service checks)
- Learning value, Zabbix is widely used in enterprise environments

### Role split

|Tool|Role|
|---|---|
|**Zabbix**|Host metrics + infrastructure monitoring + alerting|
|**Prometheus**|K8s/application metrics|
|**Wazuh**|SOC: FIM, brute force detection, CVE scan, SIEM|
|**Grafana**|Unified visualization (Zabbix + Prometheus datasources)|

---

## ADR-006: Ansible for agent deployment (Phase 5)

**Status:** Decided (revised)

### Context

Agents need to be deployed on all 10 monitored VMs. Manual installation is not repeatable.

### Decision

Phase 5 Ansible deploys:

- **Zabbix agents** on all K3s nodes + monitoring VMs → host metrics + infra monitoring
- **Wazuh agents** on all K3s nodes + monitoring VMs → SOC/security

`node_exporter` removed from scope, covered by Zabbix agent (see ADR-004).

### Why

- Repeatability: one playbook run restores the full agent stack on any VM
- Consistency: same config across all nodes, no manual drift
- Scope reduction: dropping node_exporter simplifies the playbook without losing any capability
- Portfolio value: Ansible automation across a 10-VM lab demonstrates IaC skills

### Alternatives rejected

- Manual install on each VM: done for central servers (educational value), not appropriate for repeated per-node tasks.
- K3s DaemonSet for node_exporter: valid K8s-native approach, deferred as a potential enhancement post-Phase 6.
  
## ADR-007: Prometheus self-monitoring dashboard scoped to process-level only

**Status:** Accepted

**Context:**
Building a custom Grafana dashboard for Prometheus self-monitoring, intended for community reuse.

**Decision:**
Scope strictly to process-level metrics (`process_*`, `go_*`, `prometheus_*`). Host-level metrics (RAM, disk, load) are already covered by the Zabbix agent on Prometheus-srv (see ADR-004).

**Why:**
Avoids a duplicate monitoring surface. The dashboard description explicitly states its scope to prevent user confusion ("why no disk usage panel?").

---

## ADR-008: Classic dashboard schema (v1) chosen over schema v2 for external export

**Status:** Accepted

**Context:**
Grafana 13.1's native dashboard editor defaults to schema v2 (`elements`/`layout`), which requires the experimental `dashboardNewLayouts` feature toggle to import, not enabled on standard instances or on Grafana.com.

**Decision:**
Export dashboards intended for external sharing via the classic REST API (`GET /api/dashboards/uid/<uid>`) rather than the UI's "Export as code," and manually re-add the classic-schema fields required by external validators.

**Why:**
Extra manual step per export, but guarantees compatibility with any Grafana instance ≥ 10.x and with Grafana.com's upload validator. Documented in `docs/troubleshooting/grafana-dashboard-schemaV2-export-failure.md`.

## ADR-009: Ubuntu 26.04 for the Wazuh VM (forced pre-.1 upgrade)

**Status:** Accepted

### Context

The Wazuh SOC was initially built on Ubuntu 24.04. During Phase 4, a persistent Vulnerability Detection failure triggered a long investigation across several reinstalls. To eliminate the OS as a variable, and because an earlier working Wazuh deployment had run on Ubuntu 26.04, the Wazuh VM was migrated from 24.04 to 26.04 ("Resolute Raccoon").

At the time (July 2026), the LTS→LTS upgrade path was not yet open through the standard `do-release-upgrade` channel: 26.04.1, the first point release that normally unlocks the upgrade, had not shipped. The upgrade was forced with `do-release-upgrade -d`.

The VD failure was ultimately unrelated to the OS, the root cause was the `disable_scan_manager` internal option (see `docs/troubleshooting/wazuh-soc-troubleshooting.md`, TS-1).

### Decision

Keep the Wazuh VM on Ubuntu 26.04, upgraded in place via `do-release-upgrade -d`.

### Why

- Rules out the OS variable: the migration removed any doubt that the OS was involved in the VD failure, letting the investigation converge on the real cause.
- Matches the earlier working reference: the previous successful Wazuh deployment ran on 26.04, so staying on it keeps parity with a known-good environment.
- Acceptable for a lab: the Wazuh installer flags 26.04 as outside its recommended systems list, but the deployment runs correctly and the risk is contained to a lab.

### Alternatives rejected

- Stay on 24.04: abandoned mid-investigation to eliminate the OS as a variable.
- Fresh 26.04 VM from ISO: cleaner, but slower than an in-place upgrade; the in-place path was chosen to reuse the existing network/identity config.
- Wait for 26.04.1: would have blocked progress for weeks; the forced `-d` upgrade was accepted instead.

### Consequences

- The VM runs an OS newer than Wazuh officially recommends, fine for a lab, would warrant caution in production.
- Forcing the upgrade before the .1 point release means the VM missed the first batch of post-release fixes. No issues observed in practice.
- The two kernel images left by the release upgrade account for a large share of the CVE findings; removing the old kernel clears them.

---

## ADR-010: Freeze Wazuh at version 4.14.6

**Status:** Accepted

### Context

Phase 4 was destabilised in part by version churn during troubleshooting, a partial downgrade to 4.14.5 left a manager/indexer version mismatch. A routine `apt upgrade` pulling a new Wazuh release could silently break a working SOC, a risk the official quickstart documentation explicitly warns about.

### Decision

Pin the deployment to Wazuh 4.14.6. Immediately after a successful install:

- disable the Wazuh apt repository, `sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/wazuh.list`
- disable update notifications in the dashboard (Server APIs → *Disable updates notifications*)

Upgrades become a deliberate, tested action rather than a side effect of routine maintenance.

### Why

- Protects a validated deployment: the entire SOC (VD, FIM, active response) goes offline if the manager fails to start after an upgrade. Freezing removes that class of accident.
- Version consistency: pins manager, indexer, and dashboard to the same release, avoiding the mismatch that caused problems during troubleshooting.
- Aligns with vendor guidance: the Wazuh quickstart explicitly recommends disabling the repository after install.

### Alternatives rejected

- Leave the repository enabled: convenient for updates, but exposes the SOC to accidental breakage on any `apt upgrade`.
- Auto-update: unacceptable for infrastructure where an upgrade can take the whole SOC down without warning.

### Consequences

- The SOC will not receive Wazuh security/feature updates automatically; any upgrade must be planned, tested, and the repository re-enabled explicitly.
- This freeze must be revisited before any future Wazuh 5.x migration.
  

## ADR-011: Ansible control node on Alpine Linux

**Date:** 2026-07-18 · **Status:** Accepted

**Context.** Phase 5 requires a dedicated control node. The initial Ubuntu VM
consumed 4 GiB RAM for a role with no heavy dependency; host RAM is the lab's
scarcest resource (31 GiB physical).

**Decision.** Rebuild the control node on Alpine Linux (1 GiB). Ansible is
installed from the Alpine repository (`apk add ansible`, full bundle: core +
community collections including community.zabbix). SSH key passphrase is
served by keychain in the shell profile.

**Consequences.** Minimal footprint, consistent with load-srv (already
Alpine). Trade-offs accepted: OpenRC quirks (services suffixed `-d`), Python
packages managed via apk/pipx, and the control node itself runs the 4.8.2
Wazuh agent (see ADR-010 amendment).

---

## ADR-012: Ansible scope: OPNsense and hypervisor excluded

**Date:** 2026-07-19 · **Status:** Accepted

**Context.** With 13 hosts under Ansible, two machines remained candidates:
the OPNsense firewall VM and the Arch hypervisor.

**Decision.** Both are intentionally excluded from the Ansible perimeter.

- OPNsense: FreeBSD-based, no standard Python/SSH management path; its
  configuration is managed through its own UI/config.xml and documented
  separately. A `connection: local` placeholder in the inventory produced
  misleading SUCCESS results and was removed.
- Hypervisor (My-pc): trust hierarchy. The control node holds SSH keys and
  passwordless sudo on the entire lab; granting it root on the machine that
  *runs* the lab would invert the containment model, a compromised control
  node must not yield the host. The hypervisor already runs both monitoring
  agents (installed manually in earlier phases) and is monitored like any
  other host.

**Consequences.** `ansible all` covers exactly the 13 lab hosts. Firewall and
hypervisor changes remain manual and documented.

---

## ADR-010: Amendment: Alpine agents on 4.8.2

**Date:** 2026-07-18 · **Status:** Accepted exception

ADR-010 freezes the Wazuh repository at 4.14.6 to keep manager and agents in
lockstep. The Wazuh apk repository for Alpine stops at **4.8.2**, no 4.14
build exists. The two Alpine hosts (load-srv, ansible-srv) therefore run
agent 4.8.2 against the 4.14 manager: protocol-compatible, flagged "outdated"
in the dashboard, and **excluded from Vulnerability Detection scans**. Accepted
as-is; revisit if Wazuh publishes newer Alpine builds.

---

## ADR-013: Replace OPNsense with FortiGate VM

**Status:** In progress

### Context

OPNsense has served as the inter-network router since Phase 1 (see ADR-002). FortiGate is more widely used in enterprise and government-adjacent environments (relevant to this project's target role), and Fortinet now offers a permanent evaluation license for KVM deployments, removing the earlier licensing objection.

### Decision

Replace the OPNsense VM with a FortiGate-VM, keeping the same IPs (WAN 10.10.0.254, LAN 10.20.0.254) and the same routing behavior, so no downstream host needs reconfiguration.

### Why

- Broader enterprise relevance than OPNsense for portfolio purposes.
- Permanent evaluation license removes the cost barrier for a lab.
- IP/route parity means the cutover is isolated to the router itself.

### Alternatives rejected

- **Keep OPNsense** — works fine, but FortiGate experience has more weight for the target role.
- **Time-limited FortiGate trial** — rejected once the permanent evaluation license was confirmed available for KVM/private cloud.

### Risk noted

FortiGate's NAT/policy model differs from OPNsense's outbound NAT toggle. The same asymmetric-routing failure mode hit with libvirt's `LIBVIRT_PRT` masquerade chain (ADR-002) is expected to resurface here and must be checked explicitly during cutover.

---

## ADR-013 Amendment: Migration blocked by evaluation license limits

**Status:** Blocked

### Context

ADR-013 planned to replace OPNsense with a FortiGate-VM using the permanent evaluation license, keeping IP/route parity so no downstream host would need reconfiguration. During cutover planning, the license's resource ceiling was checked against the lab's actual topology.

The FortiGate-VM permanent evaluation license caps interfaces, firewall policies, and routes at three each (Fortinet official documentation, "Permanent trial mode for FortiGate-VM," *Limitations of the Evaluation VM license*). The lab's topology already requires three interfaces (WAN, `k3s-net`, `monitoring-net`) on its own, before counting any future network such as the planned `mgmt-net` for Bastion-lab. Replicating OPNsense's current ruleset needs more than three firewall policies (inter-network allow rules in both directions, plus outbound NAT per network) and correspondingly more than three routes.

### Decision

Halt the cutover. OPNsense remains the active router (ADR-002, and the ADR-002 outbound NAT amendment). The FortiGate VM is retained, powered off, in case a licensed version becomes available or the lab topology is later simplified enough to fit within the evaluation limits.

### Why

- Diagnosed early: checking the license ceiling against the topology before cutover avoided a partial migration that would have failed mid-way (interfaces up, policies rejected once the limit was hit).
- Segmentation is non-negotiable: the two-network separation is a deliberate security boundary (ADR-001), not a detail to sacrifice for an unlicensed feature comparison.
- No budget justification: a licensed FortiGate is not justified for a personal lab when OPNsense already fulfills the same architectural role (ADR-002).

### Alternatives rejected

- Reduce to fewer networks to fit the license: rejected, would undo the security boundary that ADR-001 was written to establish.
- Purchase a licensed FortiGate: out of scope for a personal lab budget.
- Continue with a partial ruleset (fewer policies than OPNsense currently enforces): rejected, would leave gaps in the inter-network rules (Zabbix agent, Wazuh agent, Prometheus scrape) that the cutover plan explicitly required to validate before deleting OPNsense.

### Consequences

- FortiGate's NAT/policy model differs from OPNsense's outbound NAT toggle. The same asymmetric-routing failure mode hit with libvirt's `LIBVIRT_PRT` masquerade chain (ADR-002) is expected to resurface here and must be checked explicitly during cutover — never reached, but flagged for any future attempt.
- The FortiGate enterprise-relevance goal stated in the original ADR-013 context is not achieved in this lab. OPNsense remains the router of record, unchanged.
- The attempt is retained as documentation of a correctly diagnosed licensing constraint, not a technical or networking failure.
- Revisit if a licensed FortiGate becomes available, or if the lab topology is deliberately reduced to three networks or fewer.

---

## ADR-014: Canonical static routes to mgmt-net across the fleet

**Date:** 2026-08-29 · **Status:** Accepted

### Context

SSH and ICMP from the bastion (`10.30.0.10`, mgmt-net) to most of the fleet (k3s-net, monitoring-net) failed silently, timeouts with no explicit rejection. OPNsense firewall rules were correct and permissive (`pass` from `10.30.0.10` to `*`), confirmed via `Firewall → Log Files → Live View`: the outbound packet was seen and allowed on the destination interface. No state was ever created in `Firewall → Diagnostics → States` for traffic initiated from the bastion, indicating the reply never made it back.

An Ansible ad-hoc audit (`ip route` across all 13 hosts) showed the actual cause: most VMs receive their routing table via DHCP (`proto dhcp`), which never included a route back to `10.30.0.0/24`. Only `k3s-srv-1`, `load-srv`, and `ansible-srv` had a default route that happened to cover mgmt-net, explaining why they behaved differently from the rest of the fleet during initial triage. VMs without any route to `10.30.0.0/24` accepted the inbound packet but had no way to route the reply, dropping it locally, invisible to the firewall.

### Decision

Deploy a canonical `/etc/netplan/99-routes.yaml` via Ansible on all 13 hosts, explicitly listing static routes to every other lab network (k3s-net, monitoring-net, mgmt-net, k8s-ha-net) rather than relying on DHCP-provided routes for inter-network reachability.

`k3s-srv-1` (the only host on static IP via `50-cloud-init.yaml`, `dhcp4: false`) required manual `netplan apply` via `virsh console` instead of through the Ansible SSH connection, which the interface renegotiation was cutting mid-play.

### Why

- Firewall logs proved the aller path was never the problem, ruling out OPNsense saved significant time once actually consulted; should be the first diagnostic step for any "traffic passes the firewall but nothing comes back" symptom going forward.
- A single canonical routes file per network side, applied via the existing Ansible pipeline (already used for the k3s-net ↔ monitoring-net route, see original `k3s-cluster-build.md`), keeps routing config in one auditable place instead of depending on whatever OPNsense's DHCP server happens to hand out.
- `netplan apply` reconciles the full file state, so redeploying this playbook also removes any stale/manual route previously added to the same file, incidental cleanup alongside the fix.

### Alternatives rejected

- **Fix via OPNsense DHCP options** (classless static routes): would have centralized the fix server-side, but static IP hosts (`k3s-srv-1`) don't consume DHCP options at all, so full fleet coverage still requires a per-host static route regardless.
- **Per-host manual `ip route add`**: works but not persistent across reboot, and not idempotent/auditable the way a versioned netplan file is.

### Consequences

- Routing is now split across two sources of truth per host: DHCP-provided routes (host's own network + WAN-adjacent) and the static `99-routes.yaml` (everything cross-network). Acceptable, but worth remembering if a future network is added, it needs an entry in this file, not just an OPNsense DHCP scope.
- This likely explains, retroactively, part of the previously undiagnosed `k3s-db` Ansible unreachability documented earlier in the project history. Not confirmed as the sole cause, flagged here in case a similar symptom resurfaces on a host not yet covered.
- Full diagnostic trail: `docs/troubleshooting/troubleshooting-bastion-routing-2026-08-29.md`.

---

## ADR-015: netplan-only route fix left Alpine hosts uncovered

**Date:** 2026-08-29 · **Status:** Accepted

### Context

ADR-014 deployed a canonical static-routes file to fix missing routes back to `mgmt-net` (10.30.0.0/24) across the fleet, but the fix was written and validated only against **netplan**, which does not exist on Alpine Linux. Alpine hosts (`Ansible-srv`, and by extension `load-srv`, `Bastion-srv`) manage networking through `/etc/network/interfaces` instead, and were silently left out of ADR-014's coverage.

The gap surfaced during a later session, on a fresh full-fleet reboot: `ssh ansible-srv` hung with the same symptom ADR-014 had just fixed everywhere else. `ip route` on `Ansible-srv` confirmed no route to `10.30.0.0/24`, and a ping toward the bastion returned `Destination Port Unreachable` **from `10.20.0.1`** (the libvirt NAT gateway), not from OPNsense (`10.20.0.254`) — proof the packet was falling through to the default route instead of being routed via OPNsense, exactly the ADR-014 failure mode, on a host ADR-014 never touched.

A manual fix attempt made things briefly worse: editing `/etc/network/interfaces` by hand dropped the leading `i` from `iface eth0 inet static` (`face eth0 inet static`), which Alpine's `ifup` parses as an orphaned block, orphaning the `address`/`gateway` lines under it (`address '10.20.0.125' without interface`). Restarting the networking service in that state briefly took the host's networking down entirely. Recovered via `virsh console` (never dependent on the network) and a from-scratch manual `ip addr` / `ip route` sequence, then a corrected file.

### Decision

Extend route coverage to Alpine hosts via `/etc/network/interfaces`, using `up ip route add` directives mirroring the netplan file's routes, with the **correct default gateway** (`10.20.0.1`, the network's actual default gateway — not `10.20.0.254`, which is OPNsense's role as inter-network router only, a distinction blurred during the manual recovery and worth stating explicitly here):

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.20.0.125
    netmask 255.255.255.0
    gateway 10.20.0.1
    up ip route add 10.10.0.0/24 via 10.20.0.254 dev eth0
    up ip route add 10.30.0.0/24 via 10.20.0.254 dev eth0
    up ip route add 10.40.0.0/24 via 10.20.0.254 dev eth0
```

Validated with a full cold reboot of `Ansible-srv` (not just an in-memory `ip route add`, which ADR-014's fix had already relied on for its own validation) — SSH via the full bastion ProxyJump succeeded immediately post-boot, confirming the file is correctly parsed and applied on startup, not just patched live.

### Why

- ADR-014's own troubleshooting section warned that "DHCP and static configuration coexisting on the same lab creates heterogeneous routing topologies hard to audit visually" — this gap is a direct instance of that warning going unheeded across a different axis (OS/network-manager, not DHCP/static).
- A live `ip route add` is not sufficient validation for a persistence fix; only a real reboot proves the on-disk config is both syntactically valid and functionally correct. This session's own incident (a typo silently accepted by `vi`, only surfaced by `rc-service networking restart`) reinforces this.
- Fixing forward (extending route coverage) rather than migrating Alpine hosts to a different network manager avoids introducing a second migration mid-incident.

### Alternatives rejected

- **Migrate Alpine hosts off `/etc/network/interfaces` to a netplan-compatible manager**: unnecessary scope increase for a routing gap; Alpine's native networking is otherwise working as designed (ADR-011 chose Alpine specifically for its minimal footprint).
- **Add the missing routes only in memory, revisit persistence later**: rejected after this session's incident showed that "works right now" and "survives a reboot" are not the same claim, and the two can diverge exactly when least convenient (mid-diagnosis, multiple VMs already down for testing).

### Consequences

- The fleet now has **three** route-management mechanisms for the same logical requirement (netplan for Ubuntu, `/etc/network/interfaces` for Alpine, and OPNsense's own routing table) — acceptable given the lab's mixed-OS design (ADR-011), but any *future* route change must be applied in both places, not just netplan.
- `load-srv` (Alpine, k3s-net) and `Bastion-srv` were not re-verified in this session and should be checked for the same gap before being trusted as fully covered by ADR-014/015.
- Editing `/etc/network/interfaces` on a live Alpine host now has a documented failure mode (silent single-character corruption breaking interface parsing) — future edits should `cat` the file back immediately after saving, before restarting any network service, and prefer testing via a hypervisor console session over SSH when touching the file a host's own SSH session depends on.
## ADR-016: Missing pass rule on k3snet left the interface entirely inbound-blocked

**Date:** 2026-08-29 · **Status:** Accepted

### Context

Following the ADR-014/015 route fixes, a routine `health-check.yml` playbook run surfaced a new symptom unrelated to routing: all 8 hosts on `k3s-net` (k3s-srv-1/2/3, k3s-agent-1/2/3, k3s-db, load-srv) reported their Wazuh agent status as `pending`, never `connected`, despite the agent process being active on every host.

Initial diagnosis suspected a repeat of the same routing gap (ADR-014/015), but the evidence pointed elsewhere:

- `ping` from any k3s-net host to `10.10.0.254` (OPNsense's own IP on the k3snet interface) failed 100%, but **`ping` from OPNsense to any k3s-net host succeeded**, an asymmetric failure the previous ADRs' routing fixes cannot explain (routes were confirmed correct on both sides).
- A `tcpdump -i vtnet1` capture on OPNsense's own k3snet interface **showed the ICMP echo requests arriving** from the k3s-net hosts, ruling out the bridge, the hypervisor's `iptables`/`ufw` FORWARD chain (a `DEFAULT_FORWARD_POLICY="DROP"` misconfiguration was found and fixed along the way, unrelated to this issue, see Consequences), and the VM's own routing table.
- `pfctl -sr | grep icmp` on OPNsense showed only the default IPv6 RFC4890 rules, no IPv4 ICMP rule at all.
- `pfctl -ss` showed no state was ever created for the ICMP traffic, meaning `pf` dropped the packets before even considering them for a state, the traffic never matched any `pass` rule.
- The `Firewall → Rules → k3snet` page itself stated plainly: *"No k3snet rules are currently defined. All incoming connections on this interface will be blocked until you add a pass rule."*

The `k3snet` interface had never had an explicit pass rule since its creation in Phase 1 (ADR-001/002), unlike `LAN` (monitoring-net), which has carried an explicit `"Allow Bastion-srv only into monitoring-net"` rule since Bastion-lab was added. This had gone unnoticed because all of the lab's normal traffic patterns are initiated *from* monitoring-net *into* k3s-net (Prometheus scraping kubelet/cAdvisor, SSH via the bastion, Ansible), or *by* OPNsense itself (which is not subject to its own interface's inbound rules for self-initiated traffic). No prior workflow required a k3s-net host to initiate outbound traffic past its own subnet until this session's ping diagnostics and the Wazuh agent's manager-directed traffic (port 1514/1515) surfaced it.

### Decision

Add an explicit pass rule on the `k3snet` interface:

```
Action: Pass
Interface: k3snet
Direction: in
TCP/IP Version: IPv4
Protocol: any
Source: k3snet net
Destination: any
Description: Allow k3s-net outbound to all lab networks
```

Mirrors the existing pattern already in place on `LAN` (monitoring-net) and `BASTION`, both of which have explicit outbound-allow rules for their respective subnets.

### Why

- Symmetry with existing interfaces: every other interface in the topology (LAN, BASTION) already had this exact kind of rule; k3snet was the outlier, not a special case that needed different treatment.
- `pf`'s per-interface default-deny behavior means an interface with zero rules blocks all *inbound* traffic to that interface regardless of what the routing table or the bridge allows, confirmed directly by OPNsense's own UI warning banner, which should have been checked before spending time on `tcpdump`/`pfctl` state analysis.
- Fixing the rule rather than reactively allowing single ports (e.g., just 1514/1515 for Wazuh) restores k3snet to the same general-purpose reachability every other lab network has, since more asymmetric-failure symptoms of the same root cause are likely to surface later (e.g., NTP, DNS, or future agent traffic initiated from k3s-net).

### Alternatives rejected

- **Narrow rule scoped only to Wazuh's ports (1514/1515 TCP/UDP)**: would have fixed the immediate symptom but left the interface in the same fragile, inconsistent state relative to every other interface, and virtually guaranteed a repeat of this exact diagnostic session the next time a different k3s-net-initiated flow was needed.
- **Leave k3snet rule-less and route Wazuh traffic through the bastion instead**: rejected as unnecessary complexity; the interface's default-deny-on-inbound is not a security boundary anyone deliberately designed for k3s-net (compare to the genuinely deliberate isolation in ADR-001), it was an oversight, not a policy.

### Consequences

- **A red herring correction made along the way, kept for the record**: `iptables -L FORWARD -n -v` on the hypervisor (My-ship) showed `policy DROP`, traced to `/etc/default/ufw`'s `DEFAULT_FORWARD_POLICY="DROP"`, likely altered by the `ufw disable`/`ufw enable` cycle performed accidentally at the very start of this session's original troubleshooting (see ADR-014's initial symptom investigation). Corrected to `ACCEPT` (`sed` + `ufw reload`) since a routing hypervisor should default to forwarding. **This fix was necessary and correct to keep, but was not the actual cause of the k3snet symptom** — worth remembering that two real misconfigurations coexisting during the same diagnostic session is possible and each needs independent confirmation, not just "the ping works now" after any one fix.
- All 8 previously-`pending` Wazuh agents confirmed `connected` immediately after the rule was applied, closing the loop from symptom (health-check playbook) to root cause (missing firewall rule) to fix, with functional validation, not just a ping test.
- `docs/phase-1-network/README.md`'s OPNsense configuration section should be updated to explicitly call out the pass rule per interface as a required step, not just implied by "add rules as needed" — the absence of this explicit callout is likely why k3snet was missed when the interface was first created.
