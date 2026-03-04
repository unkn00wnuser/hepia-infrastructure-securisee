# SecOS-GR8-Logs - Documentation Technique

## Informations Serveur

| Attribut | Valeur |
|----------|--------|
| **Hostname** | secos-gr8-logs |
| **IP** | 192.168.100.176 |
| **OS** | Debian 12 (Bookworm) |
| **Kernel** | 6.1.0-41-amd64 |
| **Role** | Serveur de logs centralise (rsyslog) |
| **Date de configuration** | 2026-01-05 |

---

## Services Actifs

| Service | Port | Protocole | Status |
|---------|------|-----------|--------|
| SSH | 22 | TCP | Active |
| Syslog (TLS) | 6514 | TCP | Active |
| Syslog (UDP) | 514 | UDP | Active (temporaire) |
| Fail2ban | - | - | Active |
| AppArmor | - | - | Active (10 profils) |

---

## Configuration Reseau

### Pare-feu UFW

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing)

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    192.168.100.0/24    # SSH (LAN only)
514/udp                    ALLOW IN    192.168.100.0/24    # Syslog UDP (temporaire)
6514/tcp                   ALLOW IN    192.168.100.0/24    # Syslog TLS
```

### Ports en ecoute

```bash
# TCP
LISTEN 0.0.0.0:22    # SSH
LISTEN 0.0.0.0:6514  # Syslog TLS

# UDP
UNCONN 0.0.0.0:514   # Syslog UDP (temporaire)
```

---

## Securite SSH

### Configuration (`/etc/ssh/sshd_config.d/hardening.conf`)

```
# Authentification
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
MaxSessions 3

# Timeouts
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# Forwarding (tout desactive)
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no
DisableForwarding yes

# Algorithmes cryptographiques forts
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org

# Logging et restrictions
LogLevel VERBOSE
AllowUsers sysadmin
Banner /etc/issue.net
```

### Acces autorises

| Utilisateur | Methode | Cle SSH |
|-------------|---------|---------|
| sysadmin | Cle publique uniquement | ssh-ed25519 (root@secos-gr8-admin) |

---

## Configuration Rsyslog

### Architecture

```
                    Internet (BLOQUE)
                           |
                           X
                           |
    [LAN 192.168.100.0/24] |
           |               |
    +------+---------------+-------+
    |                              |
    |   SecOS-GR8-Logs             |
    |   192.168.100.176            |
    |                              |
    |   +----------------------+   |
    |   |     rsyslogd         |   |
    |   |                      |   |
    |   |  TCP 6514 (TLS)  <---|---|--- Logs chiffres (RECOMMANDE)
    |   |  UDP 514         <---|---|--- Logs non-chiffres (temporaire)
    |   |                      |   |
    |   +----------+-----------+   |
    |              |               |
    |              v               |
    |   /var/log/remote/           |
    |   +-- secos-gr8-admin/       |
    |   +-- secos-gr8-dns/         |
    |   +-- secos-gr8-auth/        |
    |   +-- ...                    |
    |                              |
    +------------------------------+
```

### Configuration serveur (`/etc/rsyslog.d/00-server.conf`)

```bash
# Module TLS (GnuTLS)
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/rsyslog-tls/ca.pem"
    DefaultNetstreamDriverCertFile="/etc/rsyslog-tls/server-cert.pem"
    DefaultNetstreamDriverKeyFile="/etc/rsyslog-tls/server-key.pem"
)

# Reception TLS securisee (port 6514)
module(load="imtcp" StreamDriver.Name="gtls" StreamDriver.Mode="1" StreamDriver.Authmode="x509/name")
input(type="imtcp" port="6514"
      StreamDriver.Name="gtls"
      StreamDriver.Mode="1"
      StreamDriver.Authmode="x509/name"
      PermittedPeer=["secos-gr8-client", "*.secos-gr8.lan"])

# Reception UDP non-chiffree (temporaire)
module(load="imudp")
input(type="imudp" port="514")

# Template pour logs par host
template(name="PerHostFile" type="string"
    string="/var/log/remote/%HOSTNAME%/syslog.log")

