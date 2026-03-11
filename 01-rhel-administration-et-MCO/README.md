# Module 01 — Administration RHEL & Maintien en Conditions Opérationnelles

> RHEL 9 — Administré sur Proxmox VE 8
> Axes couverts : packages, services, utilisateurs, LVM, sécurité, logs, planification, performances

---

## Environnement

| Composant | Détail |
|---|---|
| Hyperviseur | Proxmox VE 8.x |
| OS | RHEL 9 |
| Hostname | rhel-lab |
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

```bash
# Vérifier la version RHEL
[root@localhost ~]# cat /etc/redhat-release 
Red Hat Enterprise Linux release 9.7 (Plow)

# Enregistrer le compte Developpeur RHEL et Mettre à jour le système
[root@localhost ~]# subscription-manager register --username ******* --password '********************'
Inscription sur : subscription.rhsm.redhat.com:443/subscription
Le système a été enregistré avec l'ID : *****-******-**********-*****
Le nom de système enregistré est : localhost.localdomain

[root@localhost ~]# dnf update -y
Mise à jour des référentiels de gestion des abonnements.
En attente de la fin d’exécution du processus ayant l’identifiant (pid) 3102.
Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)                              22 MB/s |  84 MB     00:03    
Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)                                 30 MB/s | 105 MB     00:03    
Dernière vérification de l’expiration des métadonnées effectuée il y a 0:00:01 le mer. 11 mars 2026 13:27:14.
Dépendances résolues.
=====================================================================================================================
 Paquet                                  Architecture
                                                Version                       Dépôt                            Taille
=====================================================================================================================
Installation:
 kernel                                  x86_64 5.14.0-611.38.1.el9_7         rhel-9-for-x86_64-baseos-rpms    1.1 M
Mise à jour:
 NetworkManager                          x86_64 1:1.54.0-3.el9_7              rhel-9-for-x86_64-baseos-rpms    2.4 M
 NetworkManager-adsl                     x86_64 1:1.54.0-3.el9_7              rhel-9-for-x86_64-baseos-rpms     32 k
 NetworkManager-bluetooth                x86_64 1:1.54.0-3.el9_7              rhel-9-for-x86_64-baseos-rpms     58 k
 NetworkManager-config-server            noarch 1:1.54.0-3.el9_7              rhel-9-for-x86_64-baseos-rpms     18 k
 NetworkManager-libnm                    x86_64 1:1.54.0-3.el9_7              rhel-9-for-x86_64-baseos-rpms    1.9 M
.
.
.
.
  xorg-x11-server-common-1.20.11-32.el9_7.x86_64                                                                     
Installé:
  kernel-5.14.0-611.38.1.el9_7.x86_64                    kernel-core-5.14.0-611.38.1.el9_7.x86_64                   
  kernel-modules-5.14.0-611.38.1.el9_7.x86_64            kernel-modules-core-5.14.0-611.38.1.el9_7.x86_64           

Terminé !


# Configurer le hostname
[root@localhost ~]# hostnamectl set-hostname rhel9-lab
[root@localhost ~]# hostname
rhel9-lab
[root@localhost ~]# hostnamectl status
 Static hostname: rhel9-lab
       Icon name: computer-vm
         Chassis: vm 🖴
      Machine ID: 1a176aa52269400f95e02be2db21b169
         Boot ID: 8772e5624987451b8b22a9fc93d95be2
  Virtualization: kvm
Operating System: Red Hat Enterprise Linux 9.7 (Plow)         
     CPE OS Name: cpe:/o:redhat:enterprise_linux:9::baseos
          Kernel: Linux 5.14.0-611.5.1.el9_7.x86_64
    Architecture: x86-64
 Hardware Vendor: QEMU
  Hardware Model: Standard PC _i440FX + PIIX, 1996_
Firmware Version: rel-1.16.3-0-ga6ed6b701f0a-prebuilt.qemu.org

# Configurer le fuseau horaire
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

```bash
# Rechercher un package
dnf search <package>

# Installer un package
dnf install -y <package>

# Supprimer un package
dnf remove -y <package>

# Lister les packages installés
dnf list installed

# Voir l'historique des opérations DNF
dnf history

# Annuler la dernière opération DNF
dnf history undo last

# Vérifier les mises à jour disponibles
dnf check-update

# Installer les mises à jour de sécurité uniquement
dnf update --security -y

