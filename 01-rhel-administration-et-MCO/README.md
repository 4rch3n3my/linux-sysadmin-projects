# Module 01 — Administration RHEL & Maintien en Conditions Opérationnelles

> RHEL 9.7 — Administré sur Proxmox VE 8
> Axes couverts : packages, services, utilisateurs, LVM, sécurité, logs, planification, performances

---

## Environnement

| Composant | Détail |
|---|---|
| Hyperviseur | Proxmox VE 8 |
| OS | Red Hat Enterprise Linux 9.7 (Plow) |
| Hostname | rhel9-lab |
| IP | 192.168.1.x |

---

## Structure du module

```
01-rhel-administration-et-MCO/
├── README.md
├── mini-projets/
│   ├── 01-configuration-initiale.md
│   ├── 02-gestion-utilisateurs-permissions.md
│   ├── 03-lvm-stockage.md
│   ├── 04-firewalld-selinux.md
│   ├── 05-logs-journald.md
│   ├── 06-cron-planification.md
│   └── 07-performances-optimisation.md
├── incidents/
│   ├── 01-service-en-echec.md
│   ├── 02-disque-plein.md
│   ├── 03-utilisateur-verrouille.md
│   └── 04-selinux-bloque-service.md
├── checklists/
│   ├── prise-en-main-nouveau-serveur.md
│   └── mco-quotidien.md
└── entretiens/
    └── questions-reponses-rhel.md
```

---

## 🔧 Mini-projets

### 01 — Configuration initiale du serveur

**Enjeu :** Avant toute intervention sur un serveur, il faut s'assurer que la base est saine : version OS connue, système à jour, identité réseau correcte et heure synchronisée. Un serveur mal configuré à ce niveau peut causer des problèmes d'authentification Kerberos (AD), de certificats SSL expirés ou de logs inexploitables.

```bash
# Vérifier la version RHEL
# Permet de savoir exactement sur quoi on travaille avant toute intervention
[root@localhost ~]# cat /etc/redhat-release
Red Hat Enterprise Linux release 9.7 (Plow)

# Enregistrer le système sur Red Hat
# Sans cette étape, aucun dépôt n'est disponible et dnf échoue
# Le compte développeur Red Hat est gratuit pour 1 machine : developers.redhat.com
[root@localhost ~]# subscription-manager register --username ******* --password '********************'
Inscription sur : subscription.rhsm.redhat.com:443/subscription
Le système a été enregistré avec l'ID : *****-******-**********-*****
Le nom de système enregistré est : localhost.localdomain

# Mettre à jour le système
# En production : toujours commencer par une mise à jour pour éviter
# d'opérer sur un système avec des CVE connues non corrigées
[root@localhost ~]# dnf update -y
[...]
Terminé !

# Configurer le hostname
# Le hostname est utilisé dans les logs, les certificats et l'intégration AD
# Un hostname "localhost" en production est une erreur critique
[root@localhost ~]# hostnamectl set-hostname rhel9-lab
[root@localhost ~]# hostnamectl status
 Static hostname: rhel9-lab
       Icon name: computer-vm
         Chassis: vm 🖴
      Machine ID: 1a176aa52269400f95e02be2db21b169
         Boot ID: 8772e5624987451b8b22a9fc93d95be2
  Virtualization: kvm
Operating System: Red Hat Enterprise Linux 9.7 (Plow)
          Kernel: Linux 5.14.0-611.5.1.el9_7.x86_64
    Architecture: x86-64
 Hardware Vendor: QEMU

# Configurer le fuseau horaire
# Indispensable pour la cohérence des logs et l'authentification Kerberos
# Un écart de plus de 5 minutes avec le DC rompt l'auth AD
[root@localhost ~]# timedatectl set-timezone Europe/Paris
[root@localhost ~]# timedatectl status
               Local time: mer. 2026-03-11 13:50:45 CET
           Universal time: mer. 2026-03-11 12:50:45 UTC
                 RTC time: mer. 2026-03-11 12:50:45
                Time zone: Europe/Paris (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

---

### 02 — Gestion des packages RPM / DNF

**Enjeu :** DNF est le gestionnaire de packages de RHEL. Il résout les dépendances, gère les dépôts et conserve un historique complet des opérations. Savoir l'utiliser correctement est indispensable pour installer des logiciels, corriger des CVE et maintenir un système en conditions opérationnelles. Une erreur courante en production est d'installer des packages sans vérifier leur provenance ou de ne jamais appliquer les mises à jour de sécurité.

```bash
# Rechercher un package par nom ou description
# Utile quand on ne connaît pas le nom exact du package
dnf search htop

