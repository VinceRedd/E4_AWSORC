# Modélisation

Afin de réaliser notre projet de manière efficace, nous avons tout d'abord mis au point cette modélisation qui nous permet d'obtenir une vue d'ensemble de l'architecture à établir :

![alt text](image-88.png)

_En cas de difficulté, vous pouvez ouvrir les fichiers images directement, celles-ci sont présentes dans le répertoire._

# Partie 1 
## 🌐 Création du VPC

Un **VPC (Virtual Private Cloud)** est un réseau privé isolé dans le cloud AWS. Il permet de contrôler totalement l'adressage IP, les sous-réseaux, les tables de routage et les règles de sécurité de tes ressources AWS.  
C'est l'équivalent d'un réseau d'entreprise dans le cloud, où tu peux déployer des **instances EC2, des bases de données RDS, des load balancers, etc...** tout en **contrôlant l'accès et la communication entre eux**.

---
Nous créons notre VPC en passant par le **wizard** d'AWS.  
C'est une méthode simple et rapide qui configure automatiquement :
- **Deux sous-réseaux** (privé & public) dans **deux zones de disponibilité différentes** (obligatoire pour RDS).
- Une **passerelle Internet** pour permettre aux ressources publiques d'accéder à Internet.
- Des **règles de sécurité** de base.

L'utilisation du wizard est particulièrement pratique, surtout lorsque l'on travaille avec **RDS**, qui exige **au moins deux sous-réseaux distincts** pour garantir **la haute disponibilité**.

👉 **Ensuite, on pourra ajuster les paramètres du VPC selon nos besoins !** 


![alt text](image-20.png)

![alt text](image-23.png)


On peut créer notre VPC.


Ainsi, on obtient :

| Zone de Disponibilité | Sous-réseau public | Sous-réseau privé |
|-----------------------|------------------|------------------|
| **us-east-1a** | **E4_CACCIATORE_ProjetFinal-subnet-private1** | **E4_CACCIATORE_ProjetFinal-subnet-private1** |
| us-east-1b | E4_CACCIATORE_ProjetFinal-subnet-public2 | E4_CACCIATORE_ProjetFinal-subnet-private2 |

Ici, on traitera seulement la première zone de disponibilité.
Les tables de routages, passerelle Internet, et connexions réseaux sont automatiquement générées et gérées.

## 💻 Création de l'Instance (Application)

Une **instance EC2 (Elastic Compute Cloud)** est un **serveur virtuel** dans le cloud AWS.  
Elle permet d'exécuter des applications, des sites web ou des services en fonction des besoins.  
EC2 offre une grande **scalabilité**, avec la possibilité d'ajouter ou de supprimer des instances selon la charge et les performances requises.

---
Ainsi, nous allons mettre en place une **instance EC2** qui hébergera notre application et sera accessible depuis l'extérieur.

### 🔹 Étapes de création :
1. **Sélection du VPC**  
   - **VPC** précédemment créé pour s'assurer que l'instance soit bien intégrée à notre infrastructure.

2. **Choix du sous-réseau**  
   - On choisit le **sous-réseau public** pour que l'instance puisse communiquer avec Internet.

3. **Attribution d'une adresse IP publique**  
   - On active **l'attribution automatique d'une adresse IP publique** afin que l'instance soit accessible depuis l'extérieur.

4. **Création et configuration du groupe de sécurité**  
   - Ajouter les règles de sécurité nécessaires :
     - **SSH (port 22)** : pour l'administration de l'instance (limiter aux IP de confiance).
     - **Port 5085** : on ouvre ce port pour permettre l'accès à l'application depuis Internet.

👉 **Une fois l'instance créée, nous pourrons déployer notre application et configurer son accès !** 

![alt text](image-24.png)

Pour faciliter l'installation, on a fait le choix d'utiliser un cloud init afin d'installer directement docker et de cloner le répertoire git de notre application (https://github.com/app-generator/flask-datta-able.git).

