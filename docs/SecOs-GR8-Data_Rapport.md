# Configuration SecOS-GR8-Data - Documentation Complète

**Date de configuration** : 2 janvier 2026
**Configuré par** : Claude Code (Opus 4.5)

---

## 1. Informations Générales de la VM

| Élément | Valeur |
|---------|--------|
| **Nom de la VM** | SecOS-GR8-Data |
| **Rôle** | Serveur de fichiers (Samba + SFTP) |
| **OS** | Debian 12 (Bookworm) |
| **Hostname** | `secos-gr8-data` |
| **IP** | `192.168.100.149/24` |
| **Passerelle** | `192.168.100.255` (broadcast) |
| **Interface réseau** | `ens18` |

---

## 2. Comptes Utilisateurs

### 2.1 Compte sysadmin (Administration)

| Élément | Valeur |
|---------|--------|
| **Username** | `sysadmin` |
| **UID** | `1000` |
| **GID** | `1000` |
| **Shell** | `/bin/bash` |
| **Home** | `/home/sysadmin` |
| **Groupes** | sysadmin, cdrom, floppy, sudo, audio, dip, video, plugdev, users, netdev |
| **Authentification** | Clé SSH uniquement (pas de mot de passe SSH) |

**Clés SSH publiques** (dans `/home/sysadmin/.ssh/authorized_keys`) :

| Clé | Propriétaire | Usage |
|-----|--------------|-------|
| `ssh-ed25519 AAAAC3...VCDGYxt15BBmIjmpX+VW1zYMckt1JwxxitQ1KyhMVz` | sysadmin@secos-gr8 | Ancienne clé (instructions originales) |
| `ssh-ed25519 AAAAC3...GlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3` | root@secos-gr8-admin | **Clé active** (VM Admin) |

**Contenu complet du fichier** :
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDVCDGYxt15BBmIjmpX+VW1zYMckt1JwxxitQ1KyhMVz sysadmin@secos-gr8
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
```

> **Note** : La deuxième clé (`root@secos-gr8-admin`) est celle créée sur la VM Admin. La clé privée correspondante se trouve dans `/root/.ssh/id_ed25519` sur la VM Admin (192.168.100.120).

### 2.2 Compte alice (Utilisateur test SFTP + Samba)

| Élément | Valeur |
|---------|--------|
| **Username** | `alice` |
| **UID** | `1001` |
| **GID** | `1002` |
| **Shell** | `/usr/sbin/nologin` |
| **Home** | `/home/alice` |
| **Groupes** | alice, sftpusers (GID 1001), smbusers (GID 1003) |
| **Mot de passe Linux** | `alice123` |
| **Mot de passe Samba** | `alice123` |

---

## 3. Groupes Créés

| Groupe | GID | Membres | Usage |
|--------|-----|---------|-------|
| `sftpusers` | 1001 | alice | Accès SFTP avec chroot |
| `smbusers` | 1003 | alice | Accès aux partages Samba |

---

## 4. Services Installés et Configurés

### 4.1 Paquets Installés

```bash
apt -y install vim curl ca-certificates sudo
apt -y install samba smbclient openssh-server ufw apparmor apparmor-utils
```

### 4.2 État des Services

| Service | État | Port(s) |
|---------|------|---------|
| `ssh` | active (enabled) | 22/tcp |
| `smbd` | active (enabled) | 139/tcp, 445/tcp |
| `nmbd` | active (enabled) | 137/udp, 138/udp |
| `apparmor` | active (enabled) | - |
| `ufw` | active (enabled) | - |

---

## 5. Configuration SSH - Durcissement

### 5.1 Fichier de configuration

**Fichier** : `/etc/ssh/sshd_config.d/hardening.conf`

```bash
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes

X11Forwarding no
AllowAgentForwarding no

# SFTP chroot pour le groupe sftpusers (avec password autorisé)
Match Group sftpusers
    PasswordAuthentication yes
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