# Installer un package
# -y : répond oui automatiquement à toutes les confirmations
dnf install -y htop

# Supprimer un package
# Attention : dnf retire aussi les dépendances devenues orphelines
dnf remove -y htop

# Lister les packages installés
# Utile pour auditer ce qui tourne sur un serveur
dnf list installed

# Voir l'historique complet des opérations DNF
# Chaque ligne = une transaction (install, update, remove)
dnf history

# Annuler la dernière opération
# Utile si une mise à jour a cassé quelque chose — rollback propre
dnf history undo last

# Vérifier les mises à jour disponibles sans les appliquer
dnf check-update

# Installer uniquement les mises à jour de sécurité
# En MCO, on peut vouloir cibler uniquement les correctifs CVE
# sans toucher aux mises à jour fonctionnelles
dnf update --security -y

# Lister les dépôts actifs
# Permet de vérifier que les bons repos sont disponibles
[root@localhost ~]# dnf repolist
Mise à jour des référentiels de gestion des abonnements.
id du dépôt                              nom du dépôt
rhel-9-for-x86_64-appstream-rpms         Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms            Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)

# Activer / désactiver un dépôt
dnf config-manager --enable <repo>
dnf config-manager --disable <repo>

# Nettoyer le cache local DNF
# Libère de l'espace et force le rechargement des métadonnées
dnf clean all
```

---

### 03 — Gestion des services systemd

**Enjeu :** systemd est le système d'initialisation de RHEL. Il gère le démarrage, l'arrêt et la supervision de tous les services. Un administrateur N2/N3 doit savoir diagnostiquer un service en échec, comprendre ses dépendances et analyser les temps de démarrage. En production, un service critique qui ne redémarre pas après un reboot peut causer une indisponibilité majeure.

```bash
# Lister tous les services actifs
# Filtrer sur "running" pour voir uniquement les services qui tournent
[root@localhost ~]# systemctl list-units --type=service | grep running
  auditd.service       loaded active running Security Auditing Service
  chronyd.service      loaded active running NTP client/server
  crond.service        loaded active running Command Scheduler
  firewalld.service    loaded active running firewalld - dynamic firewall daemon
  NetworkManager.service loaded active running Network Manager
  rsyslog.service      loaded active running System Logging Service
  sshd.service         loaded active running OpenSSH server daemon
  [...]

# Lister les services en échec
# Premier réflexe après un reboot ou une anomalie
systemctl list-units --state=failed

# Démarrer un service
systemctl start <service>

# Arrêter un service
systemctl stop <service>

# Redémarrer un service (stop + start)
# Provoque une courte interruption — à éviter en prod si possible
systemctl restart <service>

# Recharger la configuration sans redémarrer
# Préférable à restart quand le service le supporte (nginx, sshd...)
systemctl reload <service>

# Activer un service au démarrage
# Sans cette commande, le service ne redémarre pas après reboot
systemctl enable <service>

# Activer ET démarrer en une seule commande
systemctl enable --now <service>

# Désactiver le démarrage automatique
systemctl disable <service>

# Vérifier le statut détaillé d'un service
# Affiche l'état, le PID, les dernières lignes de log et le code de sortie
systemctl status sshd

# Voir les dépendances d'un service
# Utile pour comprendre pourquoi un service ne démarre pas
# (dépendance manquante ou en échec)
systemctl list-dependencies sshd

# Analyser le temps de démarrage global
# Permet d'identifier les services qui ralentissent le boot
[root@localhost ~]# systemd-analyze
Startup finished in 1.448s (kernel) + 4.705s (initrd) + 1min 3.202s (userspace) = 1min 9.356s
graphical.target reached after 1min 3.177s in userspace.

# Détailler service par service ce qui prend du temps au boot
systemd-analyze blame
```

---

### 04 — Gestion des utilisateurs / permissions

**Enjeu :** La gestion des identités est un pilier de la sécurité système. En production, chaque service doit tourner sous un compte dédié avec les droits minimaux nécessaires — c'est le principe du moindre privilège. Donner des droits trop larges expose le système en cas de compromission d'un compte.

#### Gestion des comptes

```bash
# Créer un utilisateur avec un répertoire home et un shell bash
# -m : crée le répertoire /home/<user>
# -s : définit le shell par défaut
[root@localhost ~]# useradd -m -s /bin/bash user1
[root@localhost ~]# passwd user1
Changement de mot de passe pour l'utilisateur user1.
Nouveau mot de passe :
Retapez le nouveau mot de passe :
passwd : mise à jour réussie de tous les jetons d'authentification.

