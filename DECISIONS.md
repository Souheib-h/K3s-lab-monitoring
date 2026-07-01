# Architecture Decision Records

This file documents significant architectural decisions made during the project, including the context, reasoning, and consequences of each choice.

---

## ADR-001 — Zabbix removed from the monitoring stack

**Date:** 2026-07-01 **Status:** Decided

### Context

Zabbix was included in the initial Phase 2 plan as a monitoring server alongside Prometheus, Grafana, and Wazuh. After completing the installation and attempting Phase 3 integration (Grafana ↔ Zabbix datasource), two issues surfaced:

**Technical blocker — Grafana plugin incompatibility with Zabbix 7.4:** The official `alexanderzobnin-zabbix-app` plugin is broken with recent Grafana versions (confirmed broken since Grafana 11.5+, reported in [GitHub issue #1957](https://github.com/grafana/grafana-zabbix/issues/1957)). Exhaustive troubleshooting was performed:

|Hypothesis tested|Result|
|---|---|
|Network/firewall between Grafana-srv and Zabbix-srv|✅ Healthy — 13ms API response via curl|
|Wrong credentials|✅ Eliminated — direct web UI login successful|
|Plugin file permissions (root vs grafana ownership)|✅ Fixed — did not resolve the issue|
|Unsigned plugin blocked by Grafana|✅ Eliminated — plugin is signed|
|Plugin version 6.4.0 incompatibility|Downgraded to 6.3.0 — partial improvement (no more 60s timeout, new error surfaced)|
|Plugin version 6.2.0 incompatibility|Same error as 6.4.0|
|Grafana version 13.1.0 incompatibility|Downgraded to 12.4.5 — same error|
|HTTP Basic auth conflict|Confirmed: plugin 6.3.0 explicitly rejects Basic auth for Zabbix 7.2+|
|API token authentication|Rejected by Zabbix with "Not authorized" — confirmed via curl independently of Grafana|
|Zabbix 7.4 breaking change on `auth` parameter|Confirmed: `auth` in JSON-RPC body removed in Zabbix 7.4, `Authorization: Bearer` header also fails|
|Direct DB Connection (MySQL) as workaround|MySQL datasource connected successfully, but plugin still requires API auth before allowing Direct DB Connection|

A working MySQL datasource was established (`Database Connection OK`) and raw SQL queries against the Zabbix DB returned live data — proving the network and DB layer are fully functional. The blocker is strictly the plugin's API authentication layer being incompatible with Zabbix 7.4's authentication model.

**Architectural redundancy:** Beyond the technical blocker, a review of Zabbix's role in this specific stack revealed significant overlap:

|Capability|Zabbix|Covered by|
|---|---|---|
|File Integrity Monitoring|✅|Wazuh (more advanced)|
|Brute force detection|✅|Wazuh (native OSSEC rules)|
|CVE / Vulnerability scanning|✅|Wazuh (integrated CVE database)|
|Log monitoring / SIEM|Limited|Wazuh (full SIEM)|
|MITRE ATT&CK mapping|❌|Wazuh|
|System metrics (CPU, RAM, disk, network)|✅|Prometheus + node_exporter|
|Compliance reporting (PCI-DSS, GDPR)|❌|Wazuh|

Every capability Zabbix would provide in this lab is already covered — and covered better — by either Wazuh (security/SOC) or Prometheus (metrics/NOC).

### Decision

**Zabbix is removed from the active monitoring stack for this project.**

- `Zabbix-srv` VM remains provisioned but is no longer part of the architecture
- Zabbix will be studied in a separate, dedicated lab project at a later date
- The NOC stack is: Prometheus + node_exporter + Grafana
- The SOC stack is: Wazuh

### Consequences

- Phase 3 NOC dashboards will be built on Prometheus + node_exporter (node_exporter deployed via Ansible in Phase 5)
- Phase 4 SOC configuration focuses entirely on Wazuh (FIM, brute force detection, CVE scan)
- The Grafana ↔ Zabbix plugin troubleshooting, while ultimately unsuccessful for this project, is fully documented here and in `docs/06-noc-config.md` as a reference for future Zabbix integration attempts

---

## ADR-002 — Prometheus + node_exporter chosen for NOC metrics

**Date:** 2026-07-01 **Status:** Decided

### Context

Following ADR-001, a replacement for Zabbix's metrics collection role was needed for the NOC dashboards.

### Decision

**Prometheus scrapes `node_exporter` instances running on each monitored VM.**

- `node_exporter` exposes 1000+ host-level metrics (CPU, memory, disk, network, filesystem, systemd units...) via HTTP on port 9100
- Prometheus scrapes all targets every 15s (default, configurable)
- Grafana reads Prometheus as a datasource (already connected and validated in Phase 3)
- Dashboard import: **Node Exporter Full** (Grafana dashboard ID `1860`) — 30M+ downloads, community standard

### Why not Zabbix agent

- Requires the Grafana-Zabbix plugin which is broken (see ADR-001)
- Adds unnecessary complexity via Zabbix server as an intermediary

### Why not Wazuh for metrics

- Wazuh is a SIEM, not a metrics platform — it does not expose time-series data in a Prometheus-compatible format
- Grafana has no native Wazuh datasource plugin

### Consequences

- node_exporter deployment on all monitored VMs (K3s nodes + monitoring VMs) is handled by Ansible in Phase 5
- `prometheus.yml` scrape config will be extended with all node_exporter targets once agents are deployed
- Grafana NOC dashboards will use the Prometheus datasource exclusively