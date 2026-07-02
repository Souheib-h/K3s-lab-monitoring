# Architecture Decision Records

This document captures the key architectural decisions made throughout the `K3s-lab-monitoring` project — not _what_ was built, but _why_ it was built that way. Each ADR answers: what was the context, what was decided, what alternatives were considered, and what are the consequences.

---

## ADR-001 — Two isolated networks instead of one flat network

**Status:** Decided

### Context

The lab runs a K3s cluster (3 control planes, 3 workers, 1 LB, 1 DB) alongside a dedicated monitoring stack (Wazuh, Prometheus, Grafana, Zabbix). Both sets of VMs could have shared a single libvirt network.

### Decision

Two separate networks:

- `k3s-net` (10.10.0.0/24) — cluster traffic
- `monitoring-net` (10.20.0.0/24) — monitoring stack

Routed through OPNsense.

### Why

- **Security boundary** — monitoring VMs can observe the cluster without being part of it. If a monitoring tool is compromised, it doesn't have L2 access to the cluster nodes.
- **Realistic architecture** — production monitoring infrastructure is always network-segregated from what it monitors.
- **Traffic isolation** — Prometheus scrape traffic, Wazuh agent communication, and Zabbix polling don't pollute cluster inter-node traffic.

### Alternatives rejected

- **Single flat network** — simpler, but no security boundary and not representative of real environments.
- **VLANs on a single bridge** — overkill for a KVM lab, harder to manage without physical switching.

---

## ADR-002 — OPNsense as the inter-network router

**Status:** Decided

### Context

With two isolated networks, something needs to route between them. Options ranged from a simple kernel IP forwarding setup to a full firewall VM.

### Decision

OPNsense 26.1 deployed as a dedicated VM with:

- WAN interface on `k3s-net` (10.10.0.254)
- LAN interface on `monitoring-net` (10.20.0.254)

### Why

- **Visibility** — OPNsense provides packet capture, firewall logs, and traffic graphs. Proved invaluable during debugging (confirmed inter-network routing with live packet captures showing TTL decrements and MAC translation at the L3 boundary).
- **Realistic SOC context** — Wazuh and Grafana should ideally receive OPNsense syslog as a data source. Having a real firewall in the path makes this possible.
- **Learning value** — OPNsense config (NAT outbound disable, block private networks off on WAN) is directly transferable to real deployments.

### Alternatives rejected

- **iptables forwarding on the host** — works, but no visibility, no logs, not a realistic setup.
- **libvirt routed mode** — attempted, failed due to masquerade chain (`LIBVIRT_PRT`) intercepting traffic before custom routing rules. Required adding `RETURN` rules to exempt inter-network traffic — ultimately replaced by OPNsense for cleaner separation.

### Notable fix

libvirt's nftables/iptables masquerade chain (`LIBVIRT_PRT`) rewrites source IPs before custom routing can work. Fix: switch libvirt firewall backend to `iptables`, add `RETURN` rules to exempt inter-network traffic, disable outbound NAT on OPNsense.

---

## ADR-003 — Wazuh all-in-one for SOC

**Status:** Decided

### Context

Wazuh has three components: indexer (OpenSearch-based), manager, and dashboard. They can be deployed separately (distributed) or co-located (all-in-one).

### Decision

All-in-one deployment on a single `Wazuh-srv` VM (8GB RAM, 4 vCPU).

### Why

- **Scale** — the lab monitors fewer than 20 endpoints, well within the all-in-one supported range (up to 100 agents).
- **Simplicity** — distributed deployment adds cert complexity, inter-component networking, and failure domains that aren't justified at this scale.
- **Resource efficiency** — a single 8GB VM handles all three components comfortably.

### Alternatives rejected

- **Distributed deployment** — appropriate for production at scale, overkill here.
- **Wazuh Cloud** — defeats the purpose of a self-hosted SOC lab.

---

## ADR-004 — Zabbix agent for host metrics, Prometheus for K8s/application metrics

**Status:** Decided (revised)

### Context

The NOC stack needs metrics at two levels: host-level (CPU, RAM, disk, network) and K8s/application-level (pod status, container metrics, application counters).

### Decision

- **Zabbix agent** — host-level metrics on all monitored VMs. Already collected and stored in Zabbix, visualized in Grafana via the Zabbix datasource.
- **Prometheus** — K8s and application-level metrics if/when needed (kube-state-metrics, application exporters). Not deployed at host level since Zabbix agent covers that.

### Why node_exporter was dropped

Originally planned to deploy `node_exporter` on all VMs for Prometheus to scrape host metrics. Dropped after the Grafana-Zabbix plugin was successfully connected — Zabbix agent already collects the same host metrics and Grafana can visualize them directly via the Zabbix datasource. node_exporter would be pure duplication.

### Role split

|Tool|Scope|
|---|---|
|**Zabbix agent**|Host-level metrics — all VMs|
|**Prometheus**|K8s/app metrics — when needed|
|**Grafana**|Unified visualization (both datasources)|

### Alternatives rejected

- **node_exporter + Prometheus for host metrics** — initially planned, dropped as redundant once Zabbix datasource was working in Grafana.
- **Wazuh for metrics** — Wazuh is a SIEM, not a metrics platform. No time-series data in Prometheus-compatible format.

---

## ADR-005 — Zabbix retained in the stack for infrastructure monitoring

**Status:** Decided (revised — see history below)

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
- Infrastructure monitoring — host groups, triggers, alerting
- Complementing Wazuh for infrastructure-level visibility (process monitoring, service checks)
- Learning value — Zabbix is widely used in enterprise environments

### Role split

|Tool|Role|
|---|---|
|**Zabbix**|Host metrics + infrastructure monitoring + alerting|
|**Prometheus**|K8s/application metrics|
|**Wazuh**|SOC — FIM, brute force detection, CVE scan, SIEM|
|**Grafana**|Unified visualization (Zabbix + Prometheus datasources)|

---

## ADR-006 — Ansible for agent deployment (Phase 5)

**Status:** Decided (revised)

### Context

Agents need to be deployed on all 10 monitored VMs. Manual installation is not repeatable.

### Decision

Phase 5 Ansible deploys:

- **Zabbix agents** on all K3s nodes + monitoring VMs → host metrics + infra monitoring
- **Wazuh agents** on all K3s nodes + monitoring VMs → SOC/security

`node_exporter` removed from scope — covered by Zabbix agent (see ADR-004).

### Why

- **Repeatability** — one playbook run restores the full agent stack on any VM
- **Consistency** — same config across all nodes, no manual drift
- **Scope reduction** — dropping node_exporter simplifies the playbook without losing any capability
- **Portfolio value** — Ansible automation across a 10-VM lab demonstrates IaC skills

### Alternatives rejected

- **Manual install on each VM** — done for central servers (educational value), not appropriate for repeated per-node tasks.
- **K3s DaemonSet for node_exporter** — valid K8s-native approach, deferred as a potential enhancement post-Phase 6.