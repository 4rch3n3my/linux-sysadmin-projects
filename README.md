# linux-sysadmin-training

Parcours de montée en compétences orienté **Ingénieur Systèmes Linux N2/N3**, construit autour des exigences réelles des postes d'administration infrastructure en environnement critique.

Chaque module couvre un axe technique du poste : administration RHEL, scripting, supervision, virtualisation, stockage et DevOps. Les exercices sont conçus pour préparer aussi bien les situations réelles que les entretiens techniques.

---

## Structure du parcours

| # | Module | Compétences ciblées | Status |
|---|--------|---------------------|--------|
| [01](./01-rhel-administration-et-MCO/) | **Administration RHEL & Maintien en Conditions Opérationnelles** | gestion système, MCO, incidents N2/N3, optimisation | 🔜 |
| [02](./02-scripting-bash-python-automatisation/) | **Scripting Bash & Python — Automatisation des tâches d'exploitation** | scripts d'exploitation, automatisation, monitoring custom | 🔜 |
| [03](./03-supervision-centreon-nagios-alerting/) | **Supervision avec Centreon & Nagios — Détection et alerting** | installation, configuration, sondes, escalades d'alertes | 🔜 |
| [04](./04-virtualisation-vmware-vsphere-gestion-vms/) | **Virtualisation VMware vSphere — Gestion des VMs en production** | déploiement VMs, snapshots, migration, haute disponibilité | 🔜 |
| [05](./05-stockage-netapp-administration-baies/) | **Stockage & Sauvegarde NetApp — Administration des baies de stockage** | volumes, NFS/CIFS, snapshots, politique de sauvegarde | 🔜 |
| [06](./06-devops-ansible-docker-kubernetes-jenkins/) | **DevOps — Ansible, Docker, Kubernetes, Jenkins** | automatisation, conteneurs, orchestration, CI/CD | 🔜 |

---

## Format des exercices

Chaque module contient 4 types d'exercices :

**🔧 Mini-projets** — un objectif concret à atteindre de A à Z, documenté comme en production.

**🚨 Scénarios incident** — une situation de panne ou dégradation à diagnostiquer et résoudre, avec horodatage et procédure de reprise.

**✅ Checklists** — listes de vérification applicables directement en production ou lors d'une prise en main d'un nouveau système.

**🎯 Questions d'entretien** — questions techniques fréquentes sur le sujet, avec les réponses attendues par niveau (technicien / ingénieur).

---

## Environnement de lab

```
Proxmox VE 8.4 (192.168.1.50)
├── rhel-lab       (RHEL 9 )  → administration, scripting, MCO
├── supervision-vm (Debian 12)           → Centreon ou Nagios
├── devops-vm      (Debian 12)           → Docker, Ansible, Jenkins
└── VMs existantes → sources de logs et métriques réelles
```

---

## Progression recommandée

```
01 RHEL → socle indispensable, à maîtriser en premier
    ↓
02 Bash/Python → automatiser ce qu'on administre
    ↓
03 Supervision → monitorer ce qu'on exploite
    ↓
04 VMware → contexte d'exécution de la majorité des infras
    ↓
05 NetApp → stockage critique, souvent en entretien N3
    ↓
06 DevOps → différenciateur, souvent demandé comme "vrai plus"
```

---

## Auteur

**Thomas PICHOT** — Administrateur Systèmes & Réseaux, orienté Sécurité Défensive  
5 ans de terrain Linux/Windows · Diplômé Bac+3/4 février 2026 · Toulouse

[GitHub](https://github.com/4rch3n3my) · [LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45)