### 5.2 Résumé des paramètres SSH

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `PermitRootLogin` | no | Connexion root interdite |
| `PasswordAuthentication` | no | Auth par mot de passe désactivée (sauf SFTP) |
| `PubkeyAuthentication` | yes | Auth par clé SSH activée |
| `X11Forwarding` | no | Forwarding X11 désactivé |
| `AllowAgentForwarding` | no | Agent forwarding désactivé |

### 5.3 Configuration SFTP Chroot (groupe sftpusers)

| Paramètre | Valeur |
|-----------|--------|
| `PasswordAuthentication` | yes |
| `ChrootDirectory` | /sftp/%u |
| `ForceCommand` | internal-sftp |
| `AllowTcpForwarding` | no |

---

## 6. Configuration SFTP Chroot

### 6.1 Structure des Dossiers

```
/sftp/
└── alice/                 # root:root 755 (chroot directory)
    └── upload/            # alice:sftpusers 750 (dossier d'écriture)
```

### 6.2 Commandes de Configuration

```bash
# Création du groupe
groupadd sftpusers

# Création du dossier racine SFTP
mkdir -p /sftp
chown root:root /sftp
chmod 755 /sftp

# Création utilisateur alice
useradd -m -s /usr/sbin/nologin -G sftpusers alice
echo "alice:alice123" | chpasswd

# Création dossier chroot alice
mkdir -p /sftp/alice/upload
chown root:root /sftp/alice
chmod 755 /sftp/alice
chown alice:sftpusers /sftp/alice/upload
chmod 750 /sftp/alice/upload
```

### 6.3 Permissions Finales

```
drwxr-xr-x 3 root  root      4096 /sftp
drwxr-xr-x 3 root  root      4096 /sftp/alice
drwxr-x--- 2 alice sftpusers 4096 /sftp/alice/upload
```

---

## 7. Configuration Samba

### 7.1 Fichier de Configuration

**Fichier** : `/etc/samba/smb.conf`

```ini
[global]
   workgroup = WORKGROUP
   server role = standalone server
   map to guest = never
   disable netbios = no

   # Sécurité
   server min protocol = SMB2
   client min protocol = SMB2
   smb ports = 445 139
   restrict anonymous = 2

   # Limiter au LAN
   hosts allow = 127. 192.168.100.
   hosts deny = 0.0.0.0/0

   # Logs
   log file = /var/log/samba/log.%m
   logging = file
   log level = 1

   # Désactiver imprimantes
   load printers = no
   printing = bsd
   printcap name = /dev/null
   disable spoolss = yes

[company]
   path = /srv/samba/company
   browseable = yes
   read only = no
   valid users = @smbusers
   force group = smbusers
   create mask = 0660
   directory mask = 2770
```

### 7.2 Partage "company"

| Paramètre | Valeur |
|-----------|--------|
| **Chemin** | `/srv/samba/company` |
| **Accès** | Groupe `smbusers` uniquement |
| **Permissions fichiers** | 0660 |
| **Permissions dossiers** | 2770 (SGID) |
| **Propriétaire** | root:smbusers |

### 7.3 Commandes de Configuration

```bash
# Création dossier de partage
mkdir -p /srv/samba/company
groupadd smbusers
chown root:smbusers /srv/samba/company
chmod 2770 /srv/samba/company

# Ajout alice au groupe
usermod -aG smbusers alice

# Création mot de passe Samba pour alice
(echo "alice123"; echo "alice123") | smbpasswd -s -a alice
smbpasswd -e alice

# Sauvegarde config originale
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak.2026-01-02

# Validation et redémarrage
testparm -s
systemctl enable --now smbd nmbd
systemctl restart smbd nmbd
```

### 7.4 Permissions Finales Dossier Samba

```
drwxrws--- 2 root smbusers 4096 /srv/samba/company
```

---

## 8. Configuration Pare-feu UFW