Voici le code utilisé :
```
#cloud-config

# Configurer le fuseau horaire
timezone: Europe/Paris

# Mise à jour des paquets
package_update: true
package_upgrade: true

# Installation des paquets nécessaires
packages:
  - git
  - docker
  - docker-compose

# Configuration de l'utilisateur et des groupes
groups:
  - docker
users:
  - default
  - name: ubuntu
    groups: docker
    shell: /bin/bash

runcmd:
  # Activer et démarrer Docker
  - systemctl enable --now docker

  # Ajouter l'utilisateur "ubuntu" au groupe Docker
  - usermod -aG docker ubuntu

  # Appliquer les changements immédiatement sans redémarrer (évite d'attendre un relog)
  - newgrp docker

  # Cloner l'application Django Volt Dashboard
  - git clone https://github.com/app-generator/flask-datta-able.git /home/ubuntu/flask-datta-able
```
On l'insère tout en bas de notre page de création d'instance EC2 :

![alt text](image-25.png)
Après quelques temps, notre instance est bien créée et démarrée.

## ⚡ SSH
**SSH (Secure Shell)** est un protocole permettant de se connecter de manière sécurisée à un serveur distant.  
Il utilise le chiffrement pour garantir la confidentialité et l'intégrité des communications entre le client et le serveur.  
C'est l'outil principal pour administrer une instance **EC2** en ligne de commande.

On configure notre environnement VSCode pour qu'on puisse se connecter directement et avoir accès au répertoire distant :

![alt text](image-27.png)

![alt text](image-26.png)

Avant toute chose, il est important d'optimiser le DockerFile :

```
FROM python:3.12-alpine

# set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    FLASK_APP=run.py

COPY requirements.txt .

# install python dependencies
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

# Init migration folder
# RUN flask db init # to be executed only once
RUN flask db migrate && flask db upgrade

# gunicorn
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "run:app"]

```

Puis le docker-compose.yml :
```
version: '3.8'
services:
  appseed-app:
    container_name: appseed_app
    restart: always
    build: .
    networks:
      - db_network
      - web_network
    env_file:
        .env
  nginx:
    container_name: nginx
    restart: always
    image: "nginx:1.22.1-alpine-slim"
    ports:
      - "5085:5085"
    volumes:
      - ./nginx:/etc/nginx/conf.d
    networks:
      - web_network
    depends_on: 
      - appseed-app
networks:
  db_network:
    driver: bridge
  web_network:
    driver: bridge
 
```

On optimise aussi le .gitignore :
```
# Byte-compiled / optimized / DLL files
__pycache__/
*.py[cod]
*.pyo
*.pyd
*.so
*.dll

# Tests and coverage
*.pytest_cache
.coverage
htmlcov/
nosetests.xml
coverage.xml
*.cover
*.py,cover
.cache
junitxml/
*.log
.tox/
.nox/
.envrc

# Database & logs
*.db
#*.sqlite3
*.log
*.sql

# Virtual environments
env/
env_linux/
venv/
venv*/
pip-log.txt
pip-delete-this-directory.txt

# System files
.DS_Store
Thumbs.db
desktop.ini
*.swp
*.swo

# Sphinx docs
_build/
_static/
_templates/

# Node.js / JS dependencies
node_modules/
package-lock.json
yarn.lock
apps/static/assets/node_modules/
apps/static/assets/yarn.lock
apps/static/assets/.temp/

# Migrations (optional, uncomment if you don't want to track migrations)
#migrations/

# IDE and editor files
.vscode/
.vscode/symbols.json
.idea/
*.iml

# Docker related
docker-compose.override.yml
.env
.env.local
.env.production
.env.test
.env.sample
.envrc

# Nginx and deployment-related files
nginx/
*.crt
*.key
*.pem

# Python-specific
__pypackages__/
Pipfile
Pipfile.lock
requirements.txt
pyproject.toml
setup.py
setup.cfg

# Security and auth
*.pem
*.key
*.crt
*.pfx
*.p12

```
Et le .dockerignore :
```
.git
__pycache__
*.pyc
*.pyo
*.pyd
*.db
*.sqlite
*.log
*.log.*
instance/
*.env
*.flaskenv
.vscode
.idea
*.md
package.json
render.yaml
Dockerfile
```

