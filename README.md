# K3s-lab-monitoring

> A production-like NOC/SOC monitoring lab built on top of a self-hosted K3s HA cluster — two isolated networks, a dedicated firewall, and a full observability + security stack deployed from scratch.

**Prerequisite:** [K3s-lab](https://github.com/Souheib-h/K3s-lab) the 6-node K3s HA cluster this lab monitors.

---

## What this is

This lab extends the K3s cluster with a fully isolated monitoring infrastructure:

- **NOC** — host metrics and K8s visibility via Zabbix + Prometheus, visualized in Grafana
- **SOC** — security monitoring via Wazuh (SIEM, FIM, brute force detection, CVE scan)
- **Network isolation** — monitoring VMs live on a separate network, routed through OPNsense
- **Automation** — agent deployment across all nodes via Ansible (Phase 5)

Everything is documented phase by phase, including architecture decisions and troubleshooting post-mortems.

---

## Architecture

```
  k3s-net (10.10.0.0/24)                    monitoring-net (10.20.0.0/24)
                                                                          
  K3s-srv-1      10.10.0.11                 Zabbix-srv     10.20.0.10   
  K3s-srv-2      10.10.0.12                 Wazuh-srv      10.20.0.11   
  K3s-srv-3      10.10.0.13                 Prometheus-srv 10.20.0.12   
  K3s-agent-1    10.10.0.31                 Grafana-srv    10.20.0.13   
  K3s-agent-2    10.10.0.32                                              
  K3s-agent-3    10.10.0.33                                              
  OPNsense WAN   10.10.0.254 ────────────── OPNsense LAN  10.20.0.254  
```

The two networks are fully isolated at L2. OPNsense handles all inter-network routing and provides firewall logs, packet capture, and traffic visibility.

---

## Stack

|Tool|Role|
|---|---|
|**Zabbix 7.4**|Host metrics (CPU, RAM, disk, network) via agent + infrastructure alerting|
|**Prometheus 3.12**|K8s/application metrics — pods, deployments, namespaces|
|**Grafana 13.1 OSS**|Unified NOC dashboards — Zabbix + Prometheus datasources|
|**Wazuh 4.14**|SOC — SIEM, FIM, brute force detection, CVE scan|
|**OPNsense 26.1**|Router/firewall between k3s-net and monitoring-net|
|**Ansible**|Agent deployment automation (Phase 5)|

---


---

## Quick start

```bash
# Start the K3s cluster
k3s-start

# Start the monitoring stack (Wazuh, Prometheus, Grafana, Zabbix)
monitoring-start

# Stop
monitoring-stop
k3s-stop
```

---

## Project status

| Phase | Description | Status |
|---|---|---|
| 0 | GitHub setup, repo structure, aliases | ✅ Done |
| 1 | [Network, VMs, OPNsense, persistent routes](docs/01-network.md) | ✅ Done |
| 2 | Install [Zabbix](docs/02-zabbix.md) · [Wazuh](docs/03-wazuh.md) · [Prometheus](docs/04-prometheus.md) · [Grafana](docs/05-grafana.md) | ✅ Done |
| 3 | [NOC config — Grafana datasources + dashboards](docs/06-noc-config.md) | 🔄 In progress |
| 4 | SOC config — Wazuh FIM, brute force, CVE | ⏳ Pending |
| 5 | Ansible — agent deployment across all nodes | ⏳ Pending |
| 6 | Documentation finalization | ⏳ Pending |

---


### Reference

|Doc|Description|
|---|---|
|Architecture Decisions|Why this stack, why these choices — full ADR|
|Troubleshooting: Zabbix Apache auth|Post-mortem — Apache stripping Authorization header to PHP-FPM|

---

## Environment

- **Host**: ThinkPad E14 Gen 5 — i7-13700H, 31GB RAM, 476GB NVMe
- **Hypervisor**: KVM/libvirt on Arch Linux
- **VM OS**: Ubuntu 26.04 LTS Server