# Créer un groupe
[root@localhost ~]# groupadd testgroup

# Ajouter un utilisateur à un groupe secondaire
# -a : ajouter (sans -a, les groupes existants sont écrasés — attention)
# -G : spécifie le groupe secondaire
[root@localhost ~]# usermod -aG testgroup user1

# Vérifier les groupes d'un utilisateur
[root@localhost ~]# id user1
uid=1001(user1) gid=1001(user1) groupes=1001(user1),1002(testgroup)

# Verrouiller un compte sans le supprimer
# Cas d'usage : départ d'un collaborateur, compte à désactiver immédiatement
usermod -L user1

# Déverrouiller un compte
usermod -U user1

# Renommer un utilisateur
usermod -l user2 user1

# Supprimer un utilisateur ET son répertoire home
# -r : supprime /home/user2 
# À faire uniquement après avoir archivé les données si nécessaire
userdel -r user1
```

#### Permissions fichiers

Linux gère les permissions sur 3 niveaux : **propriétaire (u)**, **groupe (g)**, **autres (o)**.
Chaque niveau a 3 droits : **r (read=4)**, **w (write=2)**, **x (execute=1)**.

```bash
# Lecture des permissions avec ls -la
# Chaque ligne se lit ainsi :
[root@localhost ~]# ls -la /var/app/config/
total 0
drwxr-xr-x. 2 root root 22 11 mars  15:09 .
drwxr-xr-x. 3 root root 20 11 mars  15:09 ..
-rw-r--r--. 1 root root  0 11 mars  15:09 app.conf
#└┤└─┤└─┤    └──┤ └──┤
# │  │  │       │    └── groupe propriétaire
# │  │  │       └─────── utilisateur propriétaire
# │  │  └─────────────── droits des autres (r-- = 4)
# │  └────────────────── droits du groupe  (r-- = 4)
# └───────────────────── droits du proprio (rw- = 6)
# - en début de ligne = fichier | d = répertoire | l = lien symbolique
```

```bash
# --- Cas 1 : dossier applicatif ---
# Objectif : alice (proprio) a tous les droits, le groupe devops peut lire et traverser,
# les autres n'ont aucun accès — typique d'un dossier applicatif en production

[root@localhost ~]# chmod 750 /var/app
[root@localhost ~]# chown alice:devops /var/app
[root@localhost ~]# ls -la /var/
drwxr-x---.  3 alice devops   20 11 mars  15:09 app
# 750 = propriétaire(rwx=7) groupe(r-x=5) autres(---=0)
# Avant : drwxr-xr-x. root root → tout le monde pouvait lire
# Après : drwxr-x---. alice devops → les autres n'ont plus accès
```

```bash
# --- Cas 2 : fichier de configuration ---
# Objectif : le fichier est lisible par tous (utile pour une appli multi-utilisateurs)
# mais seul alice peut le modifier

[root@localhost ~]# chmod 644 /var/app/config/app.conf
[root@localhost ~]# chown alice:devops /var/app/config/app.conf
[root@localhost ~]# ls -la /var/app/config/
-rw-r--r--. 1 alice devops  0 11 mars  15:09 app.conf
# 644 = propriétaire(rw-=6) groupe(r--=4) autres(r--=4)
# Avant : root root → fichier appartenant à root
# Après : alice devops → propriété transférée à l'utilisateur applicatif
```

```bash
# --- Cas 3 : clé SSH privée ---
# Objectif : la clé privée ne doit être lisible que par son propriétaire
# SSH refuse de fonctionner si les permissions sont trop ouvertes (erreur "bad permissions")

[root@localhost ~]# ls -la ~/.ssh/id_rsa
-rw-r--r--. 1 root root 0 11 mars  15:10 id_rsa   # ← trop ouvert, SSH refuserait

[root@localhost ~]# chmod 600 ~/.ssh/id_rsa
[root@localhost ~]# ls -l ~/.ssh/
-rw-------. 1 root root 0 11 mars  15:10 id_rsa   # ← correct
# 600 = propriétaire(rw-=6) groupe(---=0) autres(---=0)
```

```bash
# --- Cas 4 : application récursive ---
# Objectif : aligner les permissions de tout l'arbre /var/app d'un coup
# -R applique chmod et chown sur le dossier ET tout son contenu

[root@localhost ~]# chmod -R 750 /var/app
[root@localhost ~]# chown -R alice:devops /var/app
[root@localhost ~]# ls -la /var/app/config/
drwxr-x---. 2 alice devops 22 11 mars  15:09 .
drwxr-x---. 3 alice devops 20 11 mars  15:09 ..
-rwxr-x---. 1 alice devops  0 11 mars  15:09 app.conf
# Tous les éléments appartiennent maintenant à alice:devops avec les droits 750
```

> ⚠️ Attention au `chmod -R 750` sur des fichiers : le bit `x` (exécution) est positionné sur les fichiers aussi, ce qui n'est pas toujours souhaitable. En production, on distingue souvent dossiers et fichiers avec `find` :
> ```bash
> find /var/app -type d -exec chmod 750 {} \;
> find /var/app -type f -exec chmod 640 {} \;
> ```

#### ACL étendues

Les permissions classiques sont limitées : un seul propriétaire, un seul groupe.
Les **ACL** permettent de donner des droits fins à des utilisateurs ou groupes supplémentaires **sans changer le propriétaire**.

```bash
# Cas d'usage : bob doit pouvoir lire /data/rapports
# sans appartenir au groupe propriétaire
setfacl -m u:bob:r-- /data/rapports

# Cas d'usage : le groupe audit doit pouvoir lire et exécuter les scripts
setfacl -m g:audit:r-x /scripts

# Vérifier les ACL en place
getfacl /data/rapports
# file: data/rapports
# owner: alice
# group: devops
# user::rwx
# user:bob:r--     ← ACL ajoutée pour bob
# group::r-x
# mask::r-x
# other::---

# Supprimer une ACL spécifique
setfacl -x u:bob /data/rapports

# Supprimer toutes les ACL
setfacl -b /data/rapports
```

> ⚠️ Le filesystem doit être monté avec l'option `acl`. Sur RHEL 9 / XFS c'est actif par défaut.

#### Sudo — élévation de privilèges

`sudo` permet d'exécuter des commandes en tant que root **de façon contrôlée et tracée**, sans partager le mot de passe root. Toutes les actions sudo sont loggées dans `/var/log/secure`.

```bash
# Ajouter un utilisateur au groupe wheel
# wheel = groupe sudoers par défaut sur RHEL
# Cas d'usage : donner les droits admin complets à un admin système
usermod -aG wheel alice

# Vérifier
id alice
# uid=1001(alice) gid=1001(alice) groups=1001(alice),10(wheel)

# visudo — éditer les règles sudo de façon sécurisée
# Vérifie la syntaxe avant de sauvegarder
# Une erreur dans sudoers sans visudo peut rendre sudo inutilisable
visudo

# Syntaxe : qui  où=(en tant que qui)  commande
# alice  ALL=(ALL)  ALL
# └──┘   └──┘ └──┘  └─┘
# user  hôtes  runas  commandes autorisées

# Cas d'usage : alice peut tout faire avec son mot de passe
alice ALL=(ALL) ALL

# Cas d'usage : compte applicatif peut redémarrer un service sans mot de passe
# Utile pour les scripts d'exploitation automatisés
appuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp

# Cas d'usage : helpdesk peut gérer les mots de passe utilisateurs
# mais PAS le compte root (! = exclusion)
helpdesk ALL=(ALL) NOPASSWD: /usr/bin/passwd [A-Za-z]*, !/usr/bin/passwd root

# Vérifier ce qu'un utilisateur peut faire avec sudo
sudo -l -U alice
```

> ⚠️ Ne jamais mettre `NOPASSWD: ALL` sur un compte utilisateur — c'est une élévation de privilèges sans friction ni traçabilité.

---

### 05 — LVM / Stockage

**Enjeu :** LVM (Logical Volume Manager) est le standard de gestion du stockage sur Linux en production. Il permet d'étendre des volumes à chaud, de créer des snapshots avant une intervention risquée et de réorganiser le stockage sans interruption de service. Ne pas utiliser LVM, c'est se condamner à des interventions risquées dès que le disque est plein.

```bash
# Lister les disques et partitions disponibles
# lsblk donne une vue arborescente claire
lsblk
fdisk -l

# --- Création d'un volume LVM ---

# Étape 1 : Initialiser le disque comme Physical Volume (PV)
# Le disque /dev/sdb devient utilisable par LVM
pvcreate /dev/sdb
pvdisplay   # vérification

# Étape 2 : Créer un Volume Group (VG)
# Le VG est un pool de stockage qui regroupe un ou plusieurs PV
# On peut ajouter d'autres disques au VG plus tard sans interruption
vgcreate vg_data /dev/sdb
vgdisplay   # vérification

