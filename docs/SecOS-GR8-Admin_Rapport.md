login : sysadmin, mdp : admin
# Rapport de Durcissement - Bastion SSH/Admin

**Date** : 2026-01-01
**Système** : Debian GNU/Linux 12 (Bookworm)
**Rôle** : Bastion SSH pour infrastructure PME
**Hostname** : secos-gr8-admin

---

## Table des Matières

1. [Partitions et Options de Montage](#1-partitions-et-options-de-montage)
2. [Durcissement SSH](#2-durcissement-ssh)
3. [Pare-feu UFW](#3-pare-feu-ufw)
4. [Fail2ban](#4-fail2ban)
5. [AppArmor](#5-apparmor)
6. [Services Système](#6-services-système)
7. [Configuration Clés SSH](#7-configuration-clés-ssh)
8. [Vérifications Finales](#8-vérifications-finales)
9. [Fichiers Modifiés](#9-fichiers-modifiés)

---

## 1. Partitions et Options de Montage

### 1.1 État Initial

Le système dispose de partitions LVM séparées :

| Partition | Taille | Options initiales |
|-----------|--------|-------------------|
| `/` | 3.2G | errors=remount-ro |
| `/boot` | 455M | defaults |
| `/home` | 9.4G | defaults |
| `/tmp` | 294M | defaults |
| `/var` | 1.4G | defaults |
| `swap` | - | sw |

### 1.2 Modifications Appliquées

Fichier modifié : `/etc/fstab`

```
/dev/mapper/secos--gr8--admin--vg-root /               ext4    errors=remount-ro 0       1
UUID=0ea47aca-e731-442b-9306-ddda2e83019a /boot           ext2    defaults        0       2
/dev/mapper/secos--gr8--admin--vg-home /home           ext4    defaults,nodev,nosuid,noexec        0       2
/dev/mapper/secos--gr8--admin--vg-tmp /tmp            ext4    defaults,nodev,nosuid,noexec        0       2
/dev/mapper/secos--gr8--admin--vg-var /var            ext4    defaults,nodev,nosuid        0       2
/dev/mapper/secos--gr8--admin--vg-swap_1 none            swap    sw              0       0
```

### 1.3 Options de Sécurité Ajoutées

| Partition | Options ajoutées | Justification |
|-----------|------------------|---------------|
| `/home` | nodev, nosuid, noexec | Empêche l'exécution de binaires, fichiers spéciaux et setuid |
| `/tmp` | nodev, nosuid, noexec | Zone temporaire - aucune exécution nécessaire |
| `/var` | nodev, nosuid | Pas de noexec car scripts système requis (cron, etc.) |

### 1.4 Vérification

```bash
findmnt / /home /tmp /var /boot
mount -a  # Doit sortir sans erreur
```

---

## 2. Durcissement SSH

### 2.1 Fichier de Configuration Créé

Fichier : `/etc/ssh/sshd_config.d/hardening.conf`

```bash
# SSH Hardening Configuration for Bastion Server
# Généré pour durcissement PME

# Désactiver connexion root
PermitRootLogin no

# Authentification par clé uniquement
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
KbdInteractiveAuthentication no

# Limiter les tentatives d'authentification
MaxAuthTries 3
MaxSessions 3
LoginGraceTime 30

# Désactiver les fonctionnalités non nécessaires
X11Forwarding no
AllowAgentForwarding no
# AllowTcpForwarding conservé pour fonction bastion
AllowTcpForwarding yes

# Timeout des sessions inactives (5 min)
ClientAliveInterval 300
ClientAliveCountMax 0

# Logging verbeux pour audit
LogLevel VERBOSE

# Bannière d'avertissement légal
Banner /etc/issue.net

# Désactiver les méthodes d'authentification non sécurisées
HostbasedAuthentication no
IgnoreRhosts yes

# Algorithmes cryptographiques robustes
Ciphers aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
```

### 2.2 Bannière d'Avertissement

Fichier : `/etc/issue.net`

```
***************************************************************************
                         AVERTISSEMENT / WARNING
***************************************************************************
Ce systeme est reserve aux utilisateurs autorises.
Toute tentative d'acces non autorisee est interdite et sera poursuivie.
Les activites sur ce systeme sont surveillees et enregistrees.

This system is restricted to authorized users only.
Unauthorized access attempts are prohibited and will be prosecuted.
All activities on this system are monitored and logged.
***************************************************************************
```

### 2.3 Paramètres Effectifs (sshd -T)

| Paramètre | Valeur |
|-----------|--------|
| port | 22 |
| permitrootlogin | no |
| passwordauthentication | no |
| kbdinteractiveauthentication | no |
| pubkeyauthentication | yes |
| maxauthtries | 3 |
| clientaliveinterval | 300 |
| clientalivecountmax | 0 |

### 2.4 Vérification

```bash
sshd -t                    # Doit sortir sans erreur
systemctl restart sshd
sshd -T | grep -E "permitrootlogin|passwordauthentication|pubkeyauthentication"
```

---

## 3. Pare-feu UFW

### 3.1 Installation

```bash
apt-get install -y ufw
```

### 3.2 Configuration

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw --force enable
```

### 3.3 Règles Actives

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

### 3.4 Vérification

```bash
ufw status verbose
ss -lntp | grep ssh
```

---

## 4. Fail2ban

### 4.1 Installation

```bash
apt-get install -y fail2ban
```

### 4.2 Configuration

Fichier : `/etc/fail2ban/jail.local`

```ini
# Configuration Fail2ban durcie pour Bastion SSH
# /etc/fail2ban/jail.local

[DEFAULT]
# Ignorer localhost
ignoreip = 127.0.0.1/8 ::1

# Durée de bannissement : 1 heure
bantime = 1h

# Fenêtre de détection : 10 minutes
findtime = 10m

# Nombre d'échecs avant bannissement
maxretry = 3

# Action par défaut : bannir via UFW
banaction = ufw

# Activation du bannissement incrémental
bantime.increment = true
bantime.factor = 2
bantime.maxtime = 1w

[sshd]
enabled = true
port = ssh
filter = sshd
backend = systemd
maxretry = 3
findtime = 10m
bantime = 1h

# Protection contre les attaques de récidive
# Activé après que fail2ban génère des logs
[recidive]
enabled = false
logpath = /var/log/fail2ban.log
banaction = ufw
bantime = 1w
findtime = 1d
maxretry = 5
```

### 4.3 Paramètres de Protection

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| maxretry | 3 | Tentatives avant ban |
| findtime | 10m | Fenêtre de détection |
| bantime | 1h | Durée initiale du ban |
| bantime.increment | true | Ban progressif |
| bantime.maxtime | 1w | Ban max 1 semaine |
| backend | systemd | Lecture journal systemd |

### 4.4 Vérification

```bash
systemctl status fail2ban --no-pager
fail2ban-client status
fail2ban-client status sshd
```

---

## 5. AppArmor

### 5.1 État

AppArmor était déjà installé et actif sur le système.

### 5.2 Profils Chargés

```
apparmor module is loaded.
10 profiles are loaded.
10 profiles are in enforce mode.
   /usr/bin/man
   /usr/lib/NetworkManager/nm-dhcp-client.action
   /usr/lib/NetworkManager/nm-dhcp-helper
   /usr/lib/connman/scripts/dhclient-script
   /{,usr/}sbin/dhclient
   lsb_release
   man_filter
   man_groff
   nvidia_modprobe
   nvidia_modprobe//kmod
```

### 5.3 Vérification

```bash
aa-status
```

---

## 6. Services Système

### 6.1 Services Actifs au Démarrage

```
apparmor.service          enabled
blk-availability.service  enabled
console-setup.service     enabled
cron.service              enabled
e2scrub_reap.service      enabled
fail2ban.service          enabled
getty@.service            enabled
keyboard-setup.service    enabled
lvm2-monitor.service      enabled
networking.service        enabled
ssh.service               enabled
systemd-pstore.service    enabled
systemd-timesyncd.service enabled
ufw.service               enabled
```

### 6.2 Analyse

L'installation est minimale. Aucun service superflu détecté (pas de avahi, cups, bluetooth, serveur web, base de données, etc.).

---

## 7. Configuration Clés SSH

### 7.1 Utilisateur Configuré

| Élément | Valeur |
|---------|--------|
| Utilisateur | sysadmin |
| UID | 1000 |
| Home | /home/sysadmin |

### 7.2 Clé Publique Autorisée

Fichier : `/home/sysadmin/.ssh/authorized_keys`

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDVCDGYxt15BBmIjmpX+VW1zYMckt1JwxxitQ1KyhMVz riad.hyseni@hes-so.ch
```

### 7.3 Permissions

```
drwx------ 2 sysadmin sysadmin 4096 .ssh/
-rw------- 1 sysadmin sysadmin  103 .ssh/authorized_keys
```

### 7.4 Connexion

```bash
ssh sysadmin@<IP_BASTION>
```

---

## 8. Vérifications Finales

### 8.1 Checklist de Validation

| Test | Commande | Résultat Attendu |
|------|----------|------------------|
| fstab valide | `mount -a` | Aucune erreur |
| Points de montage | `findmnt /home /tmp /var` | Options nodev,nosuid,noexec visibles |
| Config SSH | `sshd -t` | Aucune erreur |
| SSH effectif | `sshd -T \| grep passwordauthentication` | `no` |
| UFW actif | `ufw status` | `Status: active` |
| Port SSH | `ss -lntp \| grep ssh` | Port 22 en écoute |
| Fail2ban | `fail2ban-client status sshd` | Jail active |
| AppArmor | `aa-status` | Module loaded, profils enforce |

### 8.2 Résultats

Tous les tests ont été validés avec succès.

---

## 9. Fichiers Modifiés

| Fichier | Action | Description |
|---------|--------|-------------|
| `/etc/fstab` | Modifié | Ajout options nodev,nosuid,noexec |
| `/etc/ssh/sshd_config.d/hardening.conf` | Créé | Configuration SSH durcie |
| `/etc/issue.net` | Modifié | Bannière d'avertissement légal |
| `/etc/fail2ban/jail.local` | Créé | Configuration fail2ban |
| `/home/sysadmin/.ssh/authorized_keys` | Créé | Clé publique autorisée |

---

## Recommandations Supplémentaires

### Optionnel - À considérer

1. **Restreindre les utilisateurs SSH** :
   ```bash
   echo "AllowUsers sysadmin" >> /etc/ssh/sshd_config.d/hardening.conf
   ```

2. **Changer le port SSH** (sécurité par obscurité) :
   ```bash
   # Dans hardening.conf
   Port 2222
   # Puis adapter UFW
   ufw delete allow ssh
   ufw allow 2222/tcp
   ```

3. **Activer la jail recidive** une fois fail2ban opérationnel :
   ```bash
   # Dans /etc/fail2ban/jail.local, changer enabled = true pour [recidive]
   ```

4. **Audit régulier** :
   ```bash
   # Vérifier les bans fail2ban
   fail2ban-client status sshd

   # Vérifier les logs SSH
   journalctl -u sshd --since "1 hour ago"
   ```

---

## Auteur

Rapport généré automatiquement lors du durcissement du bastion.

**Bastion prêt pour la production.**