# Routage des logs distants
if $fromhost-ip != "127.0.0.1" then {
    action(type="omfile" dynaFile="PerHostFile")
    stop
}
```

### Certificats TLS

| Fichier | Chemin | Permissions | Description |
|---------|--------|-------------|-------------|
| CA | `/etc/rsyslog-tls/ca.pem` | 644 | Autorite de certification |
| CA Key | `/etc/rsyslog-tls/ca-key.pem` | 600 | Cle privee CA (NE PAS DISTRIBUER) |
| Server Cert | `/etc/rsyslog-tls/server-cert.pem` | 600 | Certificat serveur |
| Server Key | `/etc/rsyslog-tls/server-key.pem` | 600 | Cle privee serveur |
| Client Cert | `/etc/rsyslog-tls/client-cert.pem` | 644 | Certificat client (a distribuer) |
| Client Key | `/etc/rsyslog-tls/client-key.pem` | 600 | Cle privee client (a distribuer) |

**Validite des certificats** : 10 ans (jusqu'en 2036)

---

## Structure des Logs

```
/var/log/remote/
├── secos-gr8-admin/
│   └── syslog.log          # Logs VM Admin (FONCTIONNEL)
├── secos-gr8-dns/
│   └── syslog.log
├── secos-gr8-auth/
│   └── syslog.log
├── secos-gr8-data/
│   └── syslog.log
├── secos-gr8-backup/
│   └── syslog.log
├── secos-gr8-vpn/
│   └── syslog.log
├── secos-gr8-web/
│   └── syslog.log
├── secos-gr8-mail/
│   └── syslog.log
└── secos-gr8-client/
    └── syslog.log
```

### Rotation des logs (`/etc/logrotate.d/remote-logs`)

```
/var/log/remote/*/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 640 root adm
    sharedscripts
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

---

## Hardening Systeme

### Sysctl (`/etc/sysctl.d/99-hardening.conf`)

#### Protection Reseau

| Parametre | Valeur | Description |
|-----------|--------|-------------|
| `net.ipv4.ip_forward` | 0 | Pas de routage |
| `net.ipv4.tcp_syncookies` | 1 | Protection SYN flood |
| `net.ipv4.conf.all.accept_redirects` | 0 | Pas de redirections ICMP |
| `net.ipv4.conf.all.send_redirects` | 0 | Pas d'envoi de redirections |
| `net.ipv4.conf.all.rp_filter` | 1 | Protection anti-spoofing |
| `net.ipv4.conf.all.log_martians` | 1 | Journalisation paquets suspects |
| `net.ipv4.icmp_echo_ignore_broadcasts` | 1 | Protection Smurf |

#### Protection Kernel

| Parametre | Valeur | Description |
|-----------|--------|-------------|
| `kernel.dmesg_restrict` | 1 | Acces dmesg restreint |
| `kernel.kptr_restrict` | 2 | Pointeurs kernel masques |
| `kernel.randomize_va_space` | 2 | ASLR complet |
| `kernel.yama.ptrace_scope` | 1 | ptrace restreint |
| `fs.suid_dumpable` | 0 | Pas de core dumps SUID |
| `fs.protected_symlinks` | 1 | Protection liens symboliques |
| `fs.protected_hardlinks` | 1 | Protection liens physiques |

### Fail2ban

**Configuration** : `/etc/fail2ban/jail.local`

| Parametre | Valeur |
|-----------|--------|
| Jail | sshd |
| maxretry | 3 |
| bantime | 3600 (1 heure) |
| findtime | 600 (10 minutes) |
| backend | systemd |

**Commandes utiles** :
```bash
# Statut
fail2ban-client status sshd

# Liste des IP bannies
fail2ban-client status sshd | grep "Banned IP"

# Debannir une IP
fail2ban-client set sshd unbanip <IP>
```

### AppArmor

```
apparmor module is loaded.
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

---

## Tests Effectues

### Test 1 : Reception logs UDP (VM Admin -> Logs)

```bash
# Sur VM Admin (192.168.100.120)
apt -y install rsyslog
echo "*.* @192.168.100.176:514" | tee /etc/rsyslog.d/99-remote.conf
systemctl restart rsyslog
logger -t ADMIN-TEST "Test depuis VM Admin"

# Sur VM Logs - Verification
cat /var/log/remote/secos-gr8-admin/syslog.log | grep ADMIN-TEST
```

**Resultat** : SUCCES - Logs recus et stockes dans `/var/log/remote/secos-gr8-admin/syslog.log`

### Test 2 : Ports en ecoute

```bash
# TCP
ss -tlnp | grep -E "514|6514"
LISTEN 0.0.0.0:6514  users:(("rsyslogd"...))

# UDP
ss -ulnp | grep 514
UNCONN 0.0.0.0:514   users:(("rsyslogd"...))
```

**Resultat** : SUCCES - Ports 6514/tcp (TLS) et 514/udp actifs

### Test 3 : Validation configuration rsyslog

```bash
rsyslogd -N1
rsyslogd: version 8.2302.0, config validation run (level 1)
rsyslogd: End of config validation run. Bye.
```

**Resultat** : SUCCES - Configuration valide

### Test 4 : Fail2ban SSH

```bash
fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

**Resultat** : SUCCES - Jail SSH active

---

## Configuration des Clients

### Option 1 : UDP (Non chiffre - temporaire)

```bash
# Installation rsyslog
apt -y install rsyslog

# Configuration
echo "*.* @192.168.100.176:514" | tee /etc/rsyslog.d/99-remote.conf

# Redemarrage
systemctl restart rsyslog
```

### Option 2 : TLS (Recommande)

#### Etape 1 : Installer rsyslog-gnutls

```bash
apt -y install rsyslog rsyslog-gnutls
```

#### Etape 2 : Copier les certificats depuis le serveur

```bash
# Depuis le serveur de logs (en tant que root)
scp /etc/rsyslog-tls/ca.pem sysadmin@<CLIENT_IP>:/tmp/
scp /etc/rsyslog-tls/client-cert.pem sysadmin@<CLIENT_IP>:/tmp/
scp /etc/rsyslog-tls/client-key.pem sysadmin@<CLIENT_IP>:/tmp/

# Sur le client
mkdir -p /etc/rsyslog-tls
mv /tmp/ca.pem /tmp/client-*.pem /etc/rsyslog-tls/
chmod 600 /etc/rsyslog-tls/client-key.pem
chmod 644 /etc/rsyslog-tls/ca.pem /etc/rsyslog-tls/client-cert.pem
```

#### Etape 3 : Configurer rsyslog client

```bash
cat > /etc/rsyslog.d/99-remote-tls.conf <<'EOF'
# Configuration TLS pour envoi securise des logs
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/etc/rsyslog-tls/ca.pem"
    DefaultNetstreamDriverCertFile="/etc/rsyslog-tls/client-cert.pem"
    DefaultNetstreamDriverKeyFile="/etc/rsyslog-tls/client-key.pem"
)

# Envoi vers le serveur de logs via TLS
action(
    type="omfwd"
    target="192.168.100.176"
    port="6514"
    protocol="tcp"
    StreamDriver="gtls"
    StreamDriverMode="1"
    StreamDriverAuthMode="x509/name"
    StreamDriverPermittedPeers="secos-gr8-logs.secos-gr8.lan"
)
EOF
```

#### Etape 4 : Redemarrer rsyslog

```bash
systemctl restart rsyslog
```

#### Etape 5 : Tester

```bash
logger -t TLS-TEST "Test connexion TLS"
```

---

## Commandes Utiles

### Rsyslog

```bash
# Statut du service
systemctl status rsyslog

# Voir les logs en temps reel
tail -f /var/log/syslog

# Voir les logs distants
tail -f /var/log/remote/*/syslog.log

