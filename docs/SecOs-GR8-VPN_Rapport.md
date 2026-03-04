# Rapport de Configuration - SecOS-GR8-VPN (WireGuard)

## Informations Générales

| Élément | Valeur |
|---------|--------|
| VM | SecOS-GR8-VPN |
| Hostname | secos-gr8-vpn |
| Rôle | Serveur VPN (WireGuard) |
| OS | Debian 12 (Bookworm) |
| Date de configuration | 2026-01-03 |

---

## Configuration Réseau

### Interfaces

| Interface | IP | Rôle |
|-----------|-----|------|
| ens18 | 192.168.100.153/24 | Interface LAN principale |
| wg0 | 10.10.10.1/24 | Interface VPN WireGuard |

### Ports Ouverts (UFW)

| Port | Protocole | Service |
|------|-----------|---------|
| 22 | TCP | SSH |
| 51820 | UDP | WireGuard VPN |

---

## Clés WireGuard

### Serveur

| Type | Valeur |
|------|--------|
| Clé privée | `eD2w8MwuoU1sBci13gSD8pgmIhHMZ4P9ndzTvghEl1w=` |
| Clé publique | `DGJuY9QJD+1tYLxYZQiv4lSLzf7qJsSrZGPH0dmQgAc=` |

### Client Admin (10.10.10.2)

| Type | Valeur |
|------|--------|
| Clé privée | `OHaH9xMliHfhFCSz/1foUQHejjA9D3UNFor4xy3Ciks=` |
| Clé publique | `PBpcjsNs8SpWoah7jFmMN2qcYJRke9QFnanFCdUStUg=` |

### Client User1 (10.10.10.3)

| Type | Valeur |
|------|--------|
| Clé privée | `WMK2dWcChA8uw4nWoK2JOsvnb0R12Jnoa1HsgQqrsGY=` |
| Clé publique | `Ttot+HGwa5z9TwMdWLLgHpo17B4CG9r8lrtOcIcjUSY=` |

---

## Attribution des IPs VPN

| Client | IP VPN | Description |
|--------|--------|-------------|
| Serveur | 10.10.10.1 | Serveur WireGuard |
| Admin | 10.10.10.2 | Client administrateur |
| User1 | 10.10.10.3 | Client utilisateur 1 |
| (réservé) | 10.10.10.4-254 | Futurs clients |

---

## Fichiers de Configuration

### Configuration Serveur - `/etc/wireguard/wg0.conf`

```ini
[Interface]
# Adresse VPN du serveur
Address = 10.10.10.1/24

# Port d'écoute
ListenPort = 51820

# Clé privée du serveur
PrivateKey = eD2w8MwuoU1sBci13gSD8pgmIhHMZ4P9ndzTvghEl1w=

# Règles iptables pour le NAT
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE

# ========== CLIENTS ==========

# Client 1 : Admin
[Peer]
# Clé publique du client admin
PublicKey = PBpcjsNs8SpWoah7jFmMN2qcYJRke9QFnanFCdUStUg=
# IP VPN attribuée au client
AllowedIPs = 10.10.10.2/32

# Client 2 : User1
[Peer]
# Clé publique du client user1
PublicKey = Ttot+HGwa5z9TwMdWLLgHpo17B4CG9r8lrtOcIcjUSY=
# IP VPN attribuée au client
AllowedIPs = 10.10.10.3/32
```

### Configuration Client Admin - `/etc/wireguard/client_admin.conf`

```ini
[Interface]
# Adresse VPN du client
Address = 10.10.10.2/24

# Clé privée du client
PrivateKey = OHaH9xMliHfhFCSz/1foUQHejjA9D3UNFor4xy3Ciks=

# DNS (optionnel - utilise le DNS interne)
DNS = 192.168.100.136

[Peer]
# Clé publique du serveur
PublicKey = DGJuY9QJD+1tYLxYZQiv4lSLzf7qJsSrZGPH0dmQgAc=

# IP du serveur VPN et port
Endpoint = 192.168.100.153:51820

# Trafic à router via le VPN (tout le réseau interne)
AllowedIPs = 10.10.10.0/24, 192.168.100.0/24

# Keepalive pour maintenir la connexion
PersistentKeepalive = 25
```

### Configuration Client User1 - `/etc/wireguard/client_user1.conf`

```ini
[Interface]
Address = 10.10.10.3/24
PrivateKey = WMK2dWcChA8uw4nWoK2JOsvnb0R12Jnoa1HsgQqrsGY=
DNS = 192.168.100.136

[Peer]
PublicKey = DGJuY9QJD+1tYLxYZQiv4lSLzf7qJsSrZGPH0dmQgAc=
Endpoint = 192.168.100.153:51820
AllowedIPs = 10.10.10.0/24, 192.168.100.0/24
PersistentKeepalive = 25
```

---

## Durcissement SSH - `/etc/ssh/sshd_config.d/hardening.conf`

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowAgentForwarding no
```

### Clé SSH autorisée pour sysadmin

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
```

---

## IP Forwarding - `/etc/sysctl.d/99-wireguard.conf`