# Étape 3 : Créer un Logical Volume (LV)
# Le LV est l'équivalent d'une partition — c'est lui qu'on formate et monte
# -L : taille fixe | -l : taille en % du VG (ex: -l 100%FREE)
lvcreate -L 5G -n lv_data vg_data
lvdisplay   # vérification

# Étape 4 : Formater le LV
# XFS est le filesystem par défaut sur RHEL — performant et supporte l'extension à chaud
mkfs.xfs /dev/vg_data/lv_data

# Étape 5 : Monter le volume
mkdir /data
mount /dev/vg_data/lv_data /data

# Étape 6 : Rendre le montage permanent au reboot
# Sans cette ligne dans fstab, le volume n'est pas monté après redémarrage
echo '/dev/vg_data/lv_data /data xfs defaults 0 0' >> /etc/fstab
mount -a   # vérifier que fstab ne contient pas d'erreur

# --- Extension à chaud ---

# Étendre le LV de 2 Go supplémentaires
# Possible sans démonter le volume — aucune interruption de service
lvextend -L +2G /dev/vg_data/lv_data

# Étendre le filesystem pour qu'il utilise l'espace ajouté
xfs_growfs /data        # pour XFS
# resize2fs /dev/vg_data/lv_data   # pour ext4

# Vérifier l'espace disque
df -hT          # vue par filesystem
du -sh /data/*  # taille des dossiers dans /data
```

---

### 06 — Firewalld / SELinux

**Enjeu :** RHEL embarque deux couches de sécurité complémentaires. **Firewalld** filtre le trafic réseau entrant/sortant. **SELinux** restreint ce que chaque processus peut faire sur le système, indépendamment des permissions UNIX. Désactiver SELinux en production est une faute grave — il vaut mieux apprendre à le gérer.

#### Firewalld

Firewalld utilise le concept de **zones** : chaque interface réseau est assignée à une zone qui définit un niveau de confiance. La zone `public` est la plus restrictive, `trusted` autorise tout.

```bash
# Vérifier que firewalld est actif
systemctl status firewalld
firewall-cmd --state

# Lister toutes les règles actives sur la zone courante
firewall-cmd --list-all

# Ajouter une règle permanente (survit au reboot)
# --permanent seul ne prend pas effet immédiatement → toujours suivre d'un --reload
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=8080/tcp

# Supprimer une règle
firewall-cmd --permanent --remove-service=http

# Appliquer les règles permanentes sans redémarrer firewalld
firewall-cmd --reload

# Lister les zones disponibles et la zone active
firewall-cmd --get-zones
firewall-cmd --get-active-zones
```

#### SELinux

SELinux fonctionne avec 3 modes : **enforcing** (actif, bloque), **permissive** (actif, log sans bloquer), **disabled** (désactivé).

```bash
# Vérifier le mode actuel
getenforce
sestatus

# Passer temporairement en permissif
# Utile pour diagnostiquer si SELinux est responsable d'un problème
# Attention : temporaire uniquement, repasse en enforcing après diagnostic
setenforce 0

# Repasser en enforcing
setenforce 1

# Modifier le mode de façon permanente (nécessite reboot)
# Éditer /etc/selinux/config
# SELINUX=enforcing | permissive | disabled

# Voir les refus SELinux récents
# AVC = Access Vector Cache = log des refus d'accès
ausearch -m avc -ts recent

# Analyse détaillée avec suggestions de correction
sealert -a /var/log/audit/audit.log

# Corriger le contexte SELinux d'un fichier
# Cas d'usage : fichier déplacé manuellement qui a perdu son contexte
restorecon -Rv /var/www/html

# Lister et modifier les booléens SELinux
# Les booléens permettent d'activer des comportements prédéfinis sans écrire de politique
getsebool -a | grep http
setsebool -P httpd_can_network_connect on   # -P = permanent
```

---

### 07 — Logs / Journald

**Enjeu :** Les logs sont la mémoire du système. En MCO, la capacité à trouver rapidement la cause d'un incident dans les logs est une compétence clé. Sur RHEL 9, deux systèmes coexistent : **journald** (systemd, binaire, structuré) et **rsyslog** (fichiers texte classiques). Les deux sont complémentaires.

```bash
# Consulter tous les logs système
journalctl

# Logs depuis le dernier boot uniquement
# Évite de remonter dans l'historique si le problème est récent
journalctl -b

# Logs d'un service spécifique
journalctl -u sshd

# Suivre les logs en temps réel (comme tail -f)
journalctl -u sshd -f

# Filtrer par niveau de priorité
# Niveaux : emerg, alert, crit, err, warning, notice, info, debug
journalctl -p err -b        # erreurs uniquement depuis le dernier boot
journalctl -p warning -b    # avertissements et plus grave

# Logs sur une plage de dates
journalctl --since "2026-03-01" --until "2026-03-11"
journalctl --since "1 hour ago"

# Logs du kernel (dmesg via journald)
journalctl -k

# Voir l'espace disque utilisé par les logs
journalctl --disk-usage

# Purger les anciens logs
# Par date : supprime les logs de plus de 7 jours
journalctl --vacuum-time=7d
# Par taille : ne garde que 200 Mo de logs
journalctl --vacuum-size=200M

# Limiter la taille des logs de façon permanente
# Éditer /etc/systemd/journald.conf
# SystemMaxUse=500M
systemctl restart systemd-journald

# Logs classiques via rsyslog
tail -f /var/log/messages        # logs système généraux
tail -f /var/log/secure          # authentifications, sudo, ssh
tail -f /var/log/audit/audit.log # logs SELinux et audit
```

---

### 08 — Cron / Planification

**Enjeu :** L'automatisation des tâches répétitives est indispensable en exploitation : sauvegardes, rotations de logs, rapports, nettoyages. Cron est l'outil historique, les **systemd timers** sont l'alternative moderne plus robuste sur RHEL 9 (logs intégrés à journald, gestion des dépendances).

```bash
# Éditer la crontab de l'utilisateur courant
crontab -e

# Lister les tâches planifiées de l'utilisateur courant
crontab -l

# Lister les tâches d'un autre utilisateur (en root)
crontab -l -u alice

# Supprimer toute la crontab
crontab -r

# Syntaxe cron
# ┌───── minute       (0-59)
# │ ┌───── heure        (0-23)
# │ │ ┌───── jour du mois (1-31)
# │ │ │ ┌───── mois        (1-12)
# │ │ │ │ ┌───── jour semaine (0-7, 0 et 7 = dimanche)
# │ │ │ │ │
# * * * * * commande

# Exemples commentés
0 2 * * *   /scripts/backup.sh       # tous les jours à 2h00
*/5 * * * * /scripts/check.sh        # toutes les 5 minutes
0 9 * * 1   /scripts/report.sh       # tous les lundis à 9h00
0 0 1 * *   /scripts/monthly.sh      # le 1er de chaque mois à minuit

