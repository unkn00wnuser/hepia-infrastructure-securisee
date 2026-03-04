# Configuration SecOS-GR8-Web - Serveur Web Nginx

**Date de configuration:** 3 janvier 2026
**Projet:** Sécurité des OS - HEPIA 2025-26 - Groupe 8

---

## 📋 Informations Système

| Élément | Valeur |
|---------|--------|
| **Hostname** | secos-gr8-web |
| **Rôle** | Serveur Web (Nginx) |
| **OS** | Debian 12 (Bookworm) |
| **Kernel** | Linux 6.1.0-41-amd64 |
| **vCPU** | 2 |
| **RAM** | 2 Go |
| **Disque** | 24 Go |
| **IP LAN** | 192.168.100.157 |
| **Domaine** | web.secos-gr8.lan |
| **Serveur DNS** | 192.168.100.136 |

---

## 🔧 Logiciels Installés

### Services Principaux

| Service | Version | Statut |
|---------|---------|--------|
| **Nginx** | 1.22.1-9+deb12u3 | ✓ Actif |
| **OpenSSH Server** | 9.2p1-2+deb12u7 | ✓ Actif |
| **OpenSSL** | 3.0.17-1~deb12u3 | ✓ Installé |
| **UFW** | 0.36.2-1 | ✓ Actif |
| **AppArmor** | 3.0.8-3 | ✓ Actif |

### Paquets Utilitaires

- vim (2:9.0.1378-2+deb12u2)
- curl (7.88.1-10+deb12u14)
- ca-certificates (20230311+deb12u1)
- sudo (1.9.13p3-1+deb12u2)

---

## 🔐 Configuration Sécurité

### 1. SSH Durci

**Fichier:** `/etc/ssh/sshd_config.d/hardening.conf`

```conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowAgentForwarding no
```

**Clé SSH autorisée pour sysadmin:**
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
```

**Emplacement:** `/home/sysadmin/.ssh/authorized_keys`
**Permissions:** 700 (.ssh), 600 (authorized_keys)

### 2. Pare-feu UFW

**Configuration:**
```
Default: deny (incoming), allow (outgoing)
Logging: on (low)
```

**Règles actives:**

| Port | Protocole | Service | Direction |
|------|-----------|---------|-----------|
| 22 | TCP | SSH | IN |
| 80 | TCP | HTTP | IN |
| 443 | TCP | HTTPS | IN |

**Commandes de configuration:**
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

### 3. AppArmor

**Statut:** 10 profils chargés en mode enforce
- Profils actifs pour system utilities, NetworkManager, dhclient, etc.

---

## 🌐 Configuration Nginx

### Structure des Fichiers

| Fichier | Description |
|---------|-------------|
| `/etc/nginx/sites-available/secos-gr8` | Configuration du site |
| `/etc/nginx/sites-enabled/secos-gr8` | Lien symbolique vers la config |
| `/etc/nginx/conf.d/security.conf` | Paramètres de sécurité globaux |
| `/var/www/secos-gr8/index.html` | Page d'accueil |
| `/etc/nginx/ssl/secos-gr8-web.crt` | Certificat SSL (644) |
| `/etc/nginx/ssl/secos-gr8-web.key` | Clé privée SSL (600) |

### Configuration du Site

**Fichier:** `/etc/nginx/sites-available/secos-gr8`

**Fonctionnalités:**
- Redirection automatique HTTP (port 80) → HTTPS (port 443)
- Support HTTP/2
- Certificat SSL auto-signé
- Headers de sécurité complets
- Logs séparés pour access et error

**Protocoles SSL:**
- TLS 1.2
- TLS 1.3

**Ciphers:**
```
ECDHE-ECDSA-AES128-GCM-SHA256
ECDHE-RSA-AES128-GCM-SHA256
ECDHE-ECDSA-AES256-GCM-SHA384
ECDHE-RSA-AES256-GCM-SHA384
```

**Headers de Sécurité:**
```nginx
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### Configuration Sécurité Globale

**Fichier:** `/etc/nginx/conf.d/security.conf`

```nginx
server_tokens off;                    # Masquer version Nginx
client_max_body_size 10M;             # Limite taille requêtes
client_body_buffer_size 128k;
client_body_timeout 12;
client_header_timeout 12;
keepalive_timeout 15;
send_timeout 10;
```

### Logs