# Gérer les dépôts
dnf repolist
dnf config-manager --enable x
dnf config-manager --disable x
```

---

### 03 — Gestion des services systemd

```bash
# Lister tous les services ( on peut filtrer par les services failed aussi ) 
[root@localhost ~]# systemctl list-units --type=service | grep running
  accounts-daemon.service            loaded active running Accounts Service
  atd.service                        loaded active running Deferred execution scheduler
  auditd.service                     loaded active running Security Auditing Service
  avahi-daemon.service               loaded active running Avahi mDNS/DNS-SD Stack
  chronyd.service                    loaded active running NTP client/server
  colord.service                     loaded active running Manage, Install and Generate Color Profiles
  crond.service                      loaded active running Command Scheduler
  cups.service                       loaded active running CUPS Scheduler
  dbus-broker.service                loaded active running D-Bus System Message Bus
  firewalld.service                  loaded active running firewalld - dynamic firewall daemon
  fwupd.service                      loaded active running Firmware update daemon
  gdm.service                        loaded active running GNOME Display Manager
  irqbalance.service                 loaded active running irqbalance daemon
  libstoragemgmt.service             loaded active running libstoragemgmt plug-in server daemon
  mcelog.service                     loaded active running Machine Check Exception Logging Daemon
  ModemManager.service               loaded active running Modem Manager
  NetworkManager.service             loaded active running Network Manager
  packagekit.service                 loaded active running PackageKit Daemon
  polkit.service                     loaded active running Authorization Manager
  rhsm.service                       loaded active running RHSM dbus service
  rhsmcertd.service                  loaded active running Enable periodic update of entitlement certificates.
  rsyslog.service                    loaded active running System Logging Service
  rtkit-daemon.service               loaded active running RealtimeKit Scheduling Policy Service
  sshd.service                       loaded active running OpenSSH server daemon
  sssd-kcm.service                   loaded active running SSSD Kerberos Cache Manager
  switcheroo-control.service         loaded active running Switcheroo Control Proxy service
  systemd-journald.service           loaded active running Journal Service
  systemd-logind.service             loaded active running User Login Management
  systemd-udevd.service              loaded active running Rule-based Manager for Device Events and Files
  tuned-ppd.service                  loaded active running PPD-to-TuneD API Translation Daemon
  tuned.service                      loaded active running Dynamic System Tuning Daemon
  udisks2.service                    loaded active running Disk Manager
  upower.service                     loaded active running Daemon for power management
  user@1000.service                  loaded active running User Manager for UID 1000
  wpa_supplicant.service             loaded active running WPA supplicant

# Lister les services en échec
systemctl list-units --state=failed

# Démarrer / arrêter / redémarrer un service
systemctl start <service>
systemctl stop <service>
systemctl restart <service>

# Activer / désactiver au démarrage
systemctl enable <service>
systemctl disable <service>

# Vérifier le statut d'un service
systemctl status <service>

# Recharger la config sans redémarrer
systemctl reload <service>

# Voir les dépendances d'un service
systemctl list-dependencies <service>

# Analyser le temps de démarrage
[root@localhost ~]# systemd-analyze 
Startup finished in 1.448s (kernel) + 4.705s (initrd) + 1min 3.202s (userspace) = 1min 9.356s 
graphical.target reached after 1min 3.177s in userspace.

systemd-analyze blame
```

---

### 04 — Gestion des utilisateurs / permissions

```bash
# Créer un utilisateur
[root@localhost ~]# useradd -m -s /bin/bash user1
[root@localhost ~]# passwd user1
Changement de mot de passe pour l'utilisateur user1.
Nouveau mot de passe : 
Retapez le nouveau mot de passe : 
passwd : mise à jour réussie de tous les jetons d'authentification.

# Créer un groupe
[root@localhost ~]# groupadd testgroup

# Ajouter un utilisateur à un groupe
[root@localhost ~]# usermod -aG testgroup user1 

# Vérifier les groupes d'un utilisateur
[root@localhost ~]# id user1
uid=1001(user1) gid=1001(user1) groupes=1001(user1),1002(testgroup)
[root@localhost ~]# groups user1
user1 : user1 testgroup

# Modifier un utilisateur
usermod -l <newname> <oldname>   # renommer
usermod -L <user>                 # verrouiller
usermod -U <user>                 # déverrouiller

# Supprimer un utilisateur
userdel -r <user>   # -r supprime aussi le home

# Permissions fichiers
chmod 750 /dossier
chown user:group /dossier

# ACL étendues
setfacl -m u:<user>:rwx /dossier
getfacl /dossier

# Sudo — ajouter un utilisateur
usermod -aG wheel <user>
# Ou via visudo pour règle fine :
# <user> ALL=(ALL) NOPASSWD: /usr/bin/systemctl
```

---

### 05 — LVM / Stockage

```bash
# Lister les disques
lsblk
fdisk -l

# Créer un Physical Volume
pvcreate /dev/sdb
pvdisplay

# Créer un Volume Group
vgcreate vg_data /dev/sdb
vgdisplay

# Créer un Logical Volume
lvcreate -L 5G -n lv_data vg_data
lvdisplay

# Formater et monter
mkfs.xfs /dev/vg_data/lv_data
mkdir /data
mount /dev/vg_data/lv_data /data

