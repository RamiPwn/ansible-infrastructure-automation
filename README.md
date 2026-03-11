# 🖥️ Ansible Infrastructure Lab

Automatisation d'une infrastructure Linux avec **Ansible** sur des machines virtuelles **Proxmox**.  
Ce projet met en place un contrôleur Ansible centralisé administrant plusieurs serveurs (web, base de données, FTP) de manière entièrement automatisée.

---

## 📋 Table des matières

1. [Overview](#overview)
2. [Architecture du projet](#architecture-du-projet)
3. [Technologies utilisées](#technologies-utilisées)
4. [Structure du repository](#structure-du-repository)
5. [Préparation des serveurs](#préparation-des-serveurs)
6. [Installation du contrôleur Ansible](#installation-du-contrôleur-ansible)
7. [Configuration SSH](#configuration-ssh)
8. [Création de l'inventory](#création-de-linventory)
9. [Déploiement des services](#déploiement-des-services)
10. [Gestion des services](#gestion-des-services)
11. [Automatisation et logs](#automatisation-et-logs)
12. [Automatisation avec cron](#automatisation-avec-cron)
13. [Gestion du projet avec Git](#gestion-du-projet-avec-git)
14. [Conclusion](#conclusion)

---

## Overview

Ce lab démontre la mise en place d'une infrastructure Linux automatisée via Ansible. L'objectif est de centraliser l'administration de plusieurs serveurs depuis un unique point de contrôle, en automatisant les installations, configurations et vérifications de services.

**Cas d'usage couverts :**
- Déploiement automatique d'Apache, MariaDB et ProFTPD
- Gestion des services (start / stop / status) sur l'ensemble du parc
- Supervision des services avec écriture automatique dans des fichiers de logs
- Planification des vérifications via cron

---

## Architecture du projet

| Machine | IP | Rôle |
|---|---|---|
| `ansible-controller` | 192.168.93.81 | Contrôleur Ansible |
| `srv-web` | 192.168.93.82 | Serveur Web (Apache) |
| `srv-bdd` | 192.168.93.83 | Base de données (MariaDB) |
| `srv-web-bdd` | 192.168.93.84 | Web + BDD |
| `srv-ftp-new` | 192.168.93.85 | Serveur FTP (ProFTPD) |

> Toutes les machines sont hébergées sur **Proxmox** dans le réseau `192.168.93.0/24`.

---

## Technologies utilisées

- **Proxmox VE** — Hyperviseur de virtualisation
- **Debian / Ubuntu** — OS des machines virtuelles
- **Ansible** — Outil d'automatisation (agentless, via SSH)
- **Apache2** — Serveur Web
- **MariaDB** — Base de données relationnelle
- **ProFTPD** — Serveur FTP
- **Git** — Versioning de l'infrastructure

---

## Structure du repository

```
ansible-infrastructure-lab/
├── README.md
├── inventory                        # Inventaire des hôtes Ansible
├── ansible.cfg                      # Configuration Ansible
├── playbooks/
│   ├── install_apache.yml
│   ├── install_mariadb.yml
│   ├── install_ftp.yml
│   ├── init_new_servers.yml
│   ├── add_admin_user.yml
│   ├── deploy_apache_config.yml
│   ├── backup_db.yml
│   ├── check_services_log.yml
│   ├── start_services.yml
│   ├── stop_services.yml
│   └── update_servers.yml
├── configs/
│   └── my_custom.conf               # Config Apache custom
├── assets/                          # Captures d'écran / schémas
└── logs/                            # Logs générés automatiquement
```

---

## Préparation des serveurs

Cette étape est à répéter sur **chaque serveur** (sauf le contrôleur).

### Mise à jour et paquets requis

```bash
apt update && apt upgrade -y
apt install -y sudo openssh-server python3
```

### Vérification du service SSH

```bash
systemctl status ssh
```

![SSH status](../assets/image1.png)

### Création de l'utilisateur d'administration

```bash
# Création de l'utilisateur
adduser adminsys

# Ajout au groupe sudo
usermod -aG sudo adminsys

# Sudo sans mot de passe (requis pour Ansible)
echo "adminsys ALL=(ALL:ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers
```

> ⚠️ La configuration `NOPASSWD:ALL` est indispensable pour qu'Ansible puisse exécuter des commandes root sans interaction.

**Commande rapide (one-liner) :**
```bash
adduser --disabled-password --gecos "" adminsys && \
usermod -aG sudo adminsys && \
echo "adminsys ALL=(ALL:ALL) NOPASSWD:ALL" | tee -a /etc/sudoers
```

### Vérification des privilèges

```bash
su - adminsys
sudo whoami
# Résultat attendu : root
```

---

## Installation du contrôleur Ansible

Cette étape est réalisée **uniquement sur `ansible-controller`**.

### Connexion SSH au contrôleur

```bash
ssh adminsys@192.168.93.81
```

### Installation d'Ansible et Git

```bash
sudo apt install -y ansible git
```

### Vérification de l'installation

```bash
ansible --version
```

![ansible --version](../assets/image2.png)

---

## Configuration SSH

### Génération de la clé SSH (sur `ansible-controller`)

```bash
ssh-keygen -t ed25519
# Valider les options par défaut
```

La clé est créée dans `~/.ssh/id_ed25519`.

![Génération de la clé SSH ed25519](../assets/image3.png)

### Copie de la clé vers chaque serveur

```bash
ssh-copy-id adminsys@192.168.93.82
ssh-copy-id adminsys@192.168.93.83
ssh-copy-id adminsys@192.168.93.84
ssh-copy-id adminsys@192.168.93.85
```

### Test de connexion sans mot de passe

```bash
ssh adminsys@192.168.93.82
# La connexion doit s'établir sans demander de mot de passe
```

![Connexion SSH sans mot de passe](../assets/image4.png)

---

## Création de l'inventory

### Initialisation de l'environnement sur `ansible-controller`

```bash
mkdir ~/ansible && cd ~/ansible
mkdir playbooks configs
touch inventory
```

### Contenu du fichier `inventory`

```ini
[web]
srv-web      ansible_host=192.168.93.82
srv-web-bdd  ansible_host=192.168.93.84

[bdd]
srv-bdd      ansible_host=192.168.93.83
srv-web-bdd  ansible_host=192.168.93.84

[ftp]
srv-ftp-new  ansible_host=192.168.93.85

[nouveaux]
srv-ftp-new  ansible_host=192.168.93.85

[debian:children]
web
bdd
ftp
nouveaux

[debian:vars]
ansible_user=adminsys
```

Fichier disponible dans : [`inventory`](./inventory)

### Configuration `ansible.cfg`

Pour supprimer les avertissements inutiles :

```ini
[defaults]
inventory            = inventory
host_key_checking    = False
deprecation_warnings = False
interpreter_python   = auto_silent
```

Configuration disponible dans : [`ansible.cfg`](./ansible.cfg)

### Test de communication Ansible

```bash
ansible all -i inventory -m ping
```

![Résultat du ping Ansible sur tous les hôtes](../assets/image5.png)

---

## Déploiement des services

### Apache (groupe `web`)

Playbook disponible dans : [`playbooks/install_apache.yml`](./playbooks/install_apache.yml)

```bash
ansible-playbook -i inventory playbooks/install_apache.yml
```

![Exécution du playbook install_apache](../assets/image6.png)

---

### MariaDB (groupe `bdd`)

Playbook disponible dans : [`playbooks/install_mariadb.yml`](./playbooks/install_mariadb.yml)

```bash
ansible-playbook -i inventory playbooks/install_mariadb.yml
```

![Exécution du playbook install_mariadb](../assets/image7.png)

---

### ProFTPD (groupe `ftp`)

Playbook disponible dans : [`playbooks/install_ftp.yml`](./playbooks/install_ftp.yml)

```bash
ansible-playbook -i inventory playbooks/install_ftp.yml
```

![Exécution du playbook install_ftp](../assets/image8.png)

---

### Initialisation des nouveaux serveurs

Met à jour le système et installe les outils essentiels sur les nouvelles machines.

Playbook disponible dans : [`playbooks/init_new_servers.yml`](./playbooks/init_new_servers.yml)

```bash
ansible-playbook -i inventory playbooks/init_new_servers.yml
```

**Outils installés :** `vim`, `curl`, `git`, `htop`

---

### Ajout d'un utilisateur d'administration

Playbook disponible dans : [`playbooks/add_admin_user.yml`](./playbooks/add_admin_user.yml)

```bash
ansible-playbook -i inventory playbooks/add_admin_user.yml
```

Ce playbook crée l'utilisateur `adminremote` avec les droits sudo sur tous les serveurs du groupe `debian`.

---

### Déploiement d'une configuration Apache custom

La configuration personnalisée ajoute des logs custom et des restrictions de sécurité sans modifier le fichier de configuration principal de Debian.

Configuration disponible dans : [`configs/my_custom.conf`](./configs/my_custom.conf)

Playbook disponible dans : [`playbooks/deploy_apache_config.yml`](./playbooks/deploy_apache_config.yml)

```bash
ansible-playbook -i inventory playbooks/deploy_apache_config.yml
```

**Tests de validation :**

```bash
# Depuis le serveur web lui-même
curl http://localhost

# Depuis une autre machine
curl http://192.168.93.82

# Vérification du service
ansible web -i inventory -a "sudo systemctl status apache2"

# Vérification de la syntaxe Apache
ansible web -i inventory -a "sudo apache2ctl configtest"
# Résultat attendu : Syntax OK
```

---

## Gestion des services

### Vérification du statut des services

```bash
# Apache
ansible web -i inventory -a "systemctl status apache2"

# MariaDB
ansible bdd -i inventory -a "systemctl status mariadb"

# FTP
ansible ftp -i inventory -a "systemctl status proftpd"
```

### Arrêt de tous les services

Playbook disponible dans : [`playbooks/stop_services.yml`](./playbooks/stop_services.yml)

```bash
ansible-playbook -i inventory playbooks/stop_services.yml
```

### Démarrage de tous les services

Playbook disponible dans : [`playbooks/start_services.yml`](./playbooks/start_services.yml)

```bash
ansible-playbook -i inventory playbooks/start_services.yml
```

### Mise à jour des serveurs

Playbook disponible dans : [`playbooks/update_servers.yml`](./playbooks/update_servers.yml)

```bash
ansible-playbook -i inventory playbooks/update_servers.yml
```

### Sauvegarde des bases MariaDB

Playbook disponible dans : [`playbooks/backup_db.yml`](./playbooks/backup_db.yml)

```bash
ansible-playbook -i inventory playbooks/backup_db.yml
```

Le dump est enregistré dans `/backup/all_databases.sql` sur chaque serveur BDD.

---

## Automatisation et logs

Le playbook `check_services_log.yml` vérifie l'état de tous les services sur l'ensemble du parc et enregistre les résultats dans un fichier de log horodaté.

Playbook disponible dans : [`playbooks/check_services_log.yml`](./playbooks/check_services_log.yml)

```bash
ansible-playbook -i inventory playbooks/check_services_log.yml
```

### Consultation des logs

```bash
cat ~/ansible/logs/services_report.log
```

![Contenu du fichier services_report.log](../assets/image9.png)

Exemple de sortie par serveur :

```
--------------------------------------------------
DATE : 2026-03-11 09:57:00
SRV  : srv-web (192.168.93.82)
--------------------------------------------------
Apache2      | active
MariaDB      | inactive
ProFTPD      | inactive
--------------------------------------------------
```

---

## Automatisation avec cron

### Configuration du crontab

```bash
crontab -e
```

### Exécution toutes les minutes (phase de test)

```bash
* * * * * cd /home/adminsys/ansible && /usr/bin/ansible-playbook -i inventory playbooks/check_services_log.yml > /dev/null 2>&1
```

Surveiller la génération des fichiers en temps réel :

```bash
watch -n 1 ls -lh ~/ansible/logs/
```

![Fichiers de logs générés par cron](../assets/image10.png)

### Passage à l'intervalle horaire (production)

```bash
0 * * * * cd /home/adminsys/ansible && /usr/bin/ansible-playbook -i inventory playbooks/check_services_log.yml > /dev/null 2>&1
```

---

## Gestion du projet avec Git

### Configuration initiale

```bash
git config --global user.name "Admin Sys"
git config --global user.email "adminsys@example.com"
```

### Initialisation et premier commit

```bash
cd ~/ansible
git init
git add playbooks/*.yml configs/*.conf inventory ansible.cfg
git commit -m "Initialisation infrastructure Ansible"
```

### Workflow de versioning

À chaque modification d'un playbook ou fichier de configuration :

```bash
git add playbooks/install_apache.yml
git commit -m "Ajout configuration custom Apache"
```

### Consulter l'historique

```bash
git log
```

![Historique des commits Git](../assets/image11.png)

---

## Conclusion

Ce projet démontre la mise en place d'une infrastructure Linux entièrement administrée par Ansible :

- **Centralisation** : un seul point de contrôle gère l'ensemble du parc serveur
- **Idempotence** : les playbooks peuvent être relancés sans effets indésirables
- **Automatisation** : cron + Ansible assurent une supervision régulière et des logs horodatés
- **Versioning** : Git trace chaque évolution de l'infrastructure as code

L'approche **Infrastructure as Code** adoptée ici permet de reproduire l'environnement complet en quelques commandes, de collaborer via Git, et d'étendre facilement le parc en ajoutant de nouvelles entrées dans l'inventory.

---

> **Auteur** : adminsys  
> **Environnement** : Proxmox VE / Debian  
> **Outil principal** : Ansible Core 2.19.4