# Rediriger la sortie pour éviter les mails cron
0 2 * * * /scripts/backup.sh >> /var/log/backup.log 2>&1

# Cron système — fichiers de config par dossier
ls /etc/cron.d/       # tâches système spécifiques
ls /etc/cron.daily/   # scripts exécutés chaque jour
ls /etc/cron.weekly/  # scripts exécutés chaque semaine
ls /etc/cron.monthly/ # scripts exécutés chaque mois

# Systemd timers — alternative moderne à cron
# Avantage : logs dans journald, rattrapage des tâches manquées, dépendances
systemctl list-timers        # lister tous les timers actifs et leur prochain déclenchement
systemctl list-timers --all  # inclure les timers inactifs
```

---

### 09 — Performances / Optimisation

**Enjeu :** Identifier et résoudre un problème de performances fait partie du quotidien d'un administrateur N2/N3. CPU saturé, mémoire insuffisante, I/O disque bloquant, réseau saturé — chaque symptôme a ses outils de diagnostic. L'objectif est de localiser le goulot d'étranglement rapidement avant que l'impact utilisateur ne s'aggrave.

```bash
# --- CPU ---

# Vue temps réel des processus avec consommation CPU/RAM
# Touche P = tri par CPU, M = tri par mémoire, k = kill
top
htop   # version améliorée (à installer : dnf install -y htop)

# Statistiques CPU par cœur (utile pour détecter un déséquilibre)
# 1 = intervalle en secondes, 5 = nombre de mesures
mpstat 1 5

# Charge système (load average sur 1, 5, 15 minutes)
# Si load > nombre de cœurs CPU → système en surcharge
uptime

# --- Mémoire ---

# Vue globale RAM et swap
# Si le swap est fortement utilisé → problème de mémoire
free -h

# Statistiques mémoire et swap détaillées
# si "si" et "so" (swap in/out) sont élevés → swapping actif = problème
vmstat 1 5

# --- Disque I/O ---

# Statistiques I/O par device
# %util proche de 100% = disque saturé
iostat -xz 1 5

# Voir quel processus consomme le plus d'I/O disque
iotop

# --- Réseau ---

# Lister les ports ouverts et les connexions actives
# Plus moderne que netstat, intégré à iproute2
ss -tulnp

