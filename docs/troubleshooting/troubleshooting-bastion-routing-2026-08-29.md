# Troubleshooting: Connectivité bastion → k3s-net / monitoring-net cassée

**Date**: 2026-08-29
**Repo concerné**: K3s-lab / K3s-lab-monitoring / Bastion-lab (transverse)
**Sévérité**: Haute — bloquait tout accès SSH/Ansible depuis le poste opérateur et le bastion vers la quasi-totalité du parc
**Statut**: Résolu

---

## Symptôme initial

Connexions SSH silencieusement bloquées (timeout, aucune erreur explicite) depuis `My-ship` vers plusieurs VMs du lab (`wazuh-srv`, `grafana-srv`, `k3s-agent-1/2/3`), alors que d'autres hôtes (`k3s-srv-1`, `ansible-srv`) restaient joignables. Aucun changement de configuration connu n'avait précédé l'apparition du problème.

## Fausses pistes explorées (dans l'ordre)

Documentées ici pour éviter de les re-parcourir en cas de symptôme similaire à l'avenir.

1. **`ufw` sur My-ship** — désactivé puis réactivé par erreur en début de diagnostic. Vérifié : les 30 règles étaient intactes avant/après, aucun impact réel. *Non responsable.*
2. **Restriction sshd par IP sur les VMs cible** — hypothèse initiale vu le hardening Bastion-lab (sshd restreint à l'IP du bastion). Écartée : le comportement était un timeout muet, pas un refus explicite de sshd.
3. **Firewall OPNsense (règles par interface)** — vérifié interface par interface (BASTION, LAN/monitoring-net). Les règles étaient correctes et permissives (`pass` depuis `10.30.0.10` vers `*`). Confirmé via **Firewall → Log Files → Live View** : les paquets sortant du bastion étaient bien vus et autorisés (`pass`) sur l'interface `k3snet` en sortie.

## Cause racine identifiée

**Absence de route statique de retour vers `mgmt-net` (10.30.0.0/24) sur la quasi-totalité des VMs du parc.**

Audit réalisé via un playbook Ansible ad-hoc (`ip route` sur toutes les VMs de l'inventory) : la majorité des hôtes (k3s-srv-2/3, k3s-agent-1/2/3, k3s-db, zabbix-srv, wazuh-srv, prometheus-srv, grafana-srv) n'avaient **aucune route par défaut ni route explicite vers `10.30.0.0/24`**. Ces VMs reçoivent leurs routes principalement via DHCP (`proto dhcp`), qui ne couvrait pas mgmt-net.

Conséquence : le paquet aller (bastion → VM cible) passait bien le firewall OPNsense et atteignait la VM. Mais la VM, ne sachant pas comment router sa réponse vers `10.30.0.0/24`, droppait le paquet retour localement. D'où le symptôme de timeout muet malgré un firewall qui laissait tout passer.

Seules `k3s-srv-1`, `load-srv` et `ansible-srv` avaient une route par défaut générique qui couvrait accidentellement ce cas — d'où leur comportement différent dès le début du diagnostic.

### Méthode de confirmation

1. `Firewall → Log Files → Live View` filtré sur l'IP du bastion → paquet aller confirmé `pass`
2. `Firewall → Diagnostics → States` → aucun état actif initié depuis le bastion visible → confirme que la réponse ne revient jamais
3. `ip route` sur les VMs cible (via playbook Ansible) → absence de route vers `10.30.0.0/24` confirmée

## Fix appliqué

Déploiement d'un fichier netplan canonique (`/etc/netplan/99-routes.yaml`) sur les 13 VMs du parc, ajoutant explicitement les routes croisées entre tous les réseaux du lab (k3s-net, monitoring-net, mgmt-net, k8s-ha-net), via Ansible :

```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      routes:
        - to: 10.20.0.0/24
          via: 10.10.0.254   # (ou 10.20.0.254 selon le côté du réseau)
        - to: 10.30.0.0/24
          via: 10.10.0.254
        - to: 10.40.0.0/24
          via: 10.10.0.254
```

`netplan apply` reconcilie l'état complet du fichier avec le système — toute route précédemment définie dans ce même fichier et absente de la nouvelle version est automatiquement supprimée, ce qui a permis un cleanup implicite en plus de l'ajout.

### Cas particulier  k3s-srv-1

Seule VM du parc en IP statique pure (`dhcp4: false` dans `/etc/netplan/50-cloud-init.yaml`, contrairement aux autres qui reçoivent leur IP via DHCP). `netplan apply` exécuté via Ansible a coupé le canal SSH utilisé par la connexion elle-même pendant le renouvellement d'interface, bloquant le playbook. Contournement : exclusion temporaire du run Ansible (`--limit 'all:!k3s-srv-1'`), application manuelle via `virsh console` (connexion locale, pas de risque de coupure), puis réintégration confirmée au run suivant.

## Validation post-fix

SSH direct depuis `My-ship` (via ProxyJump bastion complet) testé et confirmé sur : `wazuh-srv`, `k3s-agent-1`, `k3s-srv-1`, `k3s-srv-2`, `k3s-srv-3`. Ping bidirectionnel bastion ↔ k3s-net et bastion ↔ monitoring-net confirmé à 0% de perte.

## Leçons / points de vigilance pour la suite

- **Toujours privilégier les logs firewall en temps réel** (`Live View`) plutôt que de deviner règle par règle — a permis d'écarter OPNsense comme coupable en quelques minutes une fois consulté, après plus d'une heure passée sur les règles statiques.
- **Un paquet aller autorisé ne garantit pas une communication bidirectionnelle** — toujours vérifier la route de retour sur l'hôte cible, pas seulement le firewall du chemin aller.
- **DHCP et configuration statique coexistant sur le même lab** créent des topologies de routage hétérogènes difficiles à auditer visuellement — préférer une source unique de vérité pour les routes inter-réseaux (le fichier `99-routes.yaml` centralisé sert maintenant cet objectif).
- Ce bug explique très probablement, rétroactivement, une partie du problème `k3s-db` documenté comme "non résolu" dans le troubleshooting Ansible historique (échec de ping systématique). À garder en tête si un symptôme similaire réapparaît sur un hôte non encore couvert.

## Effort de résolution

Diagnostic méthodique par élimination : agent SSH → clés orphelines → règles firewall interface par interface → logs firewall temps réel → routage. Environ 2h30 de troubleshooting actif.
