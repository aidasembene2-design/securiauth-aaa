# Guide Complet - Infrastructure AAA SecuriAuth Sénégal

**Version:** 1.0  
**Date:** 2025  
**Organisation:** SecuriAuth Sénégal  
**Domaine:** securiauth.com

---

## 📋 Table des Matières

1. [Vue d'ensemble](#vue-densemble)
2. [Architecture](#architecture)
3. [Prérequis](#prérequis)
4. [Installation](#installation)
5. [Configuration](#configuration)
6. [Utilisation](#utilisation)
7. [Tests](#tests)
8. [Maintenance](#maintenance)
9. [Dépannage](#dépannage)
10. [Sécurité](#sécurité)

---

## 🎯 Vue d'ensemble

### Objectif

Infrastructure AAA (Authentication, Authorization, Accounting) complète pour organisations sénégalaises, comprenant:

- **OpenLDAP**: Annuaire centralisé d'identités (6 utilisateurs, 6 groupes)
- **FreeRADIUS**: Serveur AAA pour VPN et Wi-Fi (2 instances redondantes)
- **TACACS+**: Serveur AAA pour équipements réseau (2 instances redondantes)
- **PKI**: Autorité de certification interne pour EAP-TLS
- **MySQL**: Base de données accounting RADIUS

### Cas d'usage

1. **Authentification VPN** via RADIUS avec certificats EAP-TLS
2. **Authentification équipements réseau** via TACACS+ (switches, routeurs)
3. **Mapping automatique** groupes LDAP → VLANs / privilèges
4. **Accounting centralisé** de toutes les connexions
5. **Haute disponibilité** avec failover automatique

---

## 🏗️ Architecture

### Schéma Global

```
┌─────────────────────────────────────────────────────────────┐
│                   Infrastructure SecuriAuth                  │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌───────────┐         ┌──────────────┐                     │
│  │ OpenLDAP  │◄────────┤ phpLDAPadmin │  (Web:8080)         │
│  │   :389    │         └──────────────┘                     │
│  └─────┬─────┘                                               │
│        │                                                     │
│        ├──────────┬─────────────┬────────────┐              │
│        │          │             │            │              │
│  ┌─────▼─────┐ ┌──▼──────┐ ┌───▼───────┐ ┌──▼──────┐      │
│  │ RADIUS 1  │ │ RADIUS 2│ │ TACACS+ 1 │ │TACACS+ 2│      │
│  │:1812/1813 │ │:1912/913│ │   :49     │ │  :50    │      │
│  └─────┬─────┘ └──┬──────┘ └───────────┘ └─────────┘      │
│        │          │                                         │
│        └──────────┴───────► MySQL :3306                     │
│                              (radacct)                       │
└─────────────────────────────────────────────────────────────┘
           │                        │
           ▼                        ▼
    [Clients VPN]          [Équipements réseau]
    - OpenVPN              - Switches Cisco/HP
    - strongSwan           - Routeurs
    - WPA2-Enterprise      - Firewalls
```

### Réseau Docker

- **Subnet**: 172.25.0.0/16
- **RADIUS 1**: 172.25.0.10
- **RADIUS 2**: 172.25.0.11
- **TACACS+ 1**: 172.25.0.20
- **TACACS+ 2**: 172.25.0.21

---

## 📁 Structure des Répertoires

```
securiauth-aaa/
├── docker-compose.yml          # Orchestration Docker
├── openldap/
│   ├── data/                   # Données LDAP persistantes
│   ├── config/                 # Configuration slapd
│   └── ldif/                   # Fichiers d'import LDIF
│       ├── 01-base-structure.ldif
│       ├── 02-groupes.ldif
│       └── 03-utilisateurs.ldif
├── freeradius/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── config/
│       ├── radius-schema.sql   # Schéma MySQL
│       ├── clients.conf        # Clients RADIUS autorisés
│       ├── radiusd.conf        # Configuration principale
│       └── mods-available/
│           ├── ldap            # Module LDAP
│           └── sql             # Module MySQL
├── tacacs/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── config/
│       └── tac_plus.conf       # Configuration TACACS+
├── pki/
│   ├── ca/                     # Certificats CA
│   │   ├── ca-cert.pem
│   │   └── ca-key.pem
│   ├── server/                 # Certificats serveurs RADIUS
│   │   ├── radius1-cert.pem
│   │   ├── radius1-key.pem
│   │   ├── radius2-cert.pem
│   │   └── radius2-key.pem
│   └── client/                 # Certificats utilisateurs
│       ├── dieyna.diop.p12
│       ├── maman.seck.p12
│       └── ...
├── scripts/
│   ├── generate-pki.sh         # Génération PKI
│   ├── deploy.sh               # Déploiement
│   └── test-auth.sh            # Tests authentification
└── docs/
    ├── GUIDE-COMPLET-FR.md     # Ce document
    └── ARCHITECTURE.md
```

---

## ✅ Prérequis

### Matériel

- **Machine**: Azure Standard B2ms (ou équivalent)
  - 2 vCPUs
  - 8 GB RAM
  - 120 GB disque
- **OS**: Ubuntu 22.04 LTS

### Logiciels

```bash
# Docker Engine 28.5+
# Docker Compose Plugin
# OpenSSL
# ldap-utils
```

### Réseau

- **Ports requis**:
  - 389/636: OpenLDAP
  - 8080: phpLDAPadmin
  - 1812-1813/udp: RADIUS 1
  - 1912-1913/udp: RADIUS 2
  - 49/tcp: TACACS+ 1
  - 50/tcp: TACACS+ 2
  - 3306: MySQL

---

## 🚀 Installation

### Étape 1: Cloner / Préparer l'environnement

```bash
# Le projet est déjà créé dans ~/securiauth-aaa
cd ~/securiauth-aaa

# Vérifier la structure
ls -la
```

### Étape 2: Générer les certificats PKI

```bash
# Exécuter le script de génération PKI
./scripts/generate-pki.sh

# Vérifier les certificats
ls -la pki/ca/
ls -la pki/server/
ls -la pki/client/
```

**Résultat attendu:**
- CA Root: `pki/ca/ca-cert.pem` (valide 10 ans)
- Serveurs RADIUS: `pki/server/radius{1,2}-{cert,key}.pem` (valide 2 ans)
- Clients: `pki/client/*.p12` (valide 1 an, password: `SecuriAuth2025`)

### Étape 3: Démarrer l'infrastructure

```bash
# Démarrer tous les services
sudo docker compose up -d

# Vérifier les conteneurs
sudo docker ps

# Voir les logs
sudo docker compose logs -f
```

### Étape 4: Charger les données LDAP

```bash
# Créer les OUs
ldapadd -x -H ldap://localhost:389 \
  -D "cn=admin,dc=securiauth,dc=com" \
  -w "AdminSecur1Auth2025!" \
  -f openldap/ldif/01-base-structure.ldif

# Charger utilisateurs
ldapadd -x -H ldap://localhost:389 \
  -D "cn=admin,dc=securiauth,dc=com" \
  -w "AdminSecur1Auth2025!" \
  -f openldap/ldif/03-utilisateurs.ldif

# Charger groupes
ldapadd -x -H ldap://localhost:389 \
  -D "cn=admin,dc=securiauth,dc=com" \
  -w "AdminSecur1Auth2025!" \
  -f openldap/ldif/02-groupes.ldif
```

### Étape 5: Vérification

```bash
# Test LDAP
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=securiauth,dc=com" \
  -w "AdminSecur1Auth2025!" \
  -b "dc=securiauth,dc=com" "(uid=dieyna.diop)"

# Accès web phpLDAPadmin
# http://<IP-SERVEUR>:8080
# Login: cn=admin,dc=securiauth,dc=com
# Password: AdminSecur1Auth2025!
```

---

## ⚙️ Configuration

### Utilisateurs LDAP

| Nom complet     | UID            | Groupes                    | Ville       |
|-----------------|----------------|----------------------------|-------------|
| Dieyna Diop     | dieyna.diop    | Administrateurs, Equipements-Admin | Dakar |
| Maman Seck      | maman.seck     | Techniciens                | Dakar       |
| Aida Fall       | aida.fall      | Support, VPN-Utilisateurs  | Thiès       |
| Ousmane Diallo  | ousmane.diallo | Techniciens                | Saint-Louis |
| Fatou Sarr      | fatou.sarr     | Support                    | Dakar       |
| Moussa Ndiaye   | moussa.ndiaye  | Invités                    | Dakar       |

**Note:** Mot de passe par défaut (hash SSHA dans LDIF, à changer en production)

### Groupes et Politiques

#### RADIUS (VPN/Wi-Fi)

| Groupe           | VLAN ID | Description                      |
|------------------|---------|----------------------------------|
| Administrateurs  | 10      | Accès complet réseau admin       |
| Techniciens      | 20      | Réseau technique                 |
| Support          | 30      | Réseau support                   |
| Invités          | 99      | Réseau invités (isolé)           |

#### TACACS+ (Équipements réseau)

| Groupe           | Privilege | Commandes autorisées            |
|------------------|-----------|----------------------------------|
| Administrateurs  | 15        | Toutes (enable, configure, etc.) |
| Equipements-Admin| 15        | Toutes                           |
| Techniciens      | 7         | show, ping, configure interface  |
| Support          | 1         | show, ping, traceroute           |

### Clients RADIUS

Configuration dans `radius-schema.sql`:

```sql
INSERT INTO nas (nasname, shortname, type, secret) VALUES
('172.25.0.30', 'vpn-server', 'other', 'VPNSecuriAuth2025!'),
('172.25.0.40', 'switch-dakar-01', 'cisco', 'SwitchSecuriAuth2025!');
```

---

## 🔧 Utilisation

### Authentification VPN avec EAP-TLS

#### Configuration client OpenVPN

```
client
dev tun
proto udp
remote <SERVEUR-IP> 1194
ca pki/ca/ca-cert.pem
cert pki/client/dieyna.diop-cert.pem
key pki/client/dieyna.diop-key.pem
auth-user-pass-verify "via RADIUS" via-env
plugin /usr/lib/openvpn/plugins/radiusplugin.so
```

#### Configuration strongSwan (IKEv2)

```
# /etc/ipsec.conf
conn securiauth-vpn
  keyexchange=ikev2
  leftauth=eap-tls
  leftcert=dieyna.diop-cert.pem
  rightauth=pubkey
  right=<SERVEUR-IP>
  rightid=%<SERVEUR-FQDN>
  auto=start
```

### Authentification équipement réseau (TACACS+)

#### Configuration switch Cisco

```cisco
! Configuration globale TACACS+
aaa new-model
tacacs-server host 172.25.0.20 key TacacsSecuriAuth2025!
tacacs-server host 172.25.0.21 key TacacsSecuriAuth2025!

! Authentification ligne vty (SSH/Telnet)
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local
aaa authorization commands 15 default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+

! Ligne VTY
line vty 0 15
 transport input ssh
 login authentication default
 authorization exec default
 authorization commands 15 default
 accounting exec default
```

#### Configuration switch HP/Aruba

```
tacacs-server host 172.25.0.20 key TacacsSecuriAuth2025!
tacacs-server host 172.25.0.21 key TacacsSecuriAuth2025!
aaa authentication login privilege-mode
aaa authentication ssh login tacacs local
aaa authorization commands tacacs
aaa accounting exec start-stop tacacs
aaa accounting commands stop-only tacacs
```

#### Test connexion SSH

```bash
# Connexion en tant que Dieyna (Administrateur)
ssh dieyna.diop@172.25.0.40

# Une fois connecté, vérifier privilège
Switch> enable
Switch# show privilege
Current privilege level is 15

# Connexion en tant que Aida (Support)
ssh aida.fall@172.25.0.40

Switch> enable
Switch# show privilege
Current privilege level is 1
Switch# configure terminal
% Command authorization failed  # Normal, pas autorisée
```

---

## 🧪 Tests

### Test 1: Authentification LDAP directe

```bash
# Test bind utilisateur
ldapwhoami -x -H ldap://localhost:389 \
  -D "cn=Dieyna Diop,ou=Utilisateurs,dc=securiauth,dc=com" \
  -w "<mot-de-passe>"

# Recherche appartenance groupes
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=securiauth,dc=com" \
  -w "AdminSecur1Auth2025!" \
  -b "ou=Groupes,dc=securiauth,dc=com" \
  "(member=cn=Dieyna Diop,ou=Utilisateurs,dc=securiauth,dc=com)"
```

### Test 2: RADIUS avec radtest

```bash
# Test PAP (simple)
radtest dieyna.diop <password> localhost:1812 0 testing123

# Test EAP-TLS
eapol_test -c dieyna-eap-tls.conf -s testing123
```

### Test 3: TACACS+ avec tcpdump

```bash
# Monitorer requêtes TACACS+
sudo tcpdump -i any port 49 -vv

# Dans autre terminal, tester connexion SSH vers équipement
ssh dieyna.diop@<SWITCH-IP>
```

### Test 4: Accounting RADIUS

```bash
# Consulter table MySQL radacct
sudo docker exec -it securiauth-mysql mysql -uradiususer -pRadiusDB2025! radius

mysql> SELECT username, nasipaddress, acctstarttime, acctsessiontime 
       FROM radacct 
       ORDER BY acctstarttime DESC 
       LIMIT 10;
```

### Test 5: Haute disponibilité

```bash
# Arrêter RADIUS 1
sudo docker stop securiauth-radius1

# Tester auth sur RADIUS 2
radtest dieyna.diop <password> localhost:1912 0 testing123

# Redémarrer RADIUS 1
sudo docker start securiauth-radius1
```

---

## 🔄 Maintenance

### Sauvegarde

```bash
#!/bin/bash
# Sauvegarde quotidienne

BACKUP_DIR="/backup/securiauth-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# LDAP
ldapsearch -x -H ldap://localhost:389 \
  -D "cn=admin,dc=securiauth,dc=com" \
  -w "AdminSecur1Auth2025!" \
  -b "dc=securiauth,dc=com" > $BACKUP_DIR/ldap-backup.ldif

# MySQL
sudo docker exec securiauth-mysql mysqldump \
  -uradiususer -pRadiusDB2025! radius > $BACKUP_DIR/radius-db.sql

# Configurations
tar -czf $BACKUP_DIR/configs.tar.gz \
  ~/securiauth-aaa/freeradius/config \
  ~/securiauth-aaa/tacacs/config

# PKI (CA uniquement, pas les clés privées clients)
cp ~/securiauth-aaa/pki/ca/ca-cert.pem $BACKUP_DIR/
```

### Logs

```bash
# Logs temps réel
sudo docker compose logs -f

# Logs RADIUS
sudo docker exec securiauth-radius1 tail -f /var/log/freeradius/radius.log

# Logs TACACS+
sudo docker exec securiauth-tacacs1 tail -f /var/log/tacacs/accounting.log

# Logs OpenLDAP
sudo docker logs securiauth-ldap
```

### Mises à jour certificats

```bash
# Régénérer certificats expirés
./scripts/generate-pki.sh

# Redémarrer services RADIUS
sudo docker compose restart freeradius1 freeradius2

# Redistribuer certificats clients
# Copier pki/client/*.p12 vers utilisateurs
```

---

## 🐛 Dépannage

### Problème: Impossible de se connecter à LDAP

**Symptôme:**
```
ldap_bind: Can't contact LDAP server (-1)
```

**Solution:**
```bash
# Vérifier conteneur LDAP
sudo docker ps | grep ldap

# Vérifier logs
sudo docker logs securiauth-ldap

# Tester port
nc -zv localhost 389

# Redémarrer si nécessaire
sudo docker restart securiauth-ldap
```

### Problème: RADIUS rejette authentification

**Symptôme:**
```
radtest: Access-Reject
```

**Solutions:**
```bash
# 1. Vérifier module LDAP activé
sudo docker exec securiauth-radius1 ls -la /etc/freeradius/3.0/mods-enabled/ | grep ldap

# 2. Tester LDAP depuis conteneur RADIUS
sudo docker exec securiauth-radius1 ldapsearch -x -H ldap://openldap:389 \
  -D "cn=admin,dc=securiauth,dc=com" -w "AdminSecur1Auth2025!" \
  -b "dc=securiauth,dc=com"

# 3. Mode debug
sudo docker exec -it securiauth-radius1 freeradius -X
```

### Problème: TACACS+ timeout

**Symptôme:**
```
%AUTHMGR: TACACS+ server timeout
```

**Solutions:**
```bash
# 1. Vérifier connectivité
ping 172.25.0.20

# 2. Vérifier port ouvert
nc -zv 172.25.0.20 49

# 3. Vérifier clé partagée correspond
# Dans tac_plus.conf: key = "TacacsSecuriAuth2025!"
# Sur switch: tacacs-server key TacacsSecuriAuth2025!

# 4. Logs TACACS+
sudo docker logs securiauth-tacacs1
```

### Problème: Certificats EAP-TLS invalides

**Symptôme:**
```
TLS Alert read: fatal: bad certificate
```

**Solutions:**
```bash
# 1. Vérifier validité certificat
openssl x509 -in pki/client/dieyna.diop-cert.pem -noout -dates

# 2. Vérifier chaîne de confiance
openssl verify -CAfile pki/ca/ca-cert.pem pki/client/dieyna.diop-cert.pem

# 3. Régénérer si expiré
./scripts/generate-pki.sh
```

---

## 🔐 Sécurité

### Mots de passe par défaut à changer

| Service         | Utilisateur    | Mot de passe par défaut    | Action               |
|-----------------|----------------|----------------------------|----------------------|
| LDAP Admin      | admin          | AdminSecur1Auth2025!       | ⚠️ CHANGER           |
| MySQL Root      | root           | RootMySQL2025!             | ⚠️ CHANGER           |
| MySQL RADIUS    | radiususer     | RadiusDB2025!              | ⚠️ CHANGER           |
| RADIUS Shared   | N/A            | testing123                 | ⚠️ CHANGER           |
| TACACS+ Key     | N/A            | TacacsSecuriAuth2025!      | ⚠️ CHANGER           |
| Certificats P12 | N/A            | SecuriAuth2025             | ⚠️ CHANGER           |

### Recommandations

1. **Réseau**:
   - Isoler réseau Docker sur VLAN management
   - Firewall: n'exposer que ports nécessaires
   - VPN pour accès admin distant

2. **Certificats**:
   - Renouveler annuellement
   - Révoquer immédiatement si utilisateur quitte
   - Conserver CA key offline après génération

3. **Monitoring**:
   - Alertes sur échecs auth répétés
   - Surveiller usage disque (logs)
   - Monitoring disponibilité services (Nagios/Zabbix)

4. **Audit**:
   - Réviser logs accounting hebdomadairement
   - Audit annuel des accès utilisateurs
   - Documentation changements configuration

---

## 📞 Support

### Contacts

- **Infrastructure**: Dieyna Diop - dieyna.diop@securiauth.com - +221 77 123 45 67
- **Support Technique**: Maman Seck - maman.seck@securiauth.com - +221 76 234 56 78
- **Helpdesk**: support@securiauth.com

### Ressources

- Documentation FreeRADIUS: https://freeradius.org/documentation/
- Documentation TACACS+: http://www.shrubbery.net/tac_plus/
- Documentation OpenLDAP: https://www.openldap.org/doc/

---

**Fin du guide - SecuriAuth Sénégal 2025**