# Statistiques des interfaces réseau
ip -s link

# Trafic réseau en temps réel par connexion (à installer)
iftop

# --- Processus ---

# Top 10 des processus par consommation CPU
ps aux --sort=-%cpu | head -10

# Top 10 des processus par consommation mémoire
ps aux --sort=-%mem | head -10

# Tuer un processus par PID
# -9 = SIGKILL = force, sans nettoyage — à utiliser en dernier recours
# Préférer -15 (SIGTERM) qui permet au processus de se terminer proprement
kill -15 <pid>
kill -9 <pid>

# Tuer tous les processus d'un nom
pkill <nom_process>

# --- Tuning système ---

# Lister les profils de performance disponibles
tuned-adm list

# Voir le profil actif
tuned-adm active

# Appliquer un profil adapté aux serveurs
# throughput-performance désactive les économies d'énergie pour maximiser les perfs
tuned-adm profile throughput-performance

# Swappiness : proportion d'utilisation du swap (0=RAM prioritaire, 100=swap agressif)
# En production serveur, une valeur basse (10) est recommandée
cat /proc/sys/vm/swappiness
sysctl -w vm.swappiness=10                    # temporaire
echo 'vm.swappiness=10' >> /etc/sysctl.conf   # permanent
sysctl -p                                      # recharger sysctl.conf

# Limites système (fichiers ouverts, processus...)
ulimit -a
```

---

## 🚨 Scénarios incidents

### INC-01 — Service en échec au démarrage

**Contexte :** Après un reboot ou une mise à jour, un service critique ne démarre plus. L'alerte remonte via la supervision ou un utilisateur signale une indisponibilité.

```bash
# 1. Identifier immédiatement les services en échec
systemctl list-units --state=failed

# 2. Lire le statut détaillé du service — les dernières lignes de log sont souvent suffisantes
systemctl status <service>

# 3. Remonter plus loin dans les logs si nécessaire
journalctl -u <service> -n 50

# 4. Tenter un redémarrage manuel
systemctl restart <service>

# 5. Si le redémarrage échoue — analyser l'erreur en détail
journalctl -xe

# 6. Corriger le fichier de configuration si nécessaire
# Puis recharger systemd et relancer
systemctl daemon-reload
systemctl restart <service>
```

---

### INC-02 — Disque plein

**Contexte :** Le monitoring remonte une alerte espace disque critique. Le serveur peut refuser d'écrire des logs, bloquer des applications ou devenir instable.

```bash
# 1. Identifier quel volume est plein
df -hT