### 8.1 Règles Configurées

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp      # SSH
ufw allow 445/tcp     # SMB
ufw allow 139/tcp     # SMB (NetBIOS)
ufw allow 137/udp     # NetBIOS Name Service
ufw allow 138/udp     # NetBIOS Datagram
ufw --force enable
```

### 8.2 État UFW Final

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
445/tcp                    ALLOW IN    Anywhere
139/tcp                    ALLOW IN    Anywhere
137/udp                    ALLOW IN    Anywhere
138/udp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
445/tcp (v6)               ALLOW IN    Anywhere (v6)
139/tcp (v6)               ALLOW IN    Anywhere (v6)
137/udp (v6)               ALLOW IN    Anywhere (v6)
138/udp (v6)               ALLOW IN    Anywhere (v6)
```

---

## 9. Ports Ouverts

### 9.1 Résumé des Ports

| Port | Protocole | Service | Description |
|------|-----------|---------|-------------|
| 22 | TCP | SSH | Administration sécurisée |
| 139 | TCP | SMB | NetBIOS Session Service |
| 445 | TCP | SMB | Direct SMB over TCP |
| 137 | UDP | NetBIOS | Name Service |
| 138 | UDP | NetBIOS | Datagram Service |

### 9.2 Vérification (ss -lntup)

```
tcp   LISTEN 0      128    0.0.0.0:22     0.0.0.0:*    sshd
tcp   LISTEN 0      50     0.0.0.0:139    0.0.0.0:*    smbd
tcp   LISTEN 0      50     0.0.0.0:445    0.0.0.0:*    smbd
tcp   LISTEN 0      128    [::]:22        [::]:*       sshd
tcp   LISTEN 0      50     [::]:139       [::]:*       smbd
tcp   LISTEN 0      50     [::]:445       [::]:*       smbd
```

---

## 10. AppArmor

### 10.1 État

```
apparmor module is loaded.
10 profiles are loaded.
10 profiles are in enforce mode.
```

---

## 11. Fichiers Modifiés/Créés

| Fichier | Description |
|---------|-------------|
| `/etc/ssh/sshd_config.d/hardening.conf` | Durcissement SSH + config SFTP chroot |
| `/etc/samba/smb.conf` | Configuration Samba |
| `/etc/samba/smb.conf.bak.2026-01-02` | Sauvegarde config Samba originale |
| `/home/sysadmin/.ssh/authorized_keys` | Clé SSH publique sysadmin |
| `/sftp/alice/upload/` | Dossier SFTP alice |
| `/srv/samba/company/` | Dossier partage Samba |

---

## 12. Tests de Validation

### 12.1 Test Services Actifs

```bash
$ systemctl is-active ssh smbd nmbd apparmor
active
active
active
active
```

### 12.2 Test Samba - Liste des Partages

```bash
$ smbclient -L localhost -U alice%alice123
	Sharename       Type      Comment
	---------       ----      -------
	company         Disk
	IPC$            IPC       IPC Service (Samba 4.17.12-Debian)
SMB1 disabled -- no workgroup available
```

### 12.3 Test Samba - Accès au Partage

```bash
$ smbclient //localhost/company -U alice%alice123 -c "ls"
  .                                   D        0  Fri Jan  2 19:04:57 2026
  ..                                  D        0  Fri Jan  2 19:04:57 2026
  testdir                             D        0  Fri Jan  2 19:10:12 2026

		15427464 blocks of size 1024. 12766936 blocks available
```

### 12.4 Test Samba - Création Dossier

```bash
$ smbclient //localhost/company -U alice%alice123 -c "mkdir testdir; ls"
  .                                   D        0  Fri Jan  2 19:10:12 2026
  ..                                  D        0  Fri Jan  2 19:04:57 2026
  testdir                             D        0  Fri Jan  2 19:10:12 2026
```

### 12.5 Test SFTP - Connexion Chroot

```bash
$ sshpass -p "alice123" sftp -o StrictHostKeyChecking=no alice@127.0.0.1
Connected to 127.0.0.1.
sftp> ls
upload
sftp> exit
```

> **Note** : alice voit uniquement le dossier `upload` car elle est chrootée dans `/sftp/alice/`

### 12.6 Test SSH Config

