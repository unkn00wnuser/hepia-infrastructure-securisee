# Rapport de Durcissement - Serveur DNS

**Date** : 2026-01-02
**Système** : Debian GNU/Linux 12 (Bookworm)
**Rôle** : Serveur DNS interne (Unbound)
**Hostname** : secos-gr8-dns
**IP** : 192.168.100.136

---

## Table des Matières

1. [Informations Générales](#1-informations-générales)
2. [Installation et Mise à Jour](#2-installation-et-mise-à-jour)
3. [Gestion des Utilisateurs](#3-gestion-des-utilisateurs)
4. [Configuration DNS Unbound](#4-configuration-dns-unbound)
5. [Durcissement SSH](#5-durcissement-ssh)
6. [Pare-feu UFW](#6-pare-feu-ufw)
7. [AppArmor](#7-apparmor)
8. [Vérifications et Tests](#8-vérifications-et-tests)
9. [Fichiers Modifiés](#9-fichiers-modifiés)

---

## 1. Informations Générales

| Élément | Valeur |
|---------|--------|
| VM | SecOS-GR8-DNS |
| OS | Debian 12.11 (Bookworm) |
| vCPU | 1 |
| RAM | 1 Go |
| Disque | 12 Go |
| IP | 192.168.100.136 |
| Interface graphique | Non (serveur) |
| Service principal | DNS (Unbound) |

### Utilisateurs

| Utilisateur | Rôle | Mot de passe | Groupes |
|-------------|------|--------------|---------|
| root | Superutilisateur | (désactivé SSH) | root |
| sysadmin | Administrateur | admin | sysadmin, sudo |

---

## 2. Installation et Mise à Jour

### 2.1 Mise à jour du système

```bash
apt update && apt -y full-upgrade
```

### 2.2 Paquets installés

```bash
apt -y install sudo vim-nox curl ca-certificates openssh-server ufw apparmor apparmor-utils dnsutils unbound
```

| Paquet | Rôle |
|--------|------|
| sudo | Élévation de privilèges |
| vim-nox | Éditeur de texte |
| curl | Transfert de données |
| ca-certificates | Certificats SSL/TLS |
| openssh-server | Serveur SSH |
| ufw | Pare-feu |
| apparmor | LSM (Linux Security Module) |
| apparmor-utils | Utilitaires AppArmor |
| dnsutils | Outils DNS (dig, nslookup) |
| unbound | Serveur DNS récursif |

---

## 3. Gestion des Utilisateurs

### 3.1 Ajout au groupe sudo

```bash
usermod -aG sudo sysadmin
```

### 3.2 Vérification

```bash
id sysadmin
getent group sudo
```

---

## 4. Configuration DNS Unbound

### 4.1 Fichier de Configuration

Fichier : `/etc/unbound/unbound.conf.d/secos-gr8.conf`

```yaml
server:
  interface: 0.0.0.0
  interface: ::0
  port: 53

  # Contrôle d'accès - Réseau local uniquement
  access-control: 127.0.0.0/8 allow
  access-control: ::1 allow
  access-control: 192.168.100.0/24 allow
  access-control: 0.0.0.0/0 refuse
  access-control: ::0/0 refuse

  # Durcissement DNS
  hide-identity: yes
  hide-version: yes
  harden-glue: yes
  harden-dnssec-stripped: yes
  use-caps-for-id: yes

  # Cache
  prefetch: yes
  cache-min-ttl: 300
  cache-max-ttl: 86400

  # Logs
  verbosity: 1

# Zone locale pour l'infrastructure
local-zone: "secos-gr8.lan." static

# Enregistrements DNS internes
local-data: "admin.secos-gr8.lan.  IN A 192.168.100.120"
local-data: "dns.secos-gr8.lan.    IN A 192.168.100.136"
```

### 4.2 Options de Sécurité DNS

| Option | Valeur | Description |
|--------|--------|-------------|
| hide-identity | yes | Cache l'identité du serveur |
| hide-version | yes | Cache la version d'Unbound |
| harden-glue | yes | Protection contre glue records malveillants |
| harden-dnssec-stripped | yes | Protection DNSSEC |
| use-caps-for-id | yes | Randomisation de casse (anti-spoofing) |
| access-control | 192.168.100.0/24 allow | Accès limité au réseau local |

### 4.3 Validation et Démarrage

```bash
unbound-checkconf
systemctl enable --now unbound
systemctl restart unbound
```

### 4.4 Vérification du Service

```bash
ss -lunp | grep ':53'
```

Résultat attendu :
```
UNCONN 0  0  0.0.0.0:53  0.0.0.0:*  users:(("unbound",pid=XXXX,fd=3))
UNCONN 0  0     [::]:53     [::]:*  users:(("unbound",pid=XXXX,fd=5))
```

---

## 5. Durcissement SSH

### 5.1 Fichier de Configuration

Fichier : `/etc/ssh/sshd_config.d/hardening.conf`

```bash
# SSH Hardening Configuration for DNS Server
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes
```

### 5.2 Paramètres Effectifs

| Paramètre | Valeur |
|-----------|--------|
| permitrootlogin | no |
| passwordauthentication | no |
| challengeresponseauthentication | no |
| pubkeyauthentication | yes |

### 5.3 Validation et Rechargement

```bash
sshd -t
systemctl reload ssh
```

### 5.4 Vérification

```bash
sshd -T | egrep "permitrootlogin|passwordauthentication|challengeresponseauthentication|pubkeyauthentication"
```

### 5.5 Configuration Clé SSH (optionnel)

```bash
mkdir -p /home/sysadmin/.ssh
chmod 700 /home/sysadmin/.ssh
chown -R sysadmin:sysadmin /home/sysadmin/.ssh

nano /home/sysadmin/.ssh/authorized_keys
# Coller la clé publique ssh-ed25519 ...

chmod 600 /home/sysadmin/.ssh/authorized_keys
chown sysadmin:sysadmin /home/sysadmin/.ssh/authorized_keys
```

---

## 6. Pare-feu UFW

### 6.1 Configuration

```bash
ufw default deny incoming
ufw default allow outgoing

ufw allow 22/tcp    # SSH
ufw allow 53/udp    # DNS
ufw allow 53/tcp    # DNS (transferts de zone)

ufw --force enable
```

### 6.2 Règles Actives

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
53/udp                     ALLOW IN    Anywhere
53/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
53/udp (v6)                ALLOW IN    Anywhere (v6)
53/tcp (v6)                ALLOW IN    Anywhere (v6)
```

### 6.3 Vérification

```bash
ufw status verbose
```

---

## 7. AppArmor

### 7.1 Activation

```bash
systemctl enable --now apparmor
```

### 7.2 Vérification

```bash
aa-status | head
```

Résultat attendu :
```
apparmor module is loaded.
X profiles are loaded.
X profiles are in enforce mode.
```

---

## 8. Vérifications et Tests

### 8.1 Ports en Écoute

```bash
ss -lntup | egrep ":22|:53"
```

Résultat attendu :
```
tcp   LISTEN  0  128  0.0.0.0:22    0.0.0.0:*    users:(("sshd"...))
udp   UNCONN  0  0    0.0.0.0:53    0.0.0.0:*    users:(("unbound"...))
tcp   LISTEN  0  0    0.0.0.0:53    0.0.0.0:*    users:(("unbound"...))
```

### 8.2 Tests DNS depuis la VM Admin (192.168.100.120)

```bash
# Test résolution locale
dig @192.168.100.136 admin.secos-gr8.lan +short
# Résultat attendu : 192.168.100.120

dig @192.168.100.136 dns.secos-gr8.lan +short
# Résultat attendu : 192.168.100.136

# Test résolution externe (récursive)
dig @192.168.100.136 google.com +short
# Résultat attendu : IPs publiques Google
```

### 8.3 Checklist de Validation

| Test | Commande | Résultat Attendu |
|------|----------|------------------|
| DNS local | `dig @192.168.100.136 admin.secos-gr8.lan +short` | 192.168.100.120 |
| DNS récursif | `dig @192.168.100.136 google.com +short` | IPs Google |
| SSH config | `sshd -T \| grep permitrootlogin` | no |
| UFW actif | `ufw status` | Status: active |
| AppArmor | `aa-status` | module is loaded |
| Port 53 | `ss -lunp \| grep :53` | unbound listening |

---

## 9. Fichiers Modifiés

| Fichier | Action | Description |
|---------|--------|-------------|
| `/etc/unbound/unbound.conf.d/secos-gr8.conf` | Créé | Configuration DNS Unbound |
| `/etc/ssh/sshd_config.d/hardening.conf` | Créé | Durcissement SSH |
| `/home/sysadmin/.ssh/authorized_keys` | Créé | Clé publique SSH (optionnel) |

---

## Enregistrements DNS - À Compléter

Quand les autres VMs seront configurées, ajouter leurs IPs dans `/etc/unbound/unbound.conf.d/secos-gr8.conf` :

```yaml
local-data: "logs.secos-gr8.lan.   IN A 192.168.100.XXX"
local-data: "auth.secos-gr8.lan.   IN A 192.168.100.XXX"
local-data: "data.secos-gr8.lan.   IN A 192.168.100.XXX"
local-data: "backup.secos-gr8.lan. IN A 192.168.100.XXX"
local-data: "vpn.secos-gr8.lan.    IN A 192.168.100.XXX"
local-data: "web.secos-gr8.lan.    IN A 192.168.100.XXX"
local-data: "mail.secos-gr8.lan.   IN A 192.168.100.XXX"
local-data: "client.secos-gr8.lan. IN A 192.168.100.XXX"
```

Puis recharger :
```bash
systemctl restart unbound
```

---

## Auteur

Rapport de durcissement baseline - VM DNS
**Serveur DNS opérationnel et sécurisé.**