# 2. Trouver les dossiers les plus lourds
du -sh /* 2>/dev/null | sort -rh | head -10

# 3. Identifier les gros fichiers
find / -size +500M -type f 2>/dev/null

# 4. Purger les logs anciens
journalctl --vacuum-time=7d
journalctl --vacuum-size=200M

# 5. Nettoyer le cache DNF
dnf clean all

# 6. Si insuffisant — étendre le LV à chaud
lvextend -L +5G /dev/vg_data/lv_data
xfs_growfs /data
```

---

### INC-03 — Utilisateur verrouillé / connexion impossible

**Contexte :** Un utilisateur ne peut plus se connecter suite à plusieurs tentatives échouées ou un verrouillage administratif.

```bash
# Vérifier le statut du compte
passwd -S <user>    # L = locked, P = password set
chage -l <user>     # expiration du mot de passe et du compte

# Déverrouiller le compte
usermod -U <user>
passwd -u <user>

# Réinitialiser le compteur d'échecs PAM
# PAM verrouille automatiquement après N tentatives échouées
faillock --user <user> --reset

# Vérifier les logs d'authentification
journalctl -u sshd | grep <user>
tail -50 /var/log/secure
```

---

### INC-04 — SELinux bloque un service

**Contexte :** Un service démarre en mode permissif mais échoue en enforcing. Symptôme typique : service en échec sans erreur évidente dans sa config.

```bash
# 1. Confirmer que SELinux est en cause
setenforce 0
systemctl restart <service>
# Si ça marche → c'est bien SELinux → repasser en enforcing
setenforce 1

# 2. Analyser les refus AVC
ausearch -m avc -ts recent

# 3. Obtenir des suggestions de correction automatiques
sealert -a /var/log/audit/audit.log

# 4. Corriger le contexte du fichier si déplacé manuellement
restorecon -Rv /chemin/du/fichier

# 5. Activer le booléen SELinux approprié si disponible
getsebool -a | grep <service>
setsebool -P <booleen> on

# 6. En dernier recours : générer un module de politique custom
# audit2allow lit les refus AVC et génère la politique correspondante
ausearch -m avc -ts recent | audit2allow -M mypolicy
semodule -i mypolicy.pp
```

---

## ✅ Checklists

### Prise en main d'un nouveau serveur RHEL 9

```
[ ] cat /etc/redhat-release                    → version OS exacte
[ ] hostnamectl                                → hostname / kernel / virtualisation
[ ] ip a                                       → interfaces et IPs configurées
[ ] df -hT                                     → espace disque disponible par volume
[ ] free -h                                    → RAM disponible
[ ] uptime                                     → charge système et durée depuis dernier boot
[ ] systemctl list-units --state=failed        → services en échec
[ ] dnf check-update                           → mises à jour disponibles
[ ] dnf check-update --security               → CVE en attente
[ ] subscription-manager status                → statut abonnement Red Hat
[ ] getenforce                                 → statut SELinux (doit être enforcing)
[ ] firewall-cmd --list-all                    → règles pare-feu actives
[ ] crontab -l && ls /etc/cron.d/             → tâches planifiées
[ ] last | head -10                            → dernières connexions
[ ] tail -20 /var/log/secure                   → tentatives d'auth récentes
```

### MCO quotidien

```
[ ] systemctl list-units --state=failed        → aucun service en échec
[ ] df -hT                                     → espace disque OK (< 80% utilisé)
[ ] journalctl -p err -b                       → pas d'erreurs critiques depuis le boot
[ ] dnf check-update --security               → pas de CVE critique non corrigée
[ ] tail /var/log/secure                       → pas de tentatives d'intrusion anormales
```

---

## 🎯 Questions d'entretien

### Niveau Technicien N1/N2

> **Quelle est la différence entre `systemctl stop` et `systemctl disable` ?**

`stop` arrête le service immédiatement mais il redémarrera au prochain boot. `disable` supprime le lien de démarrage automatique mais ne touche pas au service en cours. Les deux ensemble permettent d'arrêter définitivement un service.

> **Comment identifier un processus qui consomme trop de CPU ?**

`top` ou `ps aux --sort=-%cpu | head -10` pour identifier le PID et le nom du processus. Ensuite `systemctl status` si c'est un service, ou `journalctl` pour investiguer les logs.

> **Quelle commande permet de voir les ports réseau ouverts sur le système ?**

`ss -tulnp` : `-t` TCP, `-u` UDP, `-l` listening, `-n` sans résolution de nom, `-p` affiche le processus.

> **Qu'est-ce que SELinux et pourquoi est-il activé par défaut sur RHEL ?**

SELinux est un système de contrôle d'accès obligatoire (MAC). Il restreint les actions des processus au-delà des permissions UNIX classiques. Activé par défaut sur RHEL car requis pour les certifications gouvernementales (STIG, CIS) et recommandé pour tout environnement de production.

---

### Niveau Ingénieur N3

> **Comment étendre un volume LVM XFS à chaud sans interruption de service ?**

`lvextend -L +XG /dev/vg/lv` pour agrandir le LV, puis `xfs_growfs /point-de-montage` pour étendre le filesystem. XFS supporte l'extension à chaud sans démontage. Avec ext4 il faudrait `resize2fs` à la place.

> **Un service démarre en mode SELinux permissif mais échoue en enforcing. Démarche ?**

1. `ausearch -m avc -ts recent` pour lire les refus AVC.
2. `sealert -a /var/log/audit/audit.log` pour obtenir des suggestions.
3. Vérifier si un booléen existant couvre le besoin (`getsebool -a`).
4. `restorecon` si un fichier a perdu son contexte.
5. En dernier recours, `audit2allow` pour générer un module de politique custom.

> **Quelle différence entre `journalctl` et `/var/log/messages` ?**

`journalctl` interroge le journal binaire de systemd, structuré, avec filtrage par service, priorité, date ou boot. `/var/log/messages` est produit par rsyslog en texte brut. Sur RHEL 9 les deux coexistent, mais `journalctl` est plus précis et riche en métadonnées. En production, les deux sont complémentaires.

> **Comment identifier la cause d'un boot lent sur RHEL ?**

`systemd-analyze` donne le temps total de démarrage. `systemd-analyze blame` liste les services triés par temps de démarrage. `systemd-analyze critical-chain` montre la chaîne critique bloquante.

---

## Auteur

**Thomas PICHOT** — Administrateur Systèmes & Réseaux
[GitHub](https://github.com/4rch3n3my) | [LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45) | Toulouse, France