```bash
$ sshd -T | grep -E "permitrootlogin|passwordauthentication"
permitrootlogin no
passwordauthentication no
```

### 12.7 Test Hostname et IP

```bash
$ hostname
secos-gr8-data

$ ip a | grep "inet " | grep -v 127.0.0.1
    inet 192.168.100.149/24 brd 192.168.100.255 scope global dynamic ens18
```

### 12.8 Test Comptes Utilisateurs

```bash
$ id sysadmin
uid=1000(sysadmin) gid=1000(sysadmin) groupes=1000(sysadmin),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)

$ id alice
uid=1001(alice) gid=1002(alice) groupes=1002(alice),1001(sftpusers),1003(smbusers)
```

---

## 13. Architecture Réseau

```
+-------------------+          +-------------------+          +-------------------+
|  SecOS-GR8-Admin  |          |  SecOS-GR8-Data   |          |   SecOS-GR8-DNS   |
|  192.168.100.120  |  ------> |  192.168.100.149  |          |  192.168.100.136  |
|                   |   SSH    |                   |          |                   |
|  sysadmin (clé)   |   SFTP   |  Samba + SFTP     |          |   Unbound DNS     |
|                   |   SMB    |                   |          |                   |
+-------------------+          +-------------------+          +-------------------+
```

---

## 14. Commandes de Test Depuis VM Admin (192.168.100.120)

```bash
# Test connectivité
ping -c 2 192.168.100.149

# Test résolution DNS (après config DNS)
dig @192.168.100.136 data.secos-gr8.lan +short

# Liste des partages Samba
smbclient -L //192.168.100.149 -U alice

# Accès au partage
smbclient //192.168.100.149/company -U alice -c "ls"

# Test SFTP
sftp alice@192.168.100.149

# Test SSH (sysadmin avec clé)
ssh sysadmin@192.168.100.149
```

---

## 15. Configuration DNS (À faire sur VM DNS 192.168.100.136)

Ajouter dans `/etc/unbound/unbound.conf.d/secos-gr8.conf` :

```yaml
local-data: "data.secos-gr8.lan.  IN A 192.168.100.149"
```

Puis redémarrer :

```bash
systemctl restart unbound
```

---

## 16. Résumé des Identifiants

### Accès SSH

| Utilisateur | Méthode | Depuis |
|-------------|---------|--------|
| sysadmin | Clé SSH | VM Admin (192.168.100.120) |
| root | **INTERDIT** | - |

### Accès SFTP

| Utilisateur | Mot de passe | Dossier accessible |
|-------------|--------------|-------------------|
| alice | `alice123` | `/sftp/alice/upload/` |

### Accès Samba

| Utilisateur | Mot de passe | Partage |
|-------------|--------------|---------|
| alice | `alice123` | `//192.168.100.149/company` |

### Clés SSH Autorisées

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDVCDGYxt15BBmIjmpX+VW1zYMckt1JwxxitQ1KyhMVz sysadmin@secos-gr8
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
```

### Emplacement des Clés sur VM Admin (192.168.100.120)

| Fichier | Description |
|---------|-------------|
| `/root/.ssh/id_ed25519` | Clé **privée** (NE JAMAIS PARTAGER) |
| `/root/.ssh/id_ed25519.pub` | Clé **publique** |

---

## 17. Sécurité Appliquée

| Mesure | Description |
|--------|-------------|
| SSH durci | Root interdit, auth par clé uniquement |
| SFTP chroot | Utilisateurs SFTP isolés dans leur dossier |
| Samba sécurisé | SMB2 minimum, accès LAN uniquement |
| UFW actif | Seuls ports nécessaires ouverts |
| AppArmor | Profils de sécurité actifs |
| Groupes séparés | sftpusers / smbusers pour contrôle d'accès |

---

## 18. Tests Réussis depuis VM Admin

### 18.1 Test SSH (clé)

**Test effectué** : Connexion SSH depuis VM Admin (192.168.100.120) vers VM Data (192.168.100.149)

```bash
root@secos-gr8-admin:~# ssh sysadmin@192.168.100.149
```

**Résultat** : Connexion réussie sans mot de passe (authentification par clé SSH)

---

### 18.2 Test Samba (partage fichiers)

**Test effectué** : Connexion au partage Samba et upload d'un fichier depuis VM Admin

```bash
# 1. Création du fichier sur VM Admin
root@secos-gr8-admin:~# echo "Test depuis Admin" > /tmp/test.txt

# 2. Connexion au partage Samba
root@secos-gr8-admin:~# smbclient //192.168.100.149/company -U alice
Password: alice123

# 3. Upload du fichier
smb: \> lcd /tmp
smb: \> put test.txt
putting file test.txt as \test.txt
smb: \> ls
  .                                   D        0  Fri Jan  2 20:09:00 2026
  ..                                  D        0  Fri Jan  2 19:04:57 2026
  test_admin                          D        0  Fri Jan  2 20:00:00 2026
  testdir                             D        0  Fri Jan  2 19:10:12 2026
  test.txt                            A       18  Fri Jan  2 20:09:00 2026
smb: \> exit
```

**Vérification sur VM Data** :
```bash
root@secos-gr8-data:~# ls -la /srv/samba/company/
total 20
drwxrws--- 4 root  smbusers 4096  2 janv. 20:09 .
drwxr-xr-x 3 root  root     4096  2 janv. 19:04 ..
drwxrws--- 2 alice smbusers 4096  2 janv. 20:00 test_admin
drwxrws--- 2 alice smbusers 4096  2 janv. 19:10 testdir
-rw-rw---- 1 alice smbusers   18  2 janv. 20:09 test.txt

root@secos-gr8-data:~# cat /srv/samba/company/test.txt
Test depuis Admin
```

**Résultat** : Upload réussi - fichier transféré de VM Admin vers VM Data via Samba

---

### 18.3 Résumé des tests

| Test | Commande | Résultat |
|------|----------|----------|
| SSH avec clé | `ssh sysadmin@192.168.100.149` | ✓ Connexion sans mot de passe |
| Samba connexion | `smbclient //192.168.100.149/company -U alice` | ✓ Authentification OK |
| Samba création dossier | `mkdir test_admin` | ✓ Dossier créé |
| Samba upload fichier | `put test.txt` | ✓ Fichier transféré |
| Samba liste fichiers | `ls` | ✓ Contenu visible |

---

## 19. Résumé Exécutif

### Ce qui a été configuré

| Service | Description | Utilité |
|---------|-------------|---------|
| **Samba** | Partage réseau `/srv/samba/company` | Disque réseau partagé pour les employés |
| **SFTP** | Accès fichiers sécurisé avec chroot | Transfert de fichiers sécurisé, utilisateurs isolés |
| **SSH durci** | Auth par clé uniquement, root interdit | Administration sécurisée |
| **UFW** | Pare-feu avec ports minimaux | Réduction surface d'attaque |
| **AppArmor** | Profils de sécurité | Isolation des processus |

### Architecture de Sécurité

```
                    INTERNET
                        |
                   [BLOQUÉ]
                        |
            +-----------+-----------+
            |       PARE-FEU        |
            |   (UFW - deny all)    |
            +-----------+-----------+
                        |
         Seuls ports autorisés:
         22 (SSH), 139/445 (SMB)
                        |
            +-----------+-----------+
            |    VM DATA            |
            |  192.168.100.149      |
            +-----------------------+
            |                       |
    +-------+-------+   +-----------+-----------+
    |     SSH       |   |        SAMBA          |
    | (clé requise) |   | (LAN 192.168.100.x)   |
    +---------------+   +-----------------------+
            |                       |
    +-------+-------+       +-------+-------+
    |  sysadmin     |       |    alice      |
    | (admin sudo)  |       | (smbusers)    |
    +---------------+       +---------------+
```

---

*Document généré automatiquement - SecOS-GR8-Data*
*Dernière mise à jour : 2 janvier 2026*

