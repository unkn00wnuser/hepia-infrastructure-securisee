# Documentation SecOS-GR8-Mail

## Informations de la VM

| Element | Valeur |
|---------|--------|
| Hostname | `secos-gr8-mail` |
| FQDN | `secos-gr8-mail.secos-gr8.lan` |
| IP | `192.168.100.162` |
| OS | Debian 12 (Bookworm) |
| Role | Serveur Mail (Postfix + Dovecot) |
| Domaine mail | `secos-gr8.lan` |

---

## Comptes Utilisateurs

| Compte | Mot de passe | Usage |
|--------|--------------|-------|
| root | `admin` | Administration root |
| sysadmin | Cle SSH uniquement | Administration (sudo) |
| mailuser1 | `mail123` | Utilisateur mail test |
| mailuser2 | `mail123` | Utilisateur mail test |

### Cle SSH sysadmin
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
```

---

## Services Installes

| Service | Version | Status |
|---------|---------|--------|
| Postfix | 3.7.11 | Actif |
| Dovecot | 2.3.19.1 | Actif |
| SSH | OpenSSH 9.2 | Actif (durci) |
| UFW | 0.36.2 | Actif |
| AppArmor | 3.0.8 | Actif |

---

## Ports Ouverts (UFW)

| Port | Protocole | Service |
|------|-----------|---------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 587 | TCP | Submission |
| 143 | TCP | IMAP |
| 993 | TCP | IMAPS (SSL) |
| 110 | TCP | POP3 |
| 995 | TCP | POP3S (SSL) |

---

## Configuration Postfix

### Fichier principal : `/etc/postfix/main.cf`

```
# Identification
smtpd_banner = $myhostname ESMTP SecOS-GR8 Mail Server
biff = no
append_dot_mydomain = no

# Domaine et reseau
myhostname = secos-gr8-mail.secos-gr8.lan
mydomain = secos-gr8.lan
myorigin = $mydomain
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8 192.168.100.0/24

# Boites aux lettres
home_mailbox = Maildir/
mailbox_size_limit = 51200000
message_size_limit = 10240000

# Relay
relayhost =
inet_interfaces = all
inet_protocols = ipv4

# Aliases
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases

# Securite de base
smtpd_helo_required = yes
disable_vrfy_command = yes

# Restrictions SMTP
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination

# TLS (certificat auto-signe)
smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level = may
smtp_tls_security_level = may

# Compatibilite
compatibility_level = 3.6
```

### Port Submission (587) dans `/etc/postfix/master.cf`

```
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=may
  -o smtpd_sasl_auth_enable=no
  -o smtpd_relay_restrictions=permit_mynetworks,reject
  -o milter_macro_daemon_name=ORIGINATING
```

---

## Configuration Dovecot

### `/etc/dovecot/conf.d/10-master.conf`
```
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service pop3-login {
  inet_listener pop3 {
    port = 110
  }
  inet_listener pop3s {
    port = 995
    ssl = yes
  }
}

service auth {
  unix_listener auth-userdb {
    mode = 0666
  }
}
```

### `/etc/dovecot/conf.d/10-auth.conf`
```
disable_plaintext_auth = no
auth_mechanisms = plain login

!include auth-system.conf.ext
```

### `/etc/dovecot/conf.d/10-mail.conf`
```
mail_location = maildir:~/Maildir
namespace inbox {
  inbox = yes
}
```

### `/etc/dovecot/conf.d/10-ssl.conf`
```
ssl = yes
ssl_cert = </etc/ssl/certs/ssl-cert-snakeoil.pem
ssl_key = </etc/ssl/private/ssl-cert-snakeoil.key
ssl_min_protocol = TLSv1.2
```

---

## Configuration SSH (Durci)

### `/etc/ssh/sshd_config.d/hardening.conf`
```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowAgentForwarding no
```

---

## Configuration /etc/hosts

```
127.0.0.1       localhost
127.0.1.1       secos-gr8-mail.secos-gr8.lan secos-gr8-mail
192.168.100.162 secos-gr8-mail.secos-gr8.lan secos-gr8-mail mail

# Infrastructure SecOS-GR8
192.168.100.120 admin.secos-gr8.lan admin
192.168.100.136 dns.secos-gr8.lan dns
192.168.100.139 auth.secos-gr8.lan auth
192.168.100.149 data.secos-gr8.lan data
192.168.100.151 backup.secos-gr8.lan backup
192.168.100.153 vpn.secos-gr8.lan vpn
192.168.100.157 web.secos-gr8.lan web
192.168.100.159 client.secos-gr8.lan client
```

---

## Comment Envoyer un Mail

### Methode 1 : Depuis le serveur mail (ligne de commande)

```bash
# Envoyer un mail simple
echo "Corps du message" | mail -s "Sujet" mailuser1@secos-gr8.lan

# Envoyer en tant qu'utilisateur specifique
echo "Message" | sudo -u mailuser1 mail -s "Sujet" mailuser2@secos-gr8.lan
```

### Methode 2 : Depuis une autre VM (ex: Admin) avec Postfix

1. Installer Postfix sur la VM :
```bash
apt update
apt install -y postfix mailutils
```
   - Choisir : **Satellite system**
   - Mail name : `secos-gr8.lan`
   - Relay host : `192.168.100.162`

2. Envoyer un mail :
```bash
echo "Message depuis Admin" | mail -s "Test" mailuser1@secos-gr8.lan
```

### Methode 3 : Avec telnet (sans installation)

```bash
telnet 192.168.100.162 25
```
Puis taper :
```
EHLO monpc.secos-gr8.lan
MAIL FROM:<expediteur@secos-gr8.lan>
RCPT TO:<mailuser1@secos-gr8.lan>
DATA
Subject: Mon sujet

Mon message ici.
.
QUIT
```

---

## Comment Lire les Mails

### Methode 1 : Ligne de commande (sur le serveur)

```bash
# Lister les mails recus
ls /home/mailuser1/Maildir/new/

# Lire tous les mails
cat /home/mailuser1/Maildir/new/*

# Lire un mail specifique
cat /home/mailuser1/Maildir/new/nom_du_fichier
```

### Methode 2 : Via IMAP avec telnet

```bash
telnet 192.168.100.162 143
```
Puis :
```
a LOGIN mailuser1 mail123
b SELECT INBOX
c FETCH 1 BODY[]
d LOGOUT
```

### Methode 3 : Client mail (Thunderbird, etc.)

Configuration :
- Serveur IMAP : `192.168.100.162` ou `mail.secos-gr8.lan`
- Port IMAP : `143` (ou `993` pour SSL)
- Serveur SMTP : `192.168.100.162` ou `mail.secos-gr8.lan`
- Port SMTP : `25` ou `587`
- Utilisateur : `mailuser1`
- Mot de passe : `mail123`

---

## Tests Effectues

### Test 1 : Envoi local (OK)
```
De: root@secos-gr8-mail.secos-gr8.lan
Vers: mailuser1@secos-gr8.lan
Resultat: Message recu dans /home/mailuser1/Maildir/new/
```

### Test 2 : Envoi entre utilisateurs (OK)
```
De: mailuser1@secos-gr8.lan
Vers: mailuser2@secos-gr8.lan
Resultat: Message recu dans /home/mailuser2/Maildir/new/
```

### Test 3 : Envoi depuis VM Admin (OK)
```
De: admin@secos-gr8.lan
Depuis: 192.168.100.120 (VM Admin)
Vers: mailuser1@secos-gr8.lan
Resultat: Message recu avec succes
```

### Verification des ports (OK)
```
Port 25   (SMTP)       : Actif
Port 587  (Submission) : Actif
Port 143  (IMAP)       : Actif
Port 993  (IMAPS)      : Actif
Port 110  (POP3)       : Actif
Port 995  (POP3S)      : Actif
```

---

## Commandes Utiles

| Action | Commande |
|--------|----------|
| Status Postfix | `systemctl status postfix` |
| Status Dovecot | `systemctl status dovecot` |
| Voir la queue mail | `mailq` |
| Forcer envoi queue | `postfix flush` |
| Logs mail | `tail -f /var/log/mail.log` |
| Tester config Postfix | `postfix check` |
| Recharger Postfix | `systemctl reload postfix` |
| Recharger Dovecot | `systemctl reload dovecot` |
| Verifier ports | `ss -lntp \| grep -E ":25\|:587\|:143"` |

---

## Securite Implementee

- [x] Reseau limite (mynetworks = LAN 192.168.100.0/24)
- [x] Pas de relay ouvert
- [x] VRFY desactive
- [x] HELO requis
- [x] TLS disponible (certificat auto-signe)
- [x] SSH durci (cles uniquement, pas de root)
- [x] UFW avec ports minimaux
- [x] AppArmor active

---

## Configuration DNS Requise (VM DNS)

Ajouter sur la VM DNS (192.168.100.136) dans `/etc/unbound/unbound.conf.d/secos-gr8.conf` :

```
local-data: "mail.secos-gr8.lan.  IN A 192.168.100.162"
local-data: "secos-gr8.lan.       IN MX 10 mail.secos-gr8.lan."
```

Puis redemarrer :
```bash
systemctl restart unbound
```

---

## Architecture

```
                    SecOS-GR8-Mail (192.168.100.162)
    +----------------------------------------------------------+
    |                                                          |
    |   +-------------+              +-------------+           |
    |   |   Postfix   |              |   Dovecot   |           |
    |   |   (SMTP)    |              | (IMAP/POP3) |           |
    |   |             |              |             |           |
    |   |  Port 25    |              |  Port 143   |           |
    |   |  Port 587   |              |  Port 993   |           |
    |   +------+------+              +------+------+           |
    |          |                            |                  |
    |          +------------+---------------+                  |
    |                       |                                  |
    |                       v                                  |
    |              +------------------+                        |
    |              |  ~/Maildir/      |                        |
    |              |  (Stockage)      |                        |
    |              +------------------+                        |
    |                                                          |
    +----------------------------------------------------------+

    Clients autorises : 192.168.100.0/24 (LAN SecOS-GR8)
```

---

*Documentation generee le 4 janvier 2026*
*VM SecOS-GR8-Mail - Serveur Postfix/Dovecot*
