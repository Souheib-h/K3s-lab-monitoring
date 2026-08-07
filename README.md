# K3s-lab-monitoring

> A production-like NOC/SOC monitoring lab built on top of a self-hosted K3s HA cluster: two isolated networks, a dedicated firewall, and a full observability + security stack deployed from scratch, then automated with Ansible.

**Prerequisite:** [K3s-lab](https://github.com/Souheib-h/K3s-lab), the 6-node K3s HA cluster this lab monitors.

---

## What this is

This lab extends the K3s cluster with a fully isolated monitoring infrastructure:

- NOC: host metrics via Zabbix, K8s metrics via Prometheus (kube-state-metrics + kubelet/cAdvisor with explicit RBAC), unified in Grafana
- SOC: Wazuh SIEM validated end-to-end: FIM (realtime), SSH brute-force active response, CVE detection, VirusTotal integration, email alerting
- Network isolation: monitoring VMs live on a separate network, routed through OPNsense
- Automation: Zabbix + Wazuh agent rollout and host registration across all 13 VMs via Ansible

Everything is documented phase by phase, including architecture decisions (ADR) and troubleshooting post-mortems.

---

## Architecture

```
  k3s-net (10.10.0.0/24)                    monitoring-net (10.20.0.0/24)
  K3s-srv-1      10.10.0.11                 Zabbix-srv     10.20.0.10
  K3s-srv-2      10.10.0.12                 Wazuh-srv      10.20.0.11
  K3s-srv-3      10.10.0.13                 Prometheus-srv 10.20.0.12
  K3s-agent-1    10.10.0.31                 Grafana-srv    10.20.0.13
  K3s-agent-2    10.10.0.32                 Ansible-srv    10.20.0.125
  K3s-agent-3    10.10.0.33
  K3s-db         10.10.0.20
  Load-srvs      10.10.0.10
  OPNsense WAN   10.10.0.254 ────────────── OPNsense LAN  10.20.0.254
```

The two networks are fully isolated at L2. OPNsense handles all inter-network routing and provides firewall logs, packet capture, and traffic visibility.

---

## Stack

| Tool                 | Role                                                                        |
| --------------------- | --------------------------------------------------------------------------- |
| **Zabbix 7.4**       | Host metrics (CPU, RAM, disk, network) via agent + infrastructure alerting  |
| **Prometheus 3.12**  | K8s metrics: kube-state-metrics, kubelet, cAdvisor                         |
| **Grafana 13.1 OSS** | Unified NOC dashboards: Zabbix + Prometheus datasources                    |
| **Loki**             | Log aggregation, monolithic deployment, local FS storage                   |
| **Grafana Alloy**    | Log/metric shipping agent, deployed via Ansible across 12 nodes            |
| **Wazuh 4.14**       | SOC: SIEM, FIM, brute force active response, CVE scan, VirusTotal          |
| **OPNsense 26.1**    | Router/firewall between k3s-net and monitoring-net                          |
| **Ansible**          | Agent deployment + Zabbix host registration ([configs/ansible](configs/ansible)) |

---

## Project status

| Phase | Description | Status |
| ----- | ----------- | ------ |
| 0 | GitHub setup, repo structure, aliases | ✅ Done |
| 1 | [Network, VMs, OPNsense, persistent routes](docs/phase-1-network/README.md) | ✅ Done |
| 2 | Install [Zabbix](docs/phase-2-install/zabbix.md) · [Wazuh](docs/phase-2-install/wazuh.md) · [Prometheus](docs/phase-2-install/prometheus.md) · [Grafana](docs/phase-2-install/grafana.md) | ✅ Done |
| 3 | NOC: [Grafana datasources](docs/phase-3-noc/noc-config.md) · [Prometheus self-monitoring dashboard](docs/phase-3-noc/prometheus-self-monitoring-dashboard.md) | ✅ Done |
| 4 | [SOC: Wazuh FIM, active response, CVE, VirusTotal, email alerting](docs/phase-4-soc/README.md) | ✅ Done |
| 5 | Automation: [Ansible agent rollout](docs/phase-5-ansible/ansible-automation.md) · [Prometheus ↔ K3s scraping](docs/phase-5-ansible/prometheus-k3s-scraping.md) | ✅ Done |
| 6 | Dashboard suite: Golden Signals, USE/RED, cross-source panels | 🔄 In progress |
| 7 | [Router migration: OPNsense → FortiGate](docs/phase-7-fortigate-migration/README.md) | 🚫 Blocked (license limits, see [ADR-013 amendment](DECISIONS.md)) |
| 8 | [Logging: Loki + Grafana Alloy](docs/phase-8-logging/README.md) | ✅ Done |

---

## Dashboards

| Dashboard | Scope |
| --------- | ----- |
| [Prometheus Self-Monitoring](dashboards/prometheus-self-monitoring.json): published on [Grafana.com (ID 25537)](https://grafana.com/grafana/dashboards/25537) | Process-level metrics (memory, CPU, TSDB cardinality, query engine health) for the Prometheus server itself: zero external dependencies |

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

## Reference

| Doc | Description |
| --- | ----------- |
| [Architecture Decisions](DECISIONS.md) | Why this stack, why these choices: full ADR log |
| [Troubleshooting: Ansible rollout war stories](docs/troubleshooting/ansible-rollout-war-stories.md) | 13 real-world issues hit during the agent rollout: symptom → root cause → fix |
| [Troubleshooting: Wazuh SOC](docs/troubleshooting/wazuh-soc-troubleshooting.md) | Post-mortems: silent vulnerability scanner, inert XML config blocks, mail relay |
| [Troubleshooting: Zabbix Apache auth](docs/troubleshooting/zabbix-apache-authorization-header.md) | Post-mortem: Apache stripping the Authorization header before PHP-FPM |
| [Troubleshooting: Grafana schema v2 export](docs/troubleshooting/grafana-dashboard-schemaV2-export-failure.md) | Post-mortem: why the native "Export as code" flow produces an incompatible schema |

---

## Environment

- Host: ThinkPad E14 Gen 5, i7-13700H, 31GB RAM, 476GB NVMe
- Hypervisor: KVM/libvirt on Arch Linux
- VM OS: Ubuntu 26.04 LTS Server · Alpine Linux (Ansible control node)
