# SecOS-GR8-Client - Résumé de Configuration

**Date de configuration** : 3 janvier 2026
**Configuré par** : Claude Code

---

## Informations de la VM

| Élément | Valeur |
|---------|--------|
| Hostname | `secos-gr8-client` |
| OS | Debian 12 (Bookworm) |
| IP LAN | `192.168.100.159` |
| IP VPN | `10.10.10.3` (WireGuard) |
| Interface réseau | ens18 |
| GUI | Xfce |

---

## Comptes Utilisateurs

| Compte | Mot de passe | Droits | Usage |
|--------|--------------|--------|-------|
| root | `admin` | root | Administration système |
| testuser | `test` | sudo | Tests et utilisation quotidienne |

---

## Configuration Réseau

### DNS Interne

Le DNS est configuré pour utiliser le serveur interne de l'infrastructure.

**Fichier** : `/etc/resolv.conf`
```
nameserver 192.168.100.136
search secos-gr8.lan
```

**NetworkManager** : configuré pour ne pas modifier resolv.conf
**Fichier** : `/etc/NetworkManager/conf.d/dns.conf`

### Pare-feu (UFW)

| Règle | Description |
|-------|-------------|
| Default incoming | DENY |
| Default outgoing | ALLOW |
| Port 22/TCP | ALLOW (SSH) |

**Commandes utiles** :
```bash
sudo ufw status          # Voir le statut
sudo ufw allow 80/tcp    # Ouvrir un port
sudo ufw disable         # Désactiver temporairement
```

---

## Services de l'Infrastructure

### Tableau des serveurs

| Service | Hostname | IP | Port |
|---------|----------|-----|------|
| Admin | admin.secos-gr8.lan | 192.168.100.120 | 22 (SSH) |
| DNS | dns.secos-gr8.lan | 192.168.100.136 | 53 |
| Auth (LDAP) | auth.secos-gr8.lan | 192.168.100.139 | 389 |
| Data (Samba) | data.secos-gr8.lan | 192.168.100.149 | 445 |
| Backup | backup.secos-gr8.lan | 192.168.100.151 | 22 |
| VPN | vpn.secos-gr8.lan | 192.168.100.153 | 51820 |
| Web | web.secos-gr8.lan | 192.168.100.157 | 443 |
| Client | client.secos-gr8.lan | 192.168.100.159 | 22 |

---

## Outils Installés

| Paquet | Usage | Commande exemple |
|--------|-------|------------------|
| firefox-esr | Navigateur web | `firefox` |
| smbclient | Client Samba | `smbclient -L //data.secos-gr8.lan -U alice` |
| ldap-utils | Outils LDAP | `ldapsearch -x -H ldap://auth.secos-gr8.lan ...` |
| nmap | Scanner réseau | `nmap -sn 192.168.100.0/24` |
| dnsutils | Outils DNS | `dig web.secos-gr8.lan +short` |
| curl/wget | Requêtes HTTP | `curl -k https://web.secos-gr8.lan` |
| wireguard | Client VPN | `sudo wg-quick up wg0` |

---

## Configuration VPN WireGuard

**Fichier** : `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.10.10.3/24
PrivateKey = WMK2dWcChA8uw4nWoK2JOsvnb0R12Jnoa1HsgQqrsGY=
DNS = 192.168.100.136

[Peer]
PublicKey = DGJuY9QJD+1tYLxYZQiv4lSLzf7qJsSrZGPH0dmQgAc=
Endpoint = 192.168.100.153:51820
AllowedIPs = 10.10.10.0/24
PersistentKeepalive = 25
```

**Commandes** :
```bash
sudo wg-quick up wg0      # Démarrer le VPN
sudo wg show              # Voir le statut
ping 10.10.10.1           # Tester la connexion
sudo wg-quick down wg0    # Arrêter le VPN
```

---

## Configuration SSH

**Fichier** : `/etc/ssh/sshd_config.d/hardening.conf`

```
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
X11Forwarding yes
```

---