Enfin, on lance l'application : 
```
docker-compose up -d
```
Celle-ci est accessible par défaut sur le port 5085 de l'IP publique de notre instance :
![alt text](image-45.png)

On entre donc 44.197.172.233:5085 sur notre machine (hôte) dans notre navigateur :
![alt text](image-28.png)
On y a bien accès depuis l'extérieur.

## 🗄 Base de données (RDS)

On se rend dans le service RDS, puis on créé notre base de données MySQL en suivant cette configuration : 

![alt text](image-30.png)

On se met bien sur l'offre gratuite :
![alt text](image-31.png)

On définit nos identifiants :
![alt text](image-32.png)

On active et définit la mise à l'échelle : 
![alt text](image-33.png)

On met en place toute notre connectivité.

On créé un NSG dédié et on sélectionne le bon VPC (créé précédemment).
![alt text](image-35.png)

On configure le nom de la DB pour que AWS la créée et on active les sauvegardes automatiques : 

![alt text](image-36.png)

On peut désormais la créer.


On configure par la suite une connexion EC2 vers notre instance :
![alt text](image-38.png)


---
On peut désormais se connecter en SSH sur notre DB :
```
mysql -h dbcacciatoretpfinal.csdwaigwo7xn.us-east-1.rds.amazonaws.com -P 3306 -u admin -p
```

![alt text](image-42.png)
![alt text](image-41.png)

On sélectionne notre BDD, on regarde nos tables et on affiche l'ensemble des users créés. Il n'y en a pas.

On se rend donc sur notre application pour en créer un :

![alt text](image-39.png)

Notre user est bien créé.
![alt text](image-40.png)

On affiche encore une fois l'ensemble des users créés : 
```
SELECT * FROMS users;
```
![alt text](image-43.png)

Notre compte s'affiche bien (un autre "pizza" est aussi présent car on avait testé auparavant).

Pour être sûr, on en créé un autre :

![alt text](image-46.png)

![alt text](image-47.png)
On sélectionne une nouvelle fois l'ensemble des users dans notre table "users".
Notre user "vincetpfinalAWS" est bien présent.

## Création d'une seconde instance (serveur de test)

On créé une seconde instance Ubuntu dans notre VPC dans le privé :

_E4_CACCIATORE_ProjetFinal_ServTestPrive_

![alt text](image-48.png)

On créé par la suite notre passerelle NAT.
Elle permet aux instances d’un sous-réseau privé d’accéder à Internet tout en empêchant les connexions entrantes. Elle est utile pour télécharger des mises à jour ou envoyer des requêtes sans exposer directement les instances à Internet.

![alt text](image-49.png)

Enfin, depuis notre première instance (publique), on se connecte à la seconde via son ip privée :
![alt text](image-51.png)
![alt text](image-52.png)

On arrive bien à ping vers les serveurs de Google depuis notre instance en s'ayant connectée via l'IP privée au travers de notre première instance "publique" :
![alt text](image-53.png)
![alt text](image-54.png)

# 📦 Bucket

Un **bucket S3** est un conteneur de stockage dans **Amazon S3**, permettant de **stocker et organiser des fichiers** (objets) dans le cloud. Il est utilisé pour **héberger des données** comme des images, vidéos, logs ou sauvegardes, avec une **haute disponibilité** et une **sécurité renforcée**.  

On créé notre compartiment en suivant cette configuration :

![alt text](image-57.png)
![alt text](image-56.png)

On se rend dans l'onglet "Autorisations" de notre compartiment :
![alt text](image-59.png)

