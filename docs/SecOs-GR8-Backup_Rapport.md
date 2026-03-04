# SecOS-GR8-Backup - Rapport de configuration

## Informations système

| Élément | Valeur |
|---------|--------|
| Hostname | secos-gr8-backup |
| IP | 192.168.100.151/24 |
| OS | Debian 12 |
| Date config | 2026-01-03 |

---

## Services actifs

| Service | Statut |
|---------|--------|
| SSH | ✅ Actif (port 22) |
| rsnapshot | ✅ v1.4.5-1 installé |
| UFW | ✅ Actif (22/tcp autorisé) |

---

## Durcissement SSH

Fichier: `/etc/ssh/sshd_config.d/hardening.conf`

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

---

## Structure des dossiers de backup

```
/srv/backup/
├── admin/
├── auth/
├── data/
│   └── company/
├── dns/
├── mail/
├── rsnapshot/   ← Backups rsnapshot
│   ├── daily.0/ (le plus récent)
│   ├── daily.1/
│   ├── ...
│   └── daily.6/
└── web/
```

---

## Configuration rsnapshot

Fichier: `/etc/rsnapshot.conf`

```
config_version  1.2
snapshot_root   /srv/backup/rsnapshot/

# Commandes
cmd_cp          /bin/cp
cmd_rm          /bin/rm
cmd_rsync       /usr/bin/rsync
cmd_ssh         /usr/bin/ssh
cmd_logger      /usr/bin/logger

# Rétention
retain  daily   7
retain  weekly  4
retain  monthly 3

# Logs
verbose   2
loglevel  3
logfile   /var/log/rsnapshot.log
lockfile  /var/run/rsnapshot.pid

# Options rsync
rsync_short_args  -aH
rsync_long_args   --delete --numeric-ids --relative --delete-excluded
ssh_args          -i /root/.ssh/id_backup

# Sources de backup
backup  /etc/                                  localhost/
backup  /home/                                 localhost/
backup  sysadmin@192.168.100.149:/srv/samba/   data/
```

---

## Planification cron

Fichier: `/etc/cron.d/rsnapshot`

```
# daily : tous les jours à 2h00
0 2 * * *       root    /usr/bin/rsnapshot daily

# weekly : tous les dimanches à 3h00
0 3 * * 0       root    /usr/bin/rsnapshot weekly

# monthly : le 1er du mois à 4h00
0 4 1 * *       root    /usr/bin/rsnapshot monthly
```

---

## Clés SSH

### Clé de backup (pour se connecter aux VMs sources)

Fichiers:
- Clé privée: `/root/.ssh/id_backup` et `/home/sysadmin/.ssh/id_backup`
- Clé publique: `/home/sysadmin/.ssh/id_backup.pub`

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJZai3hz6vQgC28q/cmPf2yK0SO9gK9iSifF7VOPttnQ backup@secos-gr8-backup
```

Fingerprint: `SHA256:7fkiaDz5WHGbWjXgso6AJrtuKvHtkIocizjghVPsx+Y`

### Clés autorisées sur backup

Fichier: `/home/sysadmin/.ssh/authorized_keys`

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGlpQNm1UioMPHWDF8KfMlqSonODQrFyZx14A8ghnHe3 root@secos-gr8-admin
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH/wzuzqMwgch74vXrxB/xDDe1pF1yEGJryi+FOFfUHB root@secos-gr8-data
```

### Config SSH sysadmin

Fichier: `/home/sysadmin/.ssh/config`

```
Host data
    HostName 192.168.100.149
    User sysadmin
    IdentityFile ~/.ssh/id_backup
    StrictHostKeyChecking accept-new

Host admin
    HostName 192.168.100.120
    User sysadmin
    IdentityFile ~/.ssh/id_backup
    StrictHostKeyChecking accept-new

Host dns
    HostName 192.168.100.136
    User sysadmin
    IdentityFile ~/.ssh/id_backup
    StrictHostKeyChecking accept-new

Host auth
    HostName 192.168.100.139
    User sysadmin
    IdentityFile ~/.ssh/id_backup
    StrictHostKeyChecking accept-new
```

---

## VMs du réseau

| VM | IP | Rôle | Backup |
|----|-----|------|--------|
| secos-gr8-backup | 192.168.100.151 | Serveur backup | - |
| secos-gr8-data | 192.168.100.149 | Serveur Samba | ✅ Configuré |
| secos-gr8-admin | 192.168.100.120 | Admin | ⏳ À configurer |
| secos-gr8-dns | 192.168.100.136 | DNS | ⏳ À configurer |
| secos-gr8-auth | 192.168.100.139 | Auth | ⏳ À configurer |

---

## État des backups

### VM Data (192.168.100.149)

- **Source**: `/srv/samba/`
- **Destination**: `/srv/backup/rsnapshot/daily.X/data/`
- **Statut**: ✅ Fonctionnel
- **Contenu sauvegardé**:
  ```
  /srv/samba/company/
  ├── test_admin/
  ├── testdir/
  └── test.txt
  ```

### Prérequis sur les VMs sources

Pour que le backup fonctionne, chaque VM source doit:
1. Avoir la clé publique de backup dans `/home/sysadmin/.ssh/authorized_keys`:
   ```
   ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJZai3hz6vQgC28q/cmPf2yK0SO9gK9iSifF7VOPttnQ backup@secos-gr8-backup
   ```
2. Donner les permissions de lecture à sysadmin sur les dossiers à sauvegarder

---

## Commandes utiles

```bash
# Tester la config rsnapshot
rsnapshot configtest

# Lancer un backup manuel
rsnapshot -v daily

# Voir les logs
tail -f /var/log/rsnapshot.log

# Tester la connexion SSH vers data
ssh -i /root/.ssh/id_backup sysadmin@192.168.100.149 "hostname"

# Voir l'espace disque des backups
du -sh /srv/backup/rsnapshot/*
```

---

## Problèmes résolus

1. **Host key verification failed**: Ajout de la clé host de data dans `/root/.ssh/known_hosts`
2. **Permission denied (publickey)**: Correction de la clé publique sur data (caractère manquant)
3. **Permission denied sur /srv/samba/company**: Ajout de sysadmin au groupe avec accès au dossier

---

*Généré le 2026-01-03 par Claude Code*