## Tests de Validation

Tous les tests suivants ont été validés lors de la configuration.

### Test DNS

```bash
# Résolution interne
dig web.secos-gr8.lan +short
# Résultat : 192.168.100.157

# Résolution externe
dig google.com +short
# Résultat : OK
```

### Test Web

```bash
# Test HTTPS
curl -Ik https://web.secos-gr8.lan
# Résultat : HTTP/2 200

# Via Firefox
# URL : https://web.secos-gr8.lan
# Accepter le certificat auto-signé
```

### Test LDAP

```bash
# Lister la structure
ldapsearch -x -LLL -H ldap://auth.secos-gr8.lan -b "dc=secos-gr8,dc=lan" dn

# Chercher l'utilisateur alice
ldapsearch -x -LLL -H ldap://auth.secos-gr8.lan -b "dc=secos-gr8,dc=lan" "(uid=alice)" dn uid
```

### Test Samba

```bash
# Lister les partages (mot de passe alice requis)
smbclient -L //data.secos-gr8.lan -U alice

# Accéder au partage company
smbclient //data.secos-gr8.lan/company -U alice -c "ls"
```

### Test VPN

```bash
sudo wg-quick up wg0
ping -c 3 10.10.10.1
# Résultat : 3 packets transmitted, 3 received, 0% packet loss
sudo wg-quick down wg0
```

---

## Commandes Utiles

```bash
# Voir l'IP
ip a

# Voir le DNS configuré
cat /etc/resolv.conf

# Tester la connectivité
ping -c 2 web.secos-gr8.lan

# Voir les connexions réseau
ss -tnp

# Redémarrer le réseau
sudo systemctl restart NetworkManager

# Scanner le réseau
nmap -sn 192.168.100.0/24

# Voir les ports ouverts d'un serveur
nmap -sT 192.168.100.157
```

---

## Architecture Réseau

```
┌─────────────────────────────────────────────────────────────────┐
│                    Réseau 192.168.100.0/24                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Admin     │    │    DNS      │    │    Auth     │         │
│  │    .120     │    │    .136     │    │    .139     │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │    Data     │    │   Backup    │    │    VPN      │         │
│  │    .149     │    │    .151     │    │    .153     │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐                            │
│  │    Web      │    │   Client    │  ← Cette VM                │
│  │    .157     │    │    .159     │                            │
│  └─────────────┘    └─────────────┘                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

VPN (WireGuard) : 10.10.10.0/24
  - Serveur VPN  : 10.10.10.1
  - Ce client    : 10.10.10.3
```

---

## Checklist Présentation

| Service | Test | Commande |
|---------|------|----------|
| DNS | Résolution interne | `dig web.secos-gr8.lan +short` |
| DNS | Résolution externe | `dig google.com +short` |
| Web | Page HTTPS | Firefox → `https://web.secos-gr8.lan` |
| LDAP | Structure | `ldapsearch -x -H ldap://auth.secos-gr8.lan -b "dc=secos-gr8,dc=lan" dn` |
| LDAP | Utilisateur | `ldapsearch ... "(uid=alice)"` |
| Samba | Partages | `smbclient -L //data.secos-gr8.lan -U alice` |
| VPN | Connexion | `sudo wg-quick up wg0 && ping 10.10.10.1` |
| SSH | Accès serveurs | `ssh sysadmin@admin.secos-gr8.lan` |

---

## Modifications Effectuées

1. **Mise à jour système** : `apt update && apt upgrade`
2. **Hostname** : changé en `secos-gr8-client`
3. **DNS** : configuré pour utiliser 192.168.100.136
4. **NetworkManager** : empêché de modifier resolv.conf
5. **Outils** : firefox-esr, smbclient, ldap-utils, nmap, wireguard, etc.
6. **testuser** : ajouté au groupe sudo
7. **SSH** : durci (PermitRootLogin no)
8. **UFW** : activé avec port 22 ouvert
9. **WireGuard** : configuré avec IP 10.10.10.3