![alt text](image-60.png)

Tout en bas de "Propriétés" :
![alt text](image-65.png)

![alt text](image-66.png)

On glisse notre site web statique :
![alt text](image-67.png)

On a bien notre site web statique :
![alt text](image-68.png)

![alt text](image-69.png)

On a bien accès à notre site depuis Internet :

![alt text](image-70.png)


## 🖴 Sauvegardes

### EC2

Dans le menu de l'instance EC2 :

![alt text](image-71.png)

![alt text](image-72.png)

On vient de créer une snapshot de notre instance.

### DataBase

De la même façon, on se rend dans l'interface RDS :
![alt text](image-73.png)


![alt text](image-74.png)

On a bien nos snapshots de notre DB et notre application :
![alt text](image-75.png)
![alt text](image-76.png)


# Partie 2

On créé le VPC avec la bonne configuration :
![alt text](image-77.png)

On créé le sous-réseau : 
![alt text](image-78.png)

On créé la table de routage :
![alt text](image-79.png)

On associe notre subnet à notre RTB :
![alt text](image-80.png)

On met en place ensuite le peering entre nos deux VPCs.
Le **peering** permet de **connecter directement** deux réseaux, comme **deux VPC sur AWS**, sans passer par Internet.
Idéal pour **interconnecter** des services tout en gardant un **trafic privé** et **optimisé**.

![alt text](image-81.png)
![alt text](image-82.png)

RTB de mon VPC2 :
![alt text](image-83.png)

RTB de mon VPC1 Subnet Public :
![alt text](image-84.png)

---
On créé une autre instance EC2 que l'on met dans notre second VPC : 
![alt text](image-85.png)

On arrive bien à se connecter à l'instance présente dans le VPC2, depuis l'instance présente dans le subnet publique du VPC1.

![alt text](image-86.png)

On teste le ping (on a autorisé le protocole ICMP sur mon NSG) : 

![alt text](image-87.png)

On a bien accès à Internet.

### 📌 Conclusion

Dans le cadre de ce projet, nous avons mis en place une **infrastructure cloud robuste et sécurisée** sur **AWS**, en respectant les exigences du client tout en optimisant les coûts et la scalabilité.  

---

### 🔹 **Partie 1 : Déploiement de l'environnement initial**  
Nous avons construit un **Virtual Private Cloud (VPC)** avec :  
✔️ **Un sous-réseau public** hébergeant une **instance EC2** pour l’application web.  
✔️ **Un sous-réseau privé** contenant un **serveur de test** non accessible directement depuis Internet.  
✔️ Une **base de données RDS** sécurisée et scalable, connectée à l’application.  
✔️ Un **bucket S3** pour l’hébergement **statique des fichiers**.  
✔️ Une **passerelle NAT** permettant au serveur de test d’accéder à Internet sans être exposé.  
✔️ Des **sauvegardes automatisées** (snapshots EC2 & RDS) pour assurer la résilience du système.  

Le déploiement de l’application s’est fait via **Docker** et **Nginx**, permettant une **gestion efficace des conteneurs** et une meilleure modularité. La connexion entre l’application et la base de données a été vérifiée via des tests fonctionnels, garantissant le bon échange des données.  

---

### 🔹 **Partie 2 : Ajout d’un second environnement pour l’IA**  
Le client souhaitant scinder ses environnements, nous avons mis en place :  
✔️ **Un deuxième VPC**, isolé du premier, contenant un serveur pour l’équipe IA.  
✔️ **Un peering entre les VPCs** pour permettre la communication sécurisée entre les deux environnements.  
✔️ Une gestion des accès et des règles de routage optimisées, garantissant un **trafic sécurisé** et un accès à Internet sans exposition.  

---

### 🚀 **Bilan et perspectives**  

Avec cette architecture, l’entreprise dispose d’un **environnement de développement performant**, prêt à évoluer vers un **déploiement en production sécurisé et extensible**. 🚀🔒