| Type | Emplacement |
|------|-------------|
| Access | `/var/log/nginx/secos-gr8-access.log` |
| Error | `/var/log/nginx/secos-gr8-error.log` |

---

## 🔒 Certificat SSL

**Type:** Auto-signé (RSA 2048 bits)
**Validité:** 365 jours

**Détails:**
```
Subject: C=CH, ST=Geneva, L=Geneva, O=SecOS-GR8, OU=IT, CN=web.secos-gr8.lan
Not Before: Jan  3 14:49:04 2026 GMT
Not After:  Jan  3 14:49:04 2027 GMT
```

**Commande de génération:**
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/secos-gr8-web.key \
  -out /etc/nginx/ssl/secos-gr8-web.crt \
  -subj "/C=CH/ST=Geneva/L=Geneva/O=SecOS-GR8/OU=IT/CN=web.secos-gr8.lan"
```

---

## 👥 Comptes Utilisateurs

| Compte | UID | Groupes | Type | Usage |
|--------|-----|---------|------|-------|
| sysadmin | 1000 | sysadmin, sudo, cdrom, floppy, audio, dip, video, plugdev, users, netdev | Utilisateur standard | Administration SSH (clé uniquement) |
| www-data | 33 | www-data | Système | Processus Nginx |

---

## ✅ Tests de Vérification Réalisés

### 1. Services Actifs

```bash
systemctl status nginx --no-pager
systemctl status ssh --no-pager
```

**Résultat:** ✓ Tous les services actifs et fonctionnels

### 2. Ports en Écoute

```bash
ss -lntp | grep -E ":22|:80|:443"
```

**Résultat:**
```
LISTEN 0.0.0.0:22   (sshd)
LISTEN 0.0.0.0:80   (nginx)
LISTEN 0.0.0.0:443  (nginx)
LISTEN [::]:22      (sshd)
LISTEN [::]:80      (nginx)
LISTEN [::]:443     (nginx)
```

✓ Tous les ports requis en écoute

### 3. Configuration Nginx

```bash
nginx -t
```

**Résultat:**
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

✓ Configuration valide

### 4. Certificat SSL

```bash
openssl x509 -in /etc/nginx/ssl/secos-gr8-web.crt -noout -subject -dates
```

**Résultat:** ✓ Certificat valide pour web.secos-gr8.lan, expire le 3 janvier 2027

### 5. Test HTTP Local

```bash
curl -I http://localhost
```

**Résultat:**
```
HTTP/1.1 301 Moved Permanently
Server: nginx
Location: https://localhost/
```

✓ Redirection HTTP → HTTPS fonctionnelle

### 6. Test HTTPS Local

```bash
curl -Ik https://localhost
```

**Résultat:**
```
HTTP/2 200
server: nginx
x-frame-options: SAMEORIGIN
x-content-type-options: nosniff
x-xss-protection: 1; mode=block
strict-transport-security: max-age=31536000; includeSubDomains
```

✓ HTTPS fonctionnel avec HTTP/2 et headers de sécurité

### 7. Test Résolution DNS (depuis VM Admin)

```bash
dig @192.168.100.136 web.secos-gr8.lan +short
```

**Résultat:** `192.168.100.157`
✓ Résolution DNS fonctionnelle

### 8. Test Ping (depuis VM Admin)

```bash
ping -c 4 web.secos-gr8.lan
```

**Résultat:** ✓ Connectivité réseau OK

### 9. Test HTTPS Distant (depuis VM Admin)

```bash
curl -Ik https://web.secos-gr8.lan
```

**Résultat:** ✓ Page accessible via HTTPS

### 10. Test Navigateur

**URL:** `https://web.secos-gr8.lan`
**Résultat:** ✓ Page d'accueil affichée correctement (après acceptation du certificat auto-signé)

---

## 🌍 Configuration DNS

**Sur VM DNS (192.168.100.136):**

**Fichier:** `/etc/unbound/unbound.conf.d/secos-gr8.conf`

```yaml
local-data: "web.secos-gr8.lan.  IN A 192.168.100.157"
```

**Redémarrage du service:**
```bash
systemctl restart unbound
```

**Sur VM Admin - Configuration DNS client:**

**Fichier:** `/etc/resolv.conf`
```
nameserver 192.168.100.136
```

---

## 📊 Logs d'Accès (Exemples)

