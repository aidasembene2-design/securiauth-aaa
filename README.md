# securiauth-aaa
# Infrastructure AAA SecuriAuth - Sénégal

Infrastructure complète d'**Authentication, Authorization, Accounting (AAA)** dockerisée pour entreprise sénégalaise.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/docker-required-blue.svg)](https://docker.com)

## 🎯 Vue d'Ensemble

Cette infrastructure fournit une solution AAA professionnelle comprenant:

- **OpenLDAP** - Annuaire centralisé pour gestion utilisateurs/groupes
- **FreeRADIUS** (x2) - Serveurs AAA RADIUS redondants pour VPN/WiFi/802.1X
- **TACACS+** (x2) - Serveurs AAA redondants pour équipements réseau (switches/routeurs)
- **MySQL** - Base de données pour accounting RADIUS
- **OpenVPN** - Serveur VPN avec authentification RADIUS
- **Prometheus + Grafana** - Monitoring et dashboards

## 📋 Caractéristiques

✅ **Architecture Multi-Sites** (Dakar, Thiès, Saint-Louis)  
✅ **Haute disponibilité** (2x RADIUS, 2x TACACS+)  
✅ **Gestion granulaire des privilèges** (admins, opérateurs, support)  
✅ **VLANs dynamiques** via RADIUS  
✅ **Accounting complet** (sessions, commandes, connexions)  
✅ **Dockerisé** pour déploiement rapide  
✅ **Documentation complète** en français

## 🚀 Démarrage Rapide

### Pré-requis

- Ubuntu 22.04 LTS
- Docker & Docker Compose
- 8GB RAM minimum
- 2 vCPUs minimum

### Installation

```bash
# 1. Cloner le repository
cd ~
git clone https://github.com/votre-org/securiauth-aaa.git
cd securiauth-aaa

# 2. Rendre les scripts exécutables
chmod +x scripts/*.sh

# 3. Déployer l'infrastructure
sudo bash scripts/deploy.sh

# 4. Tester l'installation
sudo bash scripts/test-infrastructure.sh
```

**⏱️ Durée totale: ~5 minutes**

## 📦 Structure du Projet

```
securiauth-aaa/
├── docker-compose.yml              # Orchestration Docker
├── README.md                       # Ce fichier
│
├── openldap/                       # OpenLDAP
│   └── ldif/                       # Données LDAP
│       ├── 01-structure.ldif       # OUs et groupes
│       ├── 02-users.ldif           # Utilisateurs
│       └── 03-memberships.ldif     # Associations
│
├── freeradius/                     # FreeRADIUS
│   ├── config/                     # Configurations
│   ├── certs/                      # Certificats EAP-TLS
│   └── sql/                        # Schéma base de données
│       ├── 01-schema.sql           # Tables RADIUS
│       └── 02-data.sql             # Données initiales
│
├── tacacs/                         # TACACS+
│   ├── Dockerfile                  # Image personnalisée
│   └── config/
│       └── tac_plus.conf           # Configuration principale
│
├── scripts/                        # Scripts d'administration
│   ├── deploy.sh                   # Déploiement automatique
│   ├── test-infrastructure.sh      # Tests automatisés
│   ├── backup.sh                   # Sauvegardes
│   └── generate-pki.sh             # Génération certificats
│
├── docs/                           # Documentation
│   └── GUIDE-COMPLET.md            # Guide détaillé (200+ pages)
│
└── monitoring/                     # Prometheus/Grafana
    └── prometheus.yml              # Configuration métriques
```

## 👥 Utilisateurs Pré-configurés

| Utilisateur | Mot de Passe | Rôle | Groupes |
|-------------|--------------|------|---------|
| `dieyna` | `SecurePass2024!` | Admin Système | admins, vpn-dakar |
| `maman` | `SecurePass2024!` | Opératrice Réseau | operateurs, vpn-dakar |
| `aida` | `SecurePass2024!` | Support | support, wifi-entreprise |

**⚠️ IMPORTANT:** Changez tous les mots de passe en production!

## 🌐 Services Accessibles

| Service | URL/Port | Credentials |
|---------|----------|-------------|
| **phpLDAPadmin** | http://localhost:8080 | admin / AdminSecure2024! |
| **Grafana** | http://localhost:3000 | admin / Admin2024! |
| **Prometheus** | http://localhost:9090 | - |
| **RADIUS Auth** | localhost:1812/udp | - |
| **RADIUS Acct** | localhost:1813/udp | - |
| **TACACS+** | localhost:49/tcp | - |
| **MySQL** | localhost:3306 | radiususer / RadiusDB2024! |

## 🧪 Tests

### Test Automatique Complet

```bash
sudo bash scripts/test-infrastructure.sh
```

### Tests Manuels

```bash
# Test LDAP
ldapsearch -x -H ldap://localhost:389 \
  -b "dc=securiauth,dc=com" "(uid=dieyna)"

# Test RADIUS
radtest dieyna SecurePass2024! localhost:1812 0 testing123

# Test MySQL
docker exec securiauth_mysql mysql \
  -u radiususer -pRadiusDB2024! radius \
  -e "SELECT * FROM nas;"
```

## 📖 Documentation

**Guide complet:** [docs/GUIDE-COMPLET.md](docs/GUIDE-COMPLET.md)

Contenu:
- Architecture détaillée
- Configuration pas-à-pas
- Exemples de configuration équipements (Cisco, HP, Mikrotik)
- Procédures de dépannage
- Bonnes pratiques de sécurité
- Maintenance et monitoring

## 🔐 Sécurité

### À faire IMMÉDIATEMENT en production:

1. **Changer tous les mots de passe** (LDAP, MySQL, TACACS+, utilisateurs)
2. **Générer nouveaux certificats** SSL/TLS
3. **Configurer firewall** (iptables/NSG Azure)
4. **Activer LDAPS** (port 636)
5. **Restreindre accès management** (phpLDAPadmin, Grafana)
6. **Configurer sauvegardes** automatiques

Voir: [docs/GUIDE-COMPLET.md#sécurité](docs/GUIDE-COMPLET.md#sécurité)

## 🏗️ Architecture

### Réseau Docker

```
172.20.0.0/16
├── 172.20.0.10   OpenLDAP
├── 172.20.0.11   phpLDAPadmin
├── 172.20.0.20   MySQL
├── 172.20.0.30   FreeRADIUS 1
├── 172.20.0.31   FreeRADIUS 2
├── 172.20.0.40   TACACS+ 1
├── 172.20.0.41   TACACS+ 2
├── 172.20.0.50   OpenVPN
├── 172.20.0.60   Prometheus
└── 172.20.0.61   Grafana
```

### Flux d'Authentification RADIUS

```
Client → NAS → RADIUS → LDAP → Vérification
                  ↓
               MySQL (politiques)
                  ↓
            NAS ← Accept + VLAN
```

### Flux d'Authentification TACACS+

```
Admin → Switch → TACACS+ → LDAP → Vérification
                    ↓
              Switch ← Privilege Level
                    ↓
              Commande → Authorization Check
```

## 🛠️ Commandes Utiles

```bash
# Voir status des conteneurs
docker-compose ps

# Logs en temps réel
docker-compose logs -f

# Redémarrer un service
docker-compose restart freeradius1

# Arrêter tout
docker-compose down

# Backup LDAP
docker exec securiauth_openldap slapcat > backup-ldap.ldif

# Backup MySQL
docker exec securiauth_mysql mysqldump \
  -u root -pRootMySQL2024! radius > backup-radius.sql
```

## 📊 Monitoring

### Métriques Disponibles

- **Sessions actives** (RADIUS)
- **Tentatives d'authentification** (succès/échecs)
- **Accounting** (bande passante, durée sessions)
- **Commandes TACACS+** par utilisateur
- **Ressources système** (CPU, RAM, réseau)

### Grafana Dashboards

Connectez-vous à http://localhost:3000 et créez des dashboards pour:
- Vue d'ensemble infrastructure
- Sessions RADIUS en temps réel
- Top utilisateurs par bande passante
- Historique des authentifications

## 🔧 Configuration Équipements Réseau

### Switch Cisco

```cisco
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa accounting exec default start-stop group tacacs+

tacacs-server host 172.20.0.40 key TacacsKey2024SecuriAuth!
tacacs-server host 172.20.0.41 key TacacsKey2024SecuriAuth!

username admin privilege 15 secret BackupPass2024!
```

### Access Point WiFi

```
SSID: SecuriAuth-Enterprise
Security: WPA2-Enterprise (802.1X)
RADIUS Primary: 172.20.0.30:1812
RADIUS Secondary: 172.20.0.31:1812
RADIUS Secret: VPNSecret2024!
```

Voir guide complet pour Cisco, HP, Mikrotik, etc.

## 🐛 Dépannage

### LDAP ne démarre pas
```bash
docker-compose logs openldap
docker-compose restart openldap
```

### RADIUS ne répond pas
```bash
docker exec -it securiauth_radius1 radiusd -X
docker-compose restart freeradius1 freeradius2
```

### TACACS+ rejette authentification
```bash
docker exec securiauth_tacacs1 tail -f /var/log/tacacs/accounting.log
```

**Plus de solutions:** [docs/GUIDE-COMPLET.md#dépannage](docs/GUIDE-COMPLET.md#dépannage)

## 📞 Support

| Rôle | Contact |
|------|---------|
| **Admin Principal** | Dieyna Ndiaye - dieyna.ndiaye@securiauth.com |
| **Admin Réseau** | Moussa Fall - moussa.fall@securiauth.com |
| **Support 24/7** | support@securiauth.com |

## 📝 Roadmap

- [ ] Interface web d'administration (phpRADIUSadmin)
- [ ] Integration avec AD/Azure AD
- [ ] Multi-factor authentication (MFA)
- [ ] Certificats EAP-TLS automatisés
- [ ] API REST pour gestion utilisateurs
- [ ] Dashboard mobile

## 🤝 Contribution

Les contributions sont bienvenues! Merci de:

1. Forker le projet
2. Créer une branche (`git checkout -b feature/AmazingFeature`)
3. Commiter vos changements (`git commit -m 'Add AmazingFeature'`)
4. Pusher vers la branche (`git push origin feature/AmazingFeature`)
5. Ouvrir une Pull Request

## 📄 Licence

Ce projet est sous licence MIT. Voir [LICENSE](LICENSE) pour plus de détails.

## 🙏 Remerciements

- OpenLDAP Foundation
- FreeRADIUS Project
- Shrubbery Networks (TACACS+)
- Communauté Docker

---

**Développé avec ❤️ au Sénégal pour SecuriAuth**

*Pour toute question, consultez d'abord le [Guide Complet](docs/GUIDE-COMPLET.md)*
