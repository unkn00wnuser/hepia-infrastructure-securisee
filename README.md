# Infrastructure Serveur Securisee - Projet SecOS

Conception et deploiement d'une infrastructure serveur complete et securisee pour une organisation. Le projet couvre l'ensemble des services essentiels : DNS, messagerie, VPN, serveur web, authentification centralisee, sauvegarde, journalisation et administration.

**Contexte** : Projet realise dans le cadre du cours de Systemes d'Exploitation Securises a HEPIA Geneve, filiere Informatique et Systemes de Communication (orientation Securite Informatique), janvier 2025.

---

## Technologies et outils utilises

![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![DNS](https://img.shields.io/badge/BIND9-DNS-blue?style=flat)
![VPN](https://img.shields.io/badge/OpenVPN-EA7E20?style=flat&logo=openvpn&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-D22128?style=flat&logo=apache&logoColor=white)
![LDAP](https://img.shields.io/badge/OpenLDAP-2C2D72?style=flat)

- **DNS** : BIND9 (zones directes/inverses, DNSSEC)
- **Messagerie** : Postfix + Dovecot (SMTP/IMAP securise)
- **VPN** : OpenVPN (acces distant securise)
- **Web** : Apache (HTTPS, virtual hosts)
- **Authentification** : OpenLDAP (annuaire centralise)
- **Sauvegarde** : Scripts automatises, politique de retention
- **Journalisation** : Rsyslog, logrotate, centralisation des logs
- **Administration** : Gestion des utilisateurs, sudo, SSH securise

## Services deployes

| Service | Technologie | Description |
|---------|-------------|-------------|
| DNS | BIND9 | Resolution de noms interne et externe |
| Mail | Postfix + Dovecot | Messagerie complete SMTP/IMAP |
| VPN | OpenVPN | Tunnel securise pour acces distant |
| Web | Apache2 | Hebergement HTTPS avec certificats |
| Auth | OpenLDAP | Authentification centralisee |
| Backup | Scripts Bash | Sauvegarde automatisee et incrementale |
| Logs | Rsyslog | Centralisation et rotation des journaux |
| Admin | SSH + sudo | Administration securisee |

## Competences demontrees

- Administration systeme Linux avancee
- Deploiement et securisation de services reseau
- Configuration DNS (zones, DNSSEC)
- Mise en place d'une PKI et de certificats TLS
- Configuration VPN site-to-site et client-to-site
- Gestion centralisee des identites (LDAP)
- Politique de sauvegarde et de reprise d'activite
- Durcissement de serveurs (hardening)

## Structure du repo

```
hepia-infrastructure-securisee/
├── docs/                                       # Documentation detaillee par service
│   ├── SecOS-GR8-Admin_Rapport.md
│   ├── SecOS-GR8-Auth_Rapport.md
│   ├── SecOs-GR8-Backup_Rapport.md
│   ├── SecOS-GR8-Client_Resume.md
│   ├── SecOs-GR8-Data_Rapport.md
│   ├── SecOS-GR8-DNS_Rapport.md
│   ├── SecOS-GR8-Logs_Rapport.md
│   ├── SecOs-GR8-Mail_Rapport.md
│   ├── SecOs-GR8-VPN_Rapport.md
│   ├── SecOs-GR8-WEB_Rapport.md
│   └── secos-gr8.conf
├── rapports/
│   ├── SecOS-GR8-Documentation-Technique.pdf
│   └── SecOS-GR8-Rapport-Synthese.pdf
└── README.md
```

## Auteur

**Riad Hyseni** (Groupe GR8)
HEPIA Geneve - Securite Informatique, janvier 2025
