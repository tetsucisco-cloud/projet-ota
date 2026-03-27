# projet-ota
Présentation d'un projet
# OTA — OpenSource Tenant Architecture

> Script Bash interactif d'administration d'un hébergement web mutualisé.

---

## Sommaire
# OTA — OpenSource Tenant Architecture

> Script Bash interactif d'administration d'un hébergement web mutualisé.

---

## Sommaire

1. [Présentation](#présentation)
2. [Prérequis](#prérequis)
3. [Installation](#installation)
4. [Structure du projet](#structure-du-projet)
5. [Utilisation](#utilisation)
6. [Fonctionnalités détaillées](#fonctionnalités-détaillées)
7. [Architecture technique](#architecture-technique)
8. [Sécurité](#sécurité)
9. [Dépannage](#dépannage)

---

## Présentation

OTA (**O**penSource **T**enant **A**rchitecture) est un script Bash interactif
permettant à un administrateur système de **créer, gérer et supprimer** des
comptes d'hébergement web mutualisé sur un serveur Linux.

Chaque compte hébergé dispose de :

- Un répertoire web `/home/<user>/www`
- Un accès FTP (vsftpd) confiné dans son `www/`
- Un espace de stockage limité par quota disque
- Une base de données MySQL / MariaDB dédiée
- Un accès SSH optionnel
- Un VirtualHost Apache2 généré automatiquement

---

## Prérequis

### Système

| Élément | Version minimale |
|---------|-----------------|
| OS | Debian 11 / Ubuntu 22.04 |
| Bash | 5.x |
| Droits | root (sudo) |

### Services requis

```bash
sudo apt update
sudo apt install apache2 vsftpd mariadb-server quota curl
```

### Activation des quotas disque

```bash
# 1. Éditer /etc/fstab — ajouter usrquota sur la partition /home
#    Exemple :
#    UUID=xxxx  /home  ext4  defaults,usrquota  0 2

# 2. Remonter la partition et initialiser
sudo mount -o remount /home
sudo quotacheck -cum /home
sudo quotaon /home
```

---

## Installation

```bash
# 1. Cloner le dépôt
git clone https://github.com/<votre-username>/ota.git
cd ota

# 2. Rendre le script exécutable
chmod +x ota.sh

# 3. Lancer
sudo ./ota.sh
```

---

## Structure du projet

```
ota/
├── ota.sh                  ← Point d'entrée, menu principal
│
├── lib/
│   ├── users.sh            ← Créer / afficher / modifier / supprimer un hébergement
│   ├── ftp.sh              ← Gestion accès FTP (vsftpd)
│   ├── database.sh         ← Gestion bases de données MySQL / MariaDB
│   ├── apache.sh           ← Gestion VirtualHosts Apache2
│   └── utils.sh            ← Utilitaires, logs, server_info, WordPress
│
├── conf/
│   ├── vhost.conf.tpl      ← Template VirtualHost Apache
│   └── vsftpd.conf.tpl     ← Template configuration vsftpd
│
└── logs/
    ├── ota.log             ← Journal des actions administrateur
    └── errors.log          ← Journal des erreurs
```

**Côté serveur**, chaque hébergement crée l'arborescence suivante :

```
/home/
├── client1/
│   └── www/
│       └── index.html
├── client2/
│   └── www/
│       └── index.html
```

---

## Utilisation

### Lancement

```bash
sudo ./ota.sh
```

### Menu principal

```
  ╔══════════════════════════════════════════════════╗
  ║         OTA — Gestion Hébergement Web            ║
  ║         OpenSource Tenant Architecture           ║
  ╚══════════════════════════════════════════════════╝

  Menu principal
  ─────────────────────────────────────────────────
  1)  Créer un hébergement
  2)  Afficher un hébergement
  3)  Modifier un hébergement
  4)  Supprimer un hébergement
  ─────────────────────────────────────────────────
  5)  Gestion des bases de données
  6)  Gestion FTP
  7)  Gestion des fichiers web
  ─────────────────────────────────────────────────
  8)  Informations serveur
  9)  Quitter
  ─────────────────────────────────────────────────
```

---

## Fonctionnalités détaillées

### 1 — Créer un hébergement

Saisie interactive guidée :

```
  Nom d'utilisateur : client1
  Mot de passe :
  Confirmer le mot de passe :
  Quota disque (ex: 500M, 2G) : 500M
  Créer une base de données ? [o/N] : o
  Activer l'accès SSH ? [o/N] : n
```

Actions exécutées automatiquement :

- Création de l'utilisateur Linux (`useradd`)
- Création de `/home/client1/www/` avec page d'accueil par défaut
- Application du quota disque (`setquota`)
- Ajout à la liste FTP autorisée (`/etc/vsftpd.userlist`)
- Création de la base `client1_db` + utilisateur MySQL dédié
- Génération et activation du VirtualHost Apache

### 2 — Afficher un hébergement

- Liste tous les hébergements (uid ≥ 1000) avec espace utilisé et statut SSH
- Option de zoom sur un utilisateur précis :
  - UID / GID, shell, répertoire
  - Espace disque et quota
  - Fichiers présents dans `www/`
  - Statut base de données
  - Statut VirtualHost Apache

### 3 — Modifier un hébergement

| Option | Action |
|--------|--------|
| Changer le mot de passe | `chpasswd` avec confirmation |
| Changer le quota disque | `setquota` soft 90% / hard 100% |
| Activer / désactiver SSH | `usermod -s /bin/bash` ou `/usr/sbin/nologin` |
| Créer / supprimer la BDD | Appel `create_database()` ou `delete_database()` |

### 4 — Supprimer un hébergement

Double confirmation (retaper le nom d'utilisateur).  
Suppression dans l'ordre : VirtualHost → BDD → FTP → `userdel -r`

### 5 — Gestion des bases de données

- Créer une base (`<user>_db`) avec mot de passe généré aléatoirement
- Supprimer une base et son utilisateur MySQL
- Lister les bases (taille en Mo, nombre de tables)
- Afficher les credentials d'un utilisateur
- Statut du service MySQL / MariaDB

### 6 — Gestion FTP

- Activer / désactiver FTP pour un utilisateur
- Lister les comptes FTP actifs avec espace utilisé
- Statut du service vsftpd (port 21, fichiers de conf)
- Auto-nettoyage des utilisateurs supprimés hors OTA

### 7 — Gestion des fichiers web

- Afficher les fichiers d'un `www/` avec espace utilisé
- Vue globale de l'espace disque par hébergement
- **Installation automatique de WordPress** (voir ci-dessous)
- Accès au sous-menu VirtualHosts Apache

#### Installation WordPress

```
  Nom d'utilisateur hébergé : client1
  [INFO] Credentials DB trouvés automatiquement.
  [...] Téléchargement de WordPress...
  [...] Extraction dans /home/client1/www...
  [...] Configuration de wp-config.php...
  [...] Application des permissions...
  [SUCCÈS] WordPress installé pour 'client1'.
  Finalisez l'installation via : http://client1.local/wp-admin/install.php
```

- Télécharge la dernière version depuis `wordpress.org/latest.tar.gz`
- Génère `wp-config.php` avec les credentials DB OTA
- Injecte des clés de sécurité fraîches (API WordPress)
- Applique `chmod 644` fichiers / `755` dossiers / `600` wp-config.php

### 8 — Informations serveur

Vue d'ensemble en temps réel :

- Hostname, OS, kernel, uptime
- Espace disque et RAM
- Nombre d'hébergements actifs
- Liste des sites avec indicateur actif/inactif
- Statut des services : Apache, vsftpd, MariaDB, SSH
- Charge CPU (load average)

---

## Architecture technique

### Chargement des modules

`ota.sh` source les 5 libs dans l'ordre au démarrage :

```
ota.sh
  └── source lib/utils.sh      ← en premier (log, pause, print_header)
  └── source lib/users.sh
  └── source lib/ftp.sh
  └── source lib/database.sh
  └── source lib/apache.sh
```

### Flux de création d'un hébergement

```
create_user()
    ├── useradd + chpasswd
    ├── mkdir /home/<user>/www
    ├── _apply_quota()
    ├── enable_ftp()            ← lib/ftp.sh
    ├── create_database()       ← lib/database.sh
    └── create_vhost()          ← lib/apache.sh
```

### Génération du VirtualHost

Si `conf/vhost.conf.tpl` existe :

```bash
sed -e "s|{{USERNAME}}|client1|g" \
    -e "s|{{DOCUMENT_ROOT}}|/home/client1/www|g" \
    -e "s|{{SERVER_NAME}}|client1.local|g" \
    -e "s|{{LOG_DIR}}|/var/log/apache2|g" \
    conf/vhost.conf.tpl > /etc/apache2/sites-available/client1.conf
```

Sinon, génération directe par heredoc dans `_vhost_inline()`.

### Credentials base de données

Stockés dans `/etc/ota/db_credentials` (chmod 600, root uniquement) :

```
client1|client1_db|client1|P@ssw0rd!|2025-01-15 10:32:00
client2|client2_db|client2|X9#mK2qL|2025-01-16 14:11:45
```

---

## Sécurité

| Mesure | Détail |
|--------|--------|
| Exécution root requise | `check_root()` bloque si EUID ≠ 0 |
| Protection utilisateurs système | Toute opération sur uid < 1000 est refusée |
| Double confirmation suppression | L'admin doit retaper le nom d'utilisateur |
| Chroot FTP | Chaque user est confiné dans `/home/<user>/www` |
| Credentials DB | Fichier `/etc/ota/db_credentials` en chmod 600 |
| Mot de passe DB | Généré aléatoirement via `/dev/urandom` (16 cars) |
| wp-config.php | Permissions 600 après installation WordPress |
| Clés WordPress | Injectées depuis l'API officielle `api.wordpress.org` |
| Validation des entrées | Nom, quota, mot de passe validés avant toute action |
| Vérification syntaxe Apache | `apache2ctl configtest` avant chaque reload |

---

## Dépannage

### Le script refuse de démarrer

```
[ERREUR] Ce script doit être exécuté en tant que root (sudo).
```

→ Lancer avec `sudo ./ota.sh`

### vsftpd introuvable

```
[AVERTISSEMENT] vsftpd n'est pas installé. FTP ignoré.
```

→ `sudo apt install vsftpd && sudo systemctl enable --now vsftpd`

### Quota non appliqué

```
[AVERTISSEMENT] 'setquota' non disponible. Quota non appliqué.
```

→ `sudo apt install quota` puis vérifier que `/home` est monté avec `usrquota`

### Connexion MySQL impossible

```
[ERREUR] Impossible de se connecter à MySQL/MariaDB.
```

→ `sudo systemctl start mariadb`  
→ Si première installation : `sudo mysql_secure_installation`

### WordPress — téléchargement échoué

```
[ERREUR] Téléchargement échoué. Vérifiez la connexion réseau.
```

→ Vérifier la connexion : `curl -I https://wordpress.org`  
→ Installer curl si absent : `sudo apt install curl`

---

## Licence

Projet réalisé dans le cadre du cours — [www.pedagogeek.fr](https://www.pedagogeek.fr)

1. [Présentation](#présentation)
2. [Prérequis](#prérequis)
3. [Installation](#installation)
4. [Structure du projet](#structure-du-projet)
5. [Utilisation](#utilisation)
6. [Fonctionnalités détaillées](#fonctionnalités-détaillées)
7. [Architecture technique](#architecture-technique)
8. [Sécurité](#sécurité)
9. [Dépannage](#dépannage)

---

## Présentation

OTA (**O**penSource **T**enant **A**rchitecture) est un script Bash interactif
permettant à un administrateur système de **créer, gérer et supprimer** des
comptes d'hébergement web mutualisé sur un serveur Linux.

Chaque compte hébergé dispose de :

- Un répertoire web `/home/<user>/www`
- Un accès FTP (vsftpd) confiné dans son `www/`
- Un espace de stockage limité par quota disque
- Une base de données MySQL / MariaDB dédiée
- Un accès SSH optionnel
- Un VirtualHost Apache2 généré automatiquement

---

## Prérequis

### Système

| Élément | Version minimale |
|---------|-----------------|
| OS | Debian 11 / Ubuntu 22.04 |
| Bash | 5.x |
| Droits | root (sudo) |

### Services requis

```bash
sudo apt update
sudo apt install apache2 vsftpd mariadb-server quota curl
```

### Activation des quotas disque

```bash
# 1. Éditer /etc/fstab — ajouter usrquota sur la partition /home
#    Exemple :
#    UUID=xxxx  /home  ext4  defaults,usrquota  0 2

# 2. Remonter la partition et initialiser
sudo mount -o remount /home
sudo quotacheck -cum /home
sudo quotaon /home
```

---

## Installation

```bash
# 1. Cloner le dépôt
git clone https://github.com/<votre-username>/ota.git
cd ota

# 2. Rendre le script exécutable
chmod +x ota.sh

# 3. Lancer
sudo ./ota.sh
```

---

## Structure du projet

```
ota/
├── ota.sh                  ← Point d'entrée, menu principal
│
├── lib/
│   ├── users.sh            ← Créer / afficher / modifier / supprimer un hébergement
│   ├── ftp.sh              ← Gestion accès FTP (vsftpd)
│   ├── database.sh         ← Gestion bases de données MySQL / MariaDB
│   ├── apache.sh           ← Gestion VirtualHosts Apache2
│   └── utils.sh            ← Utilitaires, logs, server_info, WordPress
│
├── conf/
│   ├── vhost.conf.tpl      ← Template VirtualHost Apache
│   └── vsftpd.conf.tpl     ← Template configuration vsftpd
│
└── logs/
    ├── ota.log             ← Journal des actions administrateur
    └── errors.log          ← Journal des erreurs
```

**Côté serveur**, chaque hébergement crée l'arborescence suivante :

```
/home/
├── client1/
│   └── www/
│       └── index.html
├── client2/
│   └── www/
│       └── index.html
```

---

## Utilisation

### Lancement

```bash
sudo ./ota.sh
```

### Menu principal

```
  ╔══════════════════════════════════════════════════╗
  ║         OTA — Gestion Hébergement Web            ║
  ║         OpenSource Tenant Architecture           ║
  ╚══════════════════════════════════════════════════╝

  Menu principal
  ─────────────────────────────────────────────────
  1)  Créer un hébergement
  2)  Afficher un hébergement
  3)  Modifier un hébergement
  4)  Supprimer un hébergement
  ─────────────────────────────────────────────────
  5)  Gestion des bases de données
  6)  Gestion FTP
  7)  Gestion des fichiers web
  ─────────────────────────────────────────────────
  8)  Informations serveur
  9)  Quitter
  ─────────────────────────────────────────────────
```

---

## Fonctionnalités détaillées

### 1 — Créer un hébergement

Saisie interactive guidée :

```
  Nom d'utilisateur : client1
  Mot de passe :
  Confirmer le mot de passe :
  Quota disque (ex: 500M, 2G) : 500M
  Créer une base de données ? [o/N] : o
  Activer l'accès SSH ? [o/N] : n
```

Actions exécutées automatiquement :

- Création de l'utilisateur Linux (`useradd`)
- Création de `/home/client1/www/` avec page d'accueil par défaut
- Application du quota disque (`setquota`)
- Ajout à la liste FTP autorisée (`/etc/vsftpd.userlist`)
- Création de la base `client1_db` + utilisateur MySQL dédié
- Génération et activation du VirtualHost Apache

### 2 — Afficher un hébergement

- Liste tous les hébergements (uid ≥ 1000) avec espace utilisé et statut SSH
- Option de zoom sur un utilisateur précis :
  - UID / GID, shell, répertoire
  - Espace disque et quota
  - Fichiers présents dans `www/`
  - Statut base de données
  - Statut VirtualHost Apache

### 3 — Modifier un hébergement

| Option | Action |
|--------|--------|
| Changer le mot de passe | `chpasswd` avec confirmation |
| Changer le quota disque | `setquota` soft 90% / hard 100% |
| Activer / désactiver SSH | `usermod -s /bin/bash` ou `/usr/sbin/nologin` |
| Créer / supprimer la BDD | Appel `create_database()` ou `delete_database()` |

### 4 — Supprimer un hébergement

Double confirmation (retaper le nom d'utilisateur).  
Suppression dans l'ordre : VirtualHost → BDD → FTP → `userdel -r`

### 5 — Gestion des bases de données

- Créer une base (`<user>_db`) avec mot de passe généré aléatoirement
- Supprimer une base et son utilisateur MySQL
- Lister les bases (taille en Mo, nombre de tables)
- Afficher les credentials d'un utilisateur
- Statut du service MySQL / MariaDB

### 6 — Gestion FTP

- Activer / désactiver FTP pour un utilisateur
- Lister les comptes FTP actifs avec espace utilisé
- Statut du service vsftpd (port 21, fichiers de conf)
- Auto-nettoyage des utilisateurs supprimés hors OTA

### 7 — Gestion des fichiers web

- Afficher les fichiers d'un `www/` avec espace utilisé
- Vue globale de l'espace disque par hébergement
- **Installation automatique de WordPress** (voir ci-dessous)
- Accès au sous-menu VirtualHosts Apache

#### Installation WordPress

```
  Nom d'utilisateur hébergé : client1
  [INFO] Credentials DB trouvés automatiquement.
  [...] Téléchargement de WordPress...
  [...] Extraction dans /home/client1/www...
  [...] Configuration de wp-config.php...
  [...] Application des permissions...
  [SUCCÈS] WordPress installé pour 'client1'.
  Finalisez l'installation via : http://client1.local/wp-admin/install.php
```

- Télécharge la dernière version depuis `wordpress.org/latest.tar.gz`
- Génère `wp-config.php` avec les credentials DB OTA
- Injecte des clés de sécurité fraîches (API WordPress)
- Applique `chmod 644` fichiers / `755` dossiers / `600` wp-config.php

### 8 — Informations serveur

Vue d'ensemble en temps réel :

- Hostname, OS, kernel, uptime
- Espace disque et RAM
- Nombre d'hébergements actifs
- Liste des sites avec indicateur actif/inactif
- Statut des services : Apache, vsftpd, MariaDB, SSH
- Charge CPU (load average)

---

## Architecture technique

### Chargement des modules

`ota.sh` source les 5 libs dans l'ordre au démarrage :

```
ota.sh
  └── source lib/utils.sh      ← en premier (log, pause, print_header)
  └── source lib/users.sh
  └── source lib/ftp.sh
  └── source lib/database.sh
  └── source lib/apache.sh
```

### Flux de création d'un hébergement

```
create_user()
    ├── useradd + chpasswd
    ├── mkdir /home/<user>/www
    ├── _apply_quota()
    ├── enable_ftp()            ← lib/ftp.sh
    ├── create_database()       ← lib/database.sh
    └── create_vhost()          ← lib/apache.sh
```

### Génération du VirtualHost

Si `conf/vhost.conf.tpl` existe :

```bash
sed -e "s|{{USERNAME}}|client1|g" \
    -e "s|{{DOCUMENT_ROOT}}|/home/client1/www|g" \
    -e "s|{{SERVER_NAME}}|client1.local|g" \
    -e "s|{{LOG_DIR}}|/var/log/apache2|g" \
    conf/vhost.conf.tpl > /etc/apache2/sites-available/client1.conf
```

Sinon, génération directe par heredoc dans `_vhost_inline()`.

### Credentials base de données

Stockés dans `/etc/ota/db_credentials` (chmod 600, root uniquement) :

```
client1|client1_db|client1|P@ssw0rd!|2025-01-15 10:32:00
client2|client2_db|client2|X9#mK2qL|2025-01-16 14:11:45
```

---

## Sécurité

| Mesure | Détail |
|--------|--------|
| Exécution root requise | `check_root()` bloque si EUID ≠ 0 |
| Protection utilisateurs système | Toute opération sur uid < 1000 est refusée |
| Double confirmation suppression | L'admin doit retaper le nom d'utilisateur |
| Chroot FTP | Chaque user est confiné dans `/home/<user>/www` |
| Credentials DB | Fichier `/etc/ota/db_credentials` en chmod 600 |
| Mot de passe DB | Généré aléatoirement via `/dev/urandom` (16 cars) |
| wp-config.php | Permissions 600 après installation WordPress |
| Clés WordPress | Injectées depuis l'API officielle `api.wordpress.org` |
| Validation des entrées | Nom, quota, mot de passe validés avant toute action |
| Vérification syntaxe Apache | `apache2ctl configtest` avant chaque reload |

---

## Dépannage

### Le script refuse de démarrer

```
[ERREUR] Ce script doit être exécuté en tant que root (sudo).
```

→ Lancer avec `sudo ./ota.sh`

### vsftpd introuvable

```
[AVERTISSEMENT] vsftpd n'est pas installé. FTP ignoré.
```

→ `sudo apt install vsftpd && sudo systemctl enable --now vsftpd`

### Quota non appliqué

```
[AVERTISSEMENT] 'setquota' non disponible. Quota non appliqué.
```

→ `sudo apt install quota` puis vérifier que `/home` est monté avec `usrquota`

### Connexion MySQL impossible

```
[ERREUR] Impossible de se connecter à MySQL/MariaDB.
```

→ `sudo systemctl start mariadb`  
→ Si première installation : `sudo mysql_secure_installation`

### WordPress — téléchargement échoué

```
[ERREUR] Téléchargement échoué. Vérifiez la connexion réseau.
```

→ Vérifier la connexion : `curl -I https://wordpress.org`  
→ Installer curl si absent : `sudo apt install curl`

---

## Licence

Projet réalisé dans le cadre du cours — [www.pedagogeek.fr](https://www.pedagogeek.fr)
