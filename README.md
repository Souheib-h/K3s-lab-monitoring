# K3s-lab-monitoring

NOC/SOC lab — Monitoring and security for a K3s HA cluster, built from scratch.

> Prerequisite: [K3s-lab](https://github.com/Souheib-h/K3s-lab) — 6-node K3s HA cluster already deployed.

---

## Architecture

```
k3s-net (10.10.0.0/24)              monitoring-net (10.20.0.0/24)
├── K3s-srv-1      10.10.0.11       ├── Zabbix-srv     10.20.0.10
├── K3s-srv-2      10.10.0.12       ├── Wazuh-srv      10.20.0.11
├── K3s-srv-3      10.10.0.13       ├── Prometheus-srv 10.20.0.12
├── K3s-agent-1    10.10.0.31       └── Grafana-srv    10.20.0.13
├── K3s-agent-2    10.10.0.32
├── K3s-agent-3    10.10.0.33
└── OPNsense WAN   10.10.0.254 ──── OPNsense LAN 10.20.0.254
```

OPNsense routes traffic between the two isolated networks.

---

## Stack

|Tool|Role|
|---|---|
|**Zabbix**|Host metrics (CPU, RAM, disk, network) via Zabbix agent + infrastructure alerting|
|**Prometheus**|K8s/application metrics — pods, deployments, namespaces|
|**Grafana**|NOC dashboards — unified visualization (Zabbix + Prometheus datasources)|
|**Wazuh**|SOC — SIEM, FIM, brute force detection, CVE scan|
|**OPNsense**|Router/firewall between k3s-net and monitoring-net|
|**Ansible**|Agent deployment automation across all nodes|

---

## NOC / SOC

```
NOC → Grafana (http://10.20.0.13:3000)
      ├── Datasource Zabbix      → host metrics (VMs/nodes)
      └── Datasource Prometheus  → K8s pod/deployment metrics

SOC → Wazuh Dashboard (https://10.20.0.11)
      ├── Security alerts
      ├── File Integrity Monitoring
      ├── Brute force detection
      └── CVE scan
```

---

## Environment

- **Host**: ThinkPad E14 Gen 5 (i7-13700H, 31GB RAM, 476GB NVMe)
- **Hypervisor**: KVM/libvirt on Arch Linux
- **VM OS**: Ubuntu 26.04 LTS Server

---

## Phases

|Phase|Description|Status|
|---|---|---|
|0|GitHub setup + repo structure|✅|
|1|Network, VMs, OPNsense, persistent routes|✅|
|2|Install Zabbix, Wazuh, Prometheus, Grafana|✅|
|3|NOC config — Grafana datasources + dashboards|🔄|
|4|SOC config — Wazuh rules + FIM + CVE|⏳|
|5|Ansible — agent deployment across all nodes|⏳|
|6|Documentation finalization|⏳|

---

## Documentation

- 01 — Network & VMs
- 02 — Zabbix
- 03 — Wazuh
- 04 — Prometheus
- 05 — Grafana
- 06 — NOC Config
- Architecture Decisions
- Troubleshooting

---

## Quick start

```bash
# Start the K3s cluster
k3s-start

# Start the monitoring stack
monitoring-start

# Stop
k3s-stop
monitoring-stop
```