# Rapport de Configuration - Serveur LDAP (Auth)

**Date** : 2026-01-02
**Système** : Debian GNU/Linux 12 (Bookworm)
**Rôle** : Serveur LDAP (OpenLDAP) - Annuaire centralisé
**Hostname** : secos-gr8-auth
**IP** : 192.168.100.139

---

## Table des Matières

1. [Informations Générales](#1-informations-générales)
2. [Installation et Configuration](#2-installation-et-configuration)
3. [Configuration OpenLDAP](#3-configuration-openldap)
4. [Structure LDAP](#4-structure-ldap)
5. [Utilisateurs et Groupes LDAP](#5-utilisateurs-et-groupes-ldap)
6. [Durcissement Baseline](#6-durcissement-baseline)
7. [Tests et Vérifications](#7-tests-et-vérifications)
8. [Incidents et Résolutions](#8-incidents-et-résolutions)
9. [Fichiers Modifiés](#9-fichiers-modifiés)

---

## 1. Informations Générales

| Élément | Valeur |
|---------|--------|
| VM | SecOS-GR8-Auth |
| OS | Debian 12 (Bookworm) |
| vCPU | 1 |
| RAM | 1 Go |
| Disque | 16 Go |
| IP | 192.168.100.139 |
| Interface graphique | Non (serveur) |
| Service principal | OpenLDAP (slapd) |

### Utilisateurs Système

| Utilisateur | Rôle | Mot de passe | Groupes |
|-------------|------|--------------|---------|
| root | Superutilisateur | (désactivé SSH) | root |
| sysadmin | Administrateur | admin | sysadmin, sudo |

### Credentials LDAP

| Élément | Valeur |
|---------|--------|
| Base DN | dc=secos-gr8,dc=lan |
| Admin DN | cn=admin,dc=secos-gr8,dc=lan |
| Admin Password | admin |
| Port | 389/tcp |

---

## 2. Installation et Configuration

### 2.1 Mise à jour du système

```bash
apt update && apt -y full-upgrade
```

### 2.2 Paquets installés

```bash
apt -y install sudo vim-nox curl ca-certificates openssh-server ufw apparmor apparmor-utils slapd ldap-utils
```

| Paquet | Rôle |
|--------|------|
| sudo | Élévation de privilèges |
| vim-nox | Éditeur de texte |
| openssh-server | Serveur SSH |
| ufw | Pare-feu |
| apparmor | LSM (Linux Security Module) |
| slapd | Serveur OpenLDAP |
| ldap-utils | Outils LDAP (ldapsearch, ldapadd, etc.) |

---

## 3. Configuration OpenLDAP

### 3.1 Reconfiguration initiale

```bash
dpkg-reconfigure slapd
```

**Réponses fournies :**

| Question | Réponse |
|----------|---------|
| Omettre la configuration initiale ? | Non |
| Nom de domaine DNS | secos-gr8.lan |
| Organisation | SecOS-GR8 |
| Mot de passe admin | admin |
| Backend | MDB |
| Supprimer la base lors de purge ? | Non |
| Déplacer ancienne DB ? | Oui |

### 3.2 Vérification du service

```bash
systemctl status slapd --no-pager
```

Résultat : `active (running)`

```bash
ss -lntp | grep :389
```

Résultat :
```
LISTEN  0  2048  0.0.0.0:389  0.0.0.0:*  users:(("slapd",pid=XXX,fd=8))
LISTEN  0  2048     [::]:389     [::]:*  users:(("slapd",pid=XXX,fd=9))
```

---

## 4. Structure LDAP

### 4.1 Arborescence créée

```
dc=secos-gr8,dc=lan
├── ou=people      (utilisateurs)
├── ou=groups      (groupes)
└── ou=services    (comptes de service)
```

### 4.2 Fichier LDIF - Création des OUs

Fichier : `base-ou.ldif`

```ldif
dn: ou=people,dc=secos-gr8,dc=lan
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=secos-gr8,dc=lan
objectClass: organizationalUnit
ou: groups

dn: ou=services,dc=secos-gr8,dc=lan
objectClass: organizationalUnit
ou: services
```

**Commande d'ajout :**

```bash
ldapadd -x -D "cn=admin,dc=secos-gr8,dc=lan" -W -f base-ou.ldif
```

### 4.3 Vérification de la structure

```bash
ldapsearch -x -LLL -H ldap://127.0.0.1 -b "dc=secos-gr8,dc=lan" dn
```

Résultat :
```
dn: dc=secos-gr8,dc=lan
dn: ou=people,dc=secos-gr8,dc=lan
dn: ou=groups,dc=secos-gr8,dc=lan
dn: ou=services,dc=secos-gr8,dc=lan
```

---

## 5. Utilisateurs et Groupes LDAP

### 5.1 Groupe : itadmins

Fichier : `group-itadmins.ldif`

```ldif
dn: cn=itadmins,ou=groups,dc=secos-gr8,dc=lan
objectClass: top
objectClass: posixGroup
cn: itadmins
gidNumber: 2000
```

**Commande d'ajout :**

```bash
ldapadd -x -D "cn=admin,dc=secos-gr8,dc=lan" -W -f group-itadmins.ldif
```

**Vérification :**

```bash
ldapsearch -x -LLL -H ldap://127.0.0.1 -b "ou=groups,dc=secos-gr8,dc=lan" "(cn=itadmins)" dn cn gidNumber
```

Résultat :
```
dn: cn=itadmins,ou=groups,dc=secos-gr8,dc=lan
cn: itadmins
gidNumber: 2000
```

### 5.2 Utilisateur : alice

**Génération du hash mot de passe :**

```bash
slappasswd
```

Fichier : `user-alice.ldif`

```ldif
dn: uid=alice,ou=people,dc=secos-gr8,dc=lan
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: Alice Admin
sn: Admin
uid: alice
uidNumber: 30000
gidNumber: 2000
homeDirectory: /home/alice
loginShell: /bin/bash
userPassword: {SSHA}HASH_DU_MOT_DE_PASSE
```

**Commande d'ajout :**

```bash
ldapadd -x -D "cn=admin,dc=secos-gr8,dc=lan" -W -f user-alice.ldif
```

**Vérification :**

```bash
ldapsearch -x -LLL -H ldap://127.0.0.1 -b "ou=people,dc=secos-gr8,dc=lan" "(uid=alice)" dn uid uidNumber gidNumber
```

Résultat :
```
dn: uid=alice,ou=people,dc=secos-gr8,dc=lan
uid: alice
uidNumber: 30000
gidNumber: 2000
```

### 5.3 Récapitulatif des comptes LDAP

| Type | DN | Attributs clés |
|------|-----|----------------|
| Groupe | cn=itadmins,ou=groups,dc=secos-gr8,dc=lan | gidNumber: 2000 |
| Utilisateur | uid=alice,ou=people,dc=secos-gr8,dc=lan | uidNumber: 30000, gidNumber: 2000 |

---

## 6. Durcissement Baseline

### 6.1 SSH Durci

Fichier : `/etc/ssh/sshd_config.d/hardening.conf`

```bash
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
```

**Validation et rechargement :**

```bash
sshd -t
systemctl reload ssh
```

### 6.2 Pare-feu UFW

**Configuration :**

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 389/tcp
ufw --force enable
```

**Règles actives :**

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
389/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
389/tcp (v6)               ALLOW IN    Anywhere (v6)
```

### 6.3 AppArmor

```bash
systemctl enable --now apparmor
aa-status | head
```

Résultat :
```
apparmor module is loaded.
X profiles are loaded.
X profiles are in enforce mode.
```

---

## 7. Tests et Vérifications

### 7.1 Tests locaux (sur la VM Auth)

**Service actif :**

```bash
ss -lntp | grep :389
```

**Requête LDAP locale :**

```bash
ldapsearch -x -LLL -H ldap://127.0.0.1 -b "dc=secos-gr8,dc=lan" dn
```

### 7.2 Tests réseau (depuis VM Admin - 192.168.100.120)

**Connectivité :**

```bash
ping 192.168.100.139
```

Résultat : réponses OK (0% loss)

**Résolution DNS :**

```bash
dig @192.168.100.136 auth.secos-gr8.lan +short
```

Résultat : `192.168.100.139`

**Requête LDAP réseau :**

```bash
ldapsearch -x -LLL -H ldap://192.168.100.139 -b "dc=secos-gr8,dc=lan" dn
```

Résultat :
```
dn: dc=secos-gr8,dc=lan
dn: ou=people,dc=secos-gr8,dc=lan
dn: ou=groups,dc=secos-gr8,dc=lan
dn: ou=services,dc=secos-gr8,dc=lan
```

**Test utilisateur alice :**

```bash
ldapsearch -x -LLL -H ldap://192.168.100.139 -b "dc=secos-gr8,dc=lan" "(uid=alice)" dn uid
```

Résultat :
```
dn: uid=alice,ou=people,dc=secos-gr8,dc=lan
uid: alice
```

### 7.3 Checklist de validation

| Test | Commande | Résultat |
|------|----------|----------|
| Service LDAP actif | `systemctl status slapd` | active (running) |
| Port 389 en écoute | `ss -lntp \| grep :389` | ✅ |
| Structure LDAP | `ldapsearch ... dn` | 4 entrées |
| Groupe itadmins | `ldapsearch ... "(cn=itadmins)"` | ✅ |
| Utilisateur alice | `ldapsearch ... "(uid=alice)"` | ✅ |
| UFW actif | `ufw status` | active |
| AppArmor | `aa-status` | loaded |
| Accès réseau | `ldapsearch -H ldap://192.168.100.139` | ✅ |

---

## 8. Incidents et Résolutions

### 8.1 Erreur "Server is unwilling to perform (53)"

**Symptôme :**
```
ldap_add: Server is unwilling to perform (53)
additional info: no global superior knowledge
```

**Cause :** Typo dans les DN des fichiers LDIF (`dc=decos-gr8` au lieu de `dc=secos-gr8`)

**Résolution :** Correction des fichiers LDIF avec le bon DN : `dc=secos-gr8,dc=lan`

---

## 9. Fichiers Modifiés

| Fichier | Action | Description |
|---------|--------|-------------|
| `/etc/ssh/sshd_config.d/hardening.conf` | Créé | Durcissement SSH |
| `base-ou.ldif` | Créé | Création des OUs |
| `group-itadmins.ldif` | Créé | Création groupe itadmins |
| `user-alice.ldif` | Créé | Création utilisateur alice |

---

## Entrée DNS associée

Sur la VM DNS (192.168.100.136), ajouter dans `/etc/unbound/unbound.conf.d/secos-gr8.conf` :

```yaml
local-data: "auth.secos-gr8.lan.  IN A 192.168.100.139"
```

---

## Commandes utiles

**Ajouter un utilisateur :**
```bash
ldapadd -x -D "cn=admin,dc=secos-gr8,dc=lan" -W -f user.ldif
```

**Rechercher un utilisateur :**
```bash
ldapsearch -x -LLL -H ldap://localhost -b "ou=people,dc=secos-gr8,dc=lan" "(uid=USERNAME)"
```

**Supprimer une entrée :**
```bash
ldapdelete -x -D "cn=admin,dc=secos-gr8,dc=lan" -W "uid=USERNAME,ou=people,dc=secos-gr8,dc=lan"
```

**Modifier un attribut :**
```bash
ldapmodify -x -D "cn=admin,dc=secos-gr8,dc=lan" -W -f modifications.ldif
```

---

## Auteur

Rapport de configuration - VM Auth (LDAP)
**Serveur LDAP opérationnel et sécurisé.**