# Montage permanent
echo '/dev/vg_data/lv_data /data xfs defaults 0 0' >> /etc/fstab
mount -a

# Étendre un LV à chaud
lvextend -L +2G /dev/vg_data/lv_data
xfs_growfs /data   # pour XFS
# resize2fs /dev/vg_data/lv_data   # pour ext4

# Vérifier l'espace disque
df -hT
du -sh /data/*
```

---

### 06 — Firewalld / SELinux

```bash
# --- Firewalld ---

# Statut
systemctl status firewalld
firewall-cmd --state

# Lister les règles actives
firewall-cmd --list-all

# Ajouter une règle (permanente)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-port=8080/tcp

# Supprimer une règle
firewall-cmd --permanent --remove-service=http

# Recharger après modification
firewall-cmd --reload

# Lister les zones
firewall-cmd --get-zones
firewall-cmd --get-active-zones

# --- SELinux ---

# Vérifier le statut SELinux
getenforce
sestatus

# Passer en mode permissif (logs sans bloquer)
setenforce 0

# Repasser en enforcing
setenforce 1

# Voir les alertes SELinux
ausearch -m avc -ts recent
sealert -a /var/log/audit/audit.log

# Corriger un contexte de fichier
restorecon -Rv /var/www/html

# Modifier un booléen SELinux
getsebool -a | grep http
setsebool -P httpd_can_network_connect on
```

---

### 07 — Logs / Journald

```bash
# Consulter les logs système
journalctl

# Logs depuis le dernier boot
journalctl -b

# Logs d'un service
journalctl -u sshd
journalctl -u sshd -f   # suivi en temps réel

# Logs par priorité (err, warning, info...)
journalctl -p err -b

# Logs entre deux dates
journalctl --since "2026-03-01" --until "2026-03-10"

# Logs kernel
journalctl -k

# Espace occupé par les logs
journalctl --disk-usage

# Limiter la taille des logs (persistant)
# Éditer /etc/systemd/journald.conf
# SystemMaxUse=500M
systemctl restart systemd-journald

# Logs classiques
tail -f /var/log/messages
tail -f /var/log/secure      # authentifications
tail -f /var/log/audit/audit.log
```

---

### 08 — Cron / Planification

```bash
# Éditer la crontab utilisateur
crontab -e

# Lister la crontab
crontab -l

# Supprimer la crontab
crontab -r

# Syntaxe cron
# ┌───── minute (0-59)
# │ ┌───── heure (0-23)
# │ │ ┌───── jour du mois (1-31)
# │ │ │ ┌───── mois (1-12)
# │ │ │ │ ┌───── jour de la semaine (0-7, 0=dimanche)
# │ │ │ │ │
# * * * * * commande

# Exemples
0 2 * * * /scripts/backup.sh          # tous les jours à 2h
*/5 * * * * /scripts/check.sh         # toutes les 5 minutes
0 9 * * 1 /scripts/report.sh          # tous les lundis à 9h

# Cron système
ls /etc/cron.d/
ls /etc/cron.daily/
ls /etc/cron.weekly/

# Systemd timers (alternative moderne à cron)
systemctl list-timers
```

---

### 09 — Performances / Optimisation

```bash
# CPU
top
htop
mpstat 1 5          # stats CPU toutes les secondes
uptime              # charge système

# Mémoire
free -h
vmstat 1 5

# Disque I/O
iostat -xz 1 5
iotop

# Réseau
ss -tulnp           # ports ouverts
netstat -tulnp
iftop               # trafic en temps réel
ip -s link          # stats interfaces

# Processus
ps aux --sort=-%cpu | head -10    # top CPU
ps aux --sort=-%mem | head -10    # top RAM
kill -9 <pid>
pkill <nom_process>

# Tuning système
tuned-adm list
tuned-adm active
tuned-adm profile throughput-performance   # profil serveur

# Limites système
ulimit -a
cat /proc/sys/vm/swappiness
sysctl -w vm.swappiness=10
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p
```

---

## 🚨 Scénarios incidents

### INC-01 — Service en échec au démarrage

**Contexte :** Un service critique ne démarre plus après une mise à jour.

```bash
# 1. Identifier le service en échec
systemctl list-units --state=failed

# 2. Lire les logs du service
systemctl status <service>
journalctl -u <service> -n 50

# 3. Tenter un redémarrage manuel
systemctl restart <service>

# 4. Si échec — vérifier la config
journalctl -xe
# Corriger le fichier de config puis :
systemctl daemon-reload
systemctl restart <service>
```

---

### INC-02 — Disque plein

**Contexte :** Alerte espace disque, le serveur ne répond plus normalement.

```bash
# 1. Identifier le volume plein
df -hT