```
127.0.0.1 - - [03/Jan/2026:15:54:13 +0100] "HEAD / HTTP/2.0" 200 0 "-" "curl/7.88.1"
127.0.0.1 - - [03/Jan/2026:15:54:28 +0100] "GET / HTTP/2.0" 200 2884 "-" "curl/7.88.1"
192.168.100.120 - - [03/Jan/2026:16:02:49 +0100] "GET / HTTP/2.0" 200 2884 "-" "curl/7.88.1"
```

Aucune erreur détectée dans les logs.

---

## 🛠️ Commandes Utiles

### Gestion Nginx

```bash
# Vérifier la configuration
nginx -t

# Recharger la configuration (sans interruption)
systemctl reload nginx

# Redémarrer Nginx
systemctl restart nginx

# Voir le statut
systemctl status nginx

# Voir les logs en temps réel
tail -f /var/log/nginx/secos-gr8-access.log
tail -f /var/log/nginx/secos-gr8-error.log
```

### Gestion UFW

```bash
# Voir le statut
ufw status verbose

# Ajouter une règle
ufw allow <port>/tcp

# Supprimer une règle
ufw delete allow <port>/tcp

# Recharger UFW
ufw reload
```

### Gestion SSH

```bash
# Tester la configuration
sshd -t

# Redémarrer SSH
systemctl restart ssh

# Voir les connexions actives
who
```

### Diagnostic Réseau

```bash
# Voir les ports en écoute
ss -lntp

# Voir les connexions actives
ss -tnp

# Tester la connectivité
ping web.secos-gr8.lan

# Tester le DNS
dig web.secos-gr8.lan
nslookup web.secos-gr8.lan
```

### Vérification Certificat SSL

```bash
# Voir les détails du certificat
openssl x509 -in /etc/nginx/ssl/secos-gr8-web.crt -noout -text

# Tester la connexion SSL
openssl s_client -connect localhost:443 -servername web.secos-gr8.lan
```

---

## 🎯 Points de Sécurité Implémentés

### Niveau Système
- ✅ Système à jour (apt update && upgrade)
- ✅ Pare-feu UFW actif avec politique restrictive
- ✅ AppArmor actif avec profils enforce
- ✅ SSH durci (pas de root, pas de password, clés seulement)
- ✅ Utilisateur sudo configuré (sysadmin)

### Niveau Nginx
- ✅ HTTPS obligatoire (redirection HTTP → HTTPS)
- ✅ TLS moderne uniquement (1.2/1.3)
- ✅ Ciphers sécurisés
- ✅ Version Nginx masquée
- ✅ Headers de sécurité complets
- ✅ HSTS activé (max-age=31536000)
- ✅ Protection clickjacking (X-Frame-Options)
- ✅ Protection XSS
- ✅ Content-Type sniffing désactivé
- ✅ Fichiers cachés bloqués
- ✅ Limites de taille des requêtes
- ✅ Timeouts configurés

### Niveau Réseau
- ✅ Ports minimaux ouverts (22, 80, 443)
- ✅ Résolution DNS fonctionnelle
- ✅ Certificat SSL configuré

---

## 📝 Notes et Remarques

### Problèmes Résolus

**Problème 1:** Site inaccessible via nom de domaine depuis navigateur
**Cause:** `/etc/resolv.conf` sur VM Admin ne pointait pas vers le serveur DNS
**Solution:** Ajout de `nameserver 192.168.100.136` dans `/etc/resolv.conf`

### Améliorations Futures Possibles

1. **Certificat SSL:** Remplacer le certificat auto-signé par un certificat Let's Encrypt ou un CA interne
2. **Monitoring:** Installer fail2ban pour protection contre brute-force SSH
3. **Logs:** Configurer logrotate pour rotation automatique des logs
4. **Backup:** Configurer des sauvegardes automatiques de la configuration
5. **Content:** Ajouter du contenu web supplémentaire selon les besoins

---

## 📚 Références

- **Documentation Nginx:** https://nginx.org/en/docs/
- **Debian Security:** https://www.debian.org/security/
- **UFW Guide:** https://help.ubuntu.com/community/UFW
- **SSL Best Practices:** https://ssl-config.mozilla.org/

---

## ✨ Statut Final

**État:** ✅ OPÉRATIONNEL
**Sécurité:** ✅ DURCI
**Tests:** ✅ VALIDÉS
**Accès:** `https://web.secos-gr8.lan` (192.168.100.157)

---

*Configuration réalisée le 3 janvier 2026 par Claude Code*
*Projet SecOS-GR8 - Groupe 8 - HEPIA 2025-26*
