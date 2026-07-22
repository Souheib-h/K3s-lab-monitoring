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