# 2. Trouver les gros fichiers
du -sh /* 2>/dev/null | sort -rh | head -10
find / -size +500M -type f 2>/dev/null

# 3. Purger les logs anciens
journalctl --vacuum-time=7d
journalctl --vacuum-size=200M

# 4. Nettoyer le cache DNF
dnf clean all

# 5. Étendre le LV si possible
lvextend -L +5G /dev/vg_data/lv_data
xfs_growfs /data
```

---

### INC-03 — Utilisateur verrouillé / connexion impossible

```bash
# Vérifier le statut du compte
passwd -S <user>
chage -l <user>

# Déverrouiller le compte
usermod -U <user>
passwd -u <user>

# Réinitialiser le compteur d'échecs PAM
faillock --user <user> --reset

# Vérifier les logs d'auth
journalctl -u sshd | grep <user>
tail -f /var/log/secure
```

---

### INC-04 — SELinux bloque un service

```bash
# 1. Passer temporairement en permissif pour confirmer
setenforce 0
systemctl restart <service>
# Si ça marche → c'est bien SELinux

# 2. Analyser les alertes
ausearch -m avc -ts recent
sealert -a /var/log/audit/audit.log

# 3. Corriger le contexte
restorecon -Rv /chemin/du/fichier

# 4. Ou activer le booléen approprié
getsebool -a | grep <service>
setsebool -P <booleen> on

# 5. Repasser en enforcing
setenforce 1
```

---

## ✅ Checklists

### Prise en main d'un nouveau serveur RHEL 9

```
[ ] cat /etc/redhat-release                    → version OS
[ ] hostnamectl                                → hostname / OS / kernel
[ ] ip a                                       → interfaces et IPs
[ ] df -hT                                     → espace disque
[ ] free -h                                    → RAM disponible
[ ] uptime                                     → charge et uptime
[ ] systemctl list-units --state=failed        → services en échec
[ ] dnf check-update                           → mises à jour disponibles
[ ] subscription-manager status                → statut abonnement RH
[ ] getenforce                                 → statut SELinux
[ ] firewall-cmd --list-all                    → règles pare-feu
[ ] crontab -l && ls /etc/cron.d/             → tâches planifiées
[ ] last | head -10                            → dernières connexions
[ ] cat /var/log/secure | tail -20             → logs auth récents
```

### MCO quotidien

```
[ ] systemctl list-units --state=failed        → aucun service en échec
[ ] df -hT                                     → espace disque OK
[ ] journalctl -p err -b                       → pas d'erreurs critiques
[ ] dnf check-update --security               → pas de CVE critique en attente
[ ] tail /var/log/secure                       → pas de tentatives d'intrusion
```

---

## 🎯 Questions d'entretien

**Niveau Technicien N1/N2**

> Quelle est la différence entre `systemctl stop` et `systemctl disable` ?

`stop` arrête le service immédiatement. `disable` empêche son démarrage automatique au prochain boot. Les deux ensemble = service arrêté et ne redémarre pas.

> Comment identifier un processus qui consomme trop de CPU ?

`top` ou `ps aux --sort=-%cpu | head -10` pour identifier le PID, puis `kill` ou investigation des logs du processus.

> Qu'est-ce que SELinux et pourquoi est-il activé par défaut sur RHEL ?

SELinux est un système de contrôle d'accès obligatoire (MAC). Il restreint les actions des processus au-delà des permissions UNIX classiques. Activé par défaut sur RHEL car requis pour les certifications de sécurité gouvernementales et recommandé par le CIS.

---

**Niveau Ingénieur N3**

> Comment étendre un LV XFS à chaud sans interruption de service ?

`lvextend -L +XG /dev/vg/lv` puis `xfs_growfs /point-de-montage`. XFS supporte l'extension à chaud sans démontage, contrairement à ext4 qui nécessite `resize2fs`.

> Un service démarre en mode SELinux permissif mais échoue en enforcing. Quelle est ta démarche ?

1. `ausearch -m avc -ts recent` pour lire les refus AVC.
2. `sealert` pour obtenir les suggestions de correction.
3. Vérifier si un booléen existant couvre le besoin (`getsebool -a`).
4. Corriger le contexte fichier avec `restorecon` si nécessaire.
5. En dernier recours, créer un module de politique SELinux custom avec `audit2allow`.

> Quelle différence entre `journalctl` et `/var/log/messages` ?

`journalctl` interroge le journal binaire de systemd, structuré et filtrable. `/var/log/messages` est le fichier texte produit par rsyslog. Sur RHEL 9, les deux coexistent mais `journalctl` est plus riche en métadonnées et en capacités de filtrage.

---

## Auteur

**Thomas PICHOT** — Administrateur Systèmes & Réseaux  
[GitHub](https://github.com/4rch3n3my) | [LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45) | Toulouse, France