```
# Enable IP forwarding for WireGuard VPN
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

---

## Comptes Utilisateurs

| Compte | Type | Usage |
|--------|------|-------|
| sysadmin | Linux + sudo | Administration (clé SSH uniquement) |

---

## Étapes de Configuration Effectuées

### Étape 0 : Identification de l'IP
```bash
ip a | grep inet
# Résultat : 192.168.100.153 sur ens18
```

### Étape 1 : Base système + hostname
```bash
apt update && apt -y upgrade
apt -y install vim curl ca-certificates sudo
hostnamectl set-hostname secos-gr8-vpn
```

### Étape 2 : Installation WireGuard et outils
```bash
apt -y install wireguard wireguard-tools openssh-server ufw apparmor apparmor-utils qrencode
systemctl enable --now ssh
systemctl enable --now apparmor
```

### Étape 3 : Durcissement SSH
```bash
# Vérification/création utilisateur sysadmin
id sysadmin || useradd -m -s /bin/bash sysadmin
usermod -aG sudo sysadmin

# Configuration SSH durcie
cat > /etc/ssh/sshd_config.d/hardening.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowAgentForwarding no
EOF

# Clé SSH pour sysadmin
mkdir -p /home/sysadmin/.ssh
chmod 700 /home/sysadmin/.ssh
cat > /home/sysadmin/.ssh/authorized_keys <<'EOF'
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
EOF
chmod 600 /home/sysadmin/.ssh/authorized_keys
chown -R sysadmin:sysadmin /home/sysadmin/.ssh

sshd -t && systemctl restart ssh
```

### Étape 4 : Activer le routage IP
```bash
cat > /etc/sysctl.d/99-wireguard.conf <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sysctl -p /etc/sysctl.d/99-wireguard.conf
```

### Étape 5 : Génération des clés serveur
```bash
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

### Étape 6 : Génération des clés clients
```bash
cd /etc/wireguard
wg genkey | tee client_admin_private.key | wg pubkey > client_admin_public.key
wg genkey | tee client_user1_private.key | wg pubkey > client_user1_public.key
```

### Étape 7 : Configuration du serveur WireGuard
```bash
# Création de /etc/wireguard/wg0.conf avec les vraies clés
chmod 600 /etc/wireguard/*.key /etc/wireguard/wg0.conf
```

### Étape 8 : Démarrage de WireGuard
```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

### Étape 9 : Configuration du pare-feu UFW
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 51820/udp
ufw --force enable
```

### Étape 10 : Création des configurations clients
```bash
# Création de /etc/wireguard/client_admin.conf
# Création de /etc/wireguard/client_user1.conf
chmod 600 /etc/wireguard/client_*.conf

# Génération des QR codes
qrencode -t ansiutf8 < /etc/wireguard/client_admin.conf
qrencode -t ansiutf8 < /etc/wireguard/client_user1.conf
```

---

## Fichiers Importants

| Fichier | Description |
|---------|-------------|
| /etc/wireguard/wg0.conf | Configuration serveur WireGuard |
| /etc/wireguard/server_private.key | Clé privée serveur |
| /etc/wireguard/server_public.key | Clé publique serveur |
| /etc/wireguard/client_admin_private.key | Clé privée client admin |
| /etc/wireguard/client_admin_public.key | Clé publique client admin |
| /etc/wireguard/client_user1_private.key | Clé privée client user1 |
| /etc/wireguard/client_user1_public.key | Clé publique client user1 |
| /etc/wireguard/client_admin.conf | Config client Admin |
| /etc/wireguard/client_user1.conf | Config client User1 |
| /etc/sysctl.d/99-wireguard.conf | IP forwarding |
| /etc/ssh/sshd_config.d/hardening.conf | Durcissement SSH |

---

## Commandes Utiles

```bash
# Voir l'état de WireGuard
wg show

# Redémarrer WireGuard
systemctl restart wg-quick@wg0

# Voir les logs
journalctl -u wg-quick@wg0 -f

# Ajouter un nouveau client (sur le serveur)
wg set wg0 peer <PUBLIC_KEY> allowed-ips 10.10.10.X/32

# Afficher la config actuelle
wg showconf wg0

# Générer QR code pour un client
qrencode -t ansiutf8 < /etc/wireguard/client_XXX.conf

# Vérifier le statut UFW
ufw status verbose

# Vérifier IP forwarding
sysctl net.ipv4.ip_forward
```

---

## Procédure pour Ajouter un Nouveau Client

1. Générer les clés :
```bash
cd /etc/wireguard
wg genkey | tee client_NEW_private.key | wg pubkey > client_NEW_public.key
```

2. Ajouter le peer dans `/etc/wireguard/wg0.conf` :
```ini
[Peer]
PublicKey = <contenu de client_NEW_public.key>
AllowedIPs = 10.10.10.X/32
```

3. Redémarrer WireGuard :
```bash
systemctl restart wg-quick@wg0
```

4. Créer le fichier de config client :
```ini
[Interface]
Address = 10.10.10.X/24
PrivateKey = <contenu de client_NEW_private.key>
DNS = 192.168.100.136

[Peer]
PublicKey = DGJuY9QJD+1tYLxYZQiv4lSLzf7qJsSrZGPH0dmQgAc=
Endpoint = 192.168.100.153:51820
AllowedIPs = 10.10.10.0/24, 192.168.100.0/24
PersistentKeepalive = 25
```

---

## Note : Configuration DNS (Étape 11)

Sur la VM DNS (192.168.100.136), ajouter dans `/etc/unbound/unbound.conf.d/secos-gr8.conf` :

```yaml
local-data: "vpn.secos-gr8.lan.  IN A 192.168.100.153"
```

Puis redémarrer Unbound :
```bash
systemctl restart unbound
```

---

## Architecture VPN

```
Internet/Externe
       |
       v
[SecOS-GR8-VPN] 192.168.100.153:51820
       |
       | wg0 (10.10.10.1/24)
       |
   +---+---+
   |       |
Client1  Client2
10.10.10.2  10.10.10.3
(Admin)   (User1)
```