# Valider la configuration
rsyslogd -N1

# Redemarrer
systemctl restart rsyslog
```

### Fail2ban

```bash
# Statut general
fail2ban-client status

# Statut SSH
fail2ban-client status sshd

# Debannir une IP
fail2ban-client set sshd unbanip <IP>
```

### UFW

```bash
# Statut
ufw status verbose

# Ajouter une regle
ufw allow from 192.168.100.0/24 to any port <PORT>

# Supprimer une regle
ufw delete allow <PORT>
```

### Securite

```bash
# Verifier les connexions SSH actives
ss -tnp | grep ssh

# Voir les tentatives de connexion echouees
grep "Failed password" /var/log/auth.log

# Verifier AppArmor
aa-status

# Verifier les parametres sysctl
sysctl -a | grep <parametre>
```

---

## Checklist de Securite

- [x] SSH : Authentification par cle uniquement
- [x] SSH : Root login desactive
- [x] SSH : Algorithmes cryptographiques forts
- [x] SSH : Banniere de securite
- [x] SSH : Fail2ban actif
- [x] Rsyslog : TLS configure (port 6514)
- [x] Rsyslog : Certificats generes et configures
- [x] UFW : Actif, deny par defaut
- [x] UFW : Ports restreints au LAN uniquement
- [x] AppArmor : Actif avec profils en enforce
- [x] Sysctl : Hardening reseau et kernel
- [x] Logrotate : Rotation automatique des logs

---

## Prochaines Etapes

1. **Migrer tous les clients vers TLS** : Distribuer les certificats et configurer rsyslog-gnutls sur toutes les VMs
2. **Desactiver UDP 514** : Une fois tous les clients migres, commenter la ligne `input(type="imudp" port="514")` dans `/etc/rsyslog.d/00-server.conf`
3. **Mettre a jour le DNS** : Ajouter l'entree `logs.secos-gr8.lan -> 192.168.100.176` sur le serveur DNS
4. **Monitoring** : Configurer des alertes si le serveur de logs ne recoit plus de messages

---

## Contacts et Support

| Role | Nom | Acces |
|------|-----|-------|
| Admin systeme | sysadmin | SSH via cle depuis secos-gr8-admin |

---

*Document genere le 2026-01-05*
*Serveur : secos-gr8-logs (192.168.100.176)*
