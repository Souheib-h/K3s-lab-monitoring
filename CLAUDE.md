# CLAUDE.md — contexte pour les sessions Claude Code

Mémoire de travail du homelab (état au 2026-09-30). Les décisions détaillées
sont dans [DECISIONS.md](DECISIONS.md) ; ce fichier dit où on en est et comment
travailler.

## Façon de travailler

- Répondre en **français** si la conversation commence en français ; direct,
  détaillé, tableaux et blocs de code copiables.
- Toute modification passe par une **PR** ; ne fusionner que quand l'utilisateur
  dit « merge ». Fusion en **merge commit** (jamais squash) pour pouvoir relire
  ou annuler chaque changement.
- Pour les règles OPNsense : instructions explicites pas à pas (interface,
  action, source, destination, ports), l'utilisateur les applique lui-même.
- Ne jamais demander ni afficher de mot de passe/token en clair ; masquer dans
  les sorties (`sed -E 's#(//[^:]+:)[^@]+@#\1****@#'`).
- Ne jamais affirmer un test que l'utilisateur n'a pas lancé : n'écrire dans la
  doc que ce qui est prouvé par une sortie.
- Avant de pousser : `ansible-lint` (profil production) et `yamllint` dans
  `configs/ansible/`, liens vérifiés par la CI `docs` (lychee).

## Repos

| Repo | Statut |
|---|---|
| K3s-lab-monitoring | actif — repo central (NOC/SOC, Ansible, ADR) |
| Bastion-lab | actif — bastion SSH ; §9 de `ssh-setup.md` : le cluster ne joint pas le bastion |
| Dot-files | actif (privé) — `starship.toml` + linter `.github/scripts/check_starship.py` |
| Souheib-h | actif — README de profil, vérif hebdo des liens |
| K3s-lab | **archivé** (public, lecture seule) — remplacé par un cluster **kubeadm** |
| Packet-Analysis-Lab | **en pause** — ne rien toucher sans demande explicite |

## État du lab

- **C1 segmentation — terminé, vérifié le 2026-09-30** : règles k3snet R1–R5
  (amendement ADR-016), sortie Internet de k3s-net via OPNsense (ADR-017),
  option DHCP 121 sur une seule ligne, route par défaut dans
  `playbooks/fix-routes.yml`. load-srv (Alpine/udhcpc) ignore l'option 121.
- **C2 K3s — accepté comme limites connues**, pas corrigé ; tout correctif va
  dans le cluster kubeadm : checklist K1–K7 / N1–N3 / M1–M3 dans
  `K3s-lab/docs/08-suite-kubeadm.md`.
- ansible-srv exécute les playbooks depuis un clone git sparse de ce repo
  (pas de copie manuelle).
- My-ship (hyperviseur Arch) a été réinstallé : agents de l'hyperviseur
  inactifs (ADR-012).

## À faire (au choix de l'utilisateur)

| Chantier | Contenu |
|---|---|
| C3 Wazuh | `white_list` active response (bastion 10.30.0.10, ansible-srv 10.20.0.125), FIM dans `agent.conf`, mot de passe d'enrôlement authd |
| C4 | services exposés |
| C5 | bastion |
| C6 | inventaire : loki-srv et Bastion-srv absents, agents hyperviseur |
| kubeadm | règles CLUSTERHANET, option 121 sur cluster-ha-net, fix-routes |
| C7 | Packet lab — en pause |

À vérifier : accès Internet de prometheus-srv (échec vers
changelogs.ubuntu.com) ; disque ansible-srv à 82 % (`~/alloy-staging` est
nettoyé par `install-alloy.yml`, le déplacer vers `~/alloy-backup` pour le
garder) ; nœuds sur des versions d'Ubuntu différentes (26.04.1 proposé).
