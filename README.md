# K3s-lab-monitoring

NOC/SOC lab — Monitoring et sécurité d'un cluster K3s HA from scratch.

> Prérequis : [K3s-lab](https://github.com/Souheib-h/K3s-lab) — cluster K3s HA 6 nœuds déjà déployé.

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

OPNsense joue le rôle de routeur entre les deux réseaux.

---

## Stack

| Outil | Rôle |
|---|---|
| **Zabbix** | Monitoring infra — CPU, RAM, Disk, réseau des nodes |
| **Prometheus** | Métriques K8s — pods, deployments, namespaces |
| **Grafana** | NOC Dashboard — visualisation Zabbix + Prometheus |
| **Wazuh** | SOC — SIEM, IDS, File Integrity, CVE scan |
| **OPNsense** | Routeur/Firewall entre k3s-net et monitoring-net |
| **Ansible** | Automatisation — déploiement agents sur les nodes |

---

## NOC / SOC

```
NOC → Grafana (http://10.20.0.13:3000)
      ├── Datasource Zabbix   → santé VMs/nodes
      └── Datasource Prometheus → pods K8s

SOC → Wazuh Dashboard (https://10.20.0.11)
      ├── Alertes sécurité
      ├── File Integrity Monitoring
      └── CVE scan
```

---

## Environnement

- **Host** : ThinkPad E14 Gen 5 (i7-13700H, 31GB RAM, 476GB NVMe)
- **Hyperviseur** : KVM/libvirt sur Arch Linux
- **OS VMs** : Ubuntu 26.04 LTS Server

---

## Phases

| Phase | Description | Status |
|---|---|---|
| 0 | Préparation GitHub + structure | ✅ |
| 1 | Réseau, VMs, OPNsense, routes persistantes | ✅ |
| 2 | Installation Zabbix, Wazuh, Prometheus, Grafana | 🔄 |
| 3 | Configuration NOC (Grafana dashboards) | ⏳ |
| 4 | Configuration SOC (Wazuh rules) | ⏳ |
| 5 | Ansible — déploiement agents sur les nodes | ⏳ |
| 6 | Finalisation documentation | ⏳ |

---

## Documentation

- [01 — Réseau & VMs](docs/01-reseau.md)
- [02 — Zabbix](docs/02-zabbix.md)
- [03 — Wazuh](docs/03-wazuh.md)
- [04 — Prometheus](docs/04-prometheus.md)
- [05 — Grafana](docs/05-grafana.md)
- [06 — Ansible](docs/06-ansible.md)

---

## Démarrage rapide

```bash
# Démarrer le cluster K3s
k3s-start

# Démarrer le stack monitoring
monitoring-start

# Arrêter
k3s-stop
monitoring-stop
```