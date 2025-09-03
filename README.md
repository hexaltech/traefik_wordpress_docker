# 🚀 Lab WordPress + Traefik v3 + SSL (Debian 12)

Ce projet déploie un **WordPress hautement disponible** avec **MariaDB** et **Traefik v3** comme reverse proxy.  
Le tout est **sécurisé en HTTPS** grâce à Let's Encrypt, avec **load balancing** entre plusieurs containers WordPress.  

---

## 📦 Prérequis

- Serveur **Debian 12** avec accès root
- **Docker** et **Docker Compose** installés
- Un domaine configuré (ex: `jetestmonlab.fr`) pointant vers l’IP publique du serveur
- Ports ouverts / NATés :
  - **80 TCP** → serveur (pour Let's Encrypt)
  - **443 TCP** → serveur (HTTPS)

---

## ⚙️ Installation

### 1️⃣ Installer Docker et Docker Compose
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker
```

### 2️⃣ Préparer le projet
```bash
mkdir -p ~/traefik-wordpress-lab/letsencrypt
cd ~/traefik-wordpress-lab
touch docker-compose.yml
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

### 3️⃣ Copier le fichier `docker-compose.yml`

```yaml
version: "3.9"

services:
  # ----------- Traefik -----------
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.sslresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.sslresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.sslresolver.acme.email=contact@hexaltech.fr"
      - "--certificatesresolvers.sslresolver.acme.storage=/letsencrypt/acme.json"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    networks:
      - web

  # ----------- Base de données -----------
  db:
    image: mariadb:10.6
    container_name: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - web

  # ----------- WordPress (scalable) -----------
  wordpress:
    image: wordpress:php8.2-apache
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.rule=Host(`hexaltech.fr`)"
      - "traefik.http.routers.wordpress.entrypoints=websecure"
      - "traefik.http.routers.wordpress.tls.certresolver=sslresolver"
      - "traefik.http.services.wordpress.loadbalancer.server.port=80"
    volumes:
      - wordpress_data:/var/www/html
    networks:
      - web

networks:
  web:

volumes:
  db_data:
  wordpress_data:
```

---

## 🚀 Déploiement

Démarrer les services :
```bash
docker compose-up -d
```

Vérifier que Traefik a bien généré le certificat :
```bash
docker logs -f traefik
```
Tu dois voir `Obtained certificates from ACME`.

---

## 🌍 Accès aux services

- **WordPress** : [https://hexaltech.fr](https://hexaltech.fr)  
- **Dashboard Traefik (optionnel)** : `http://<IP_SERVEUR>:8080`  

---

## ⚖️ Activer le Load Balancing

Par défaut, 1 seul WordPress est lancé.  
Tu peux en déployer plusieurs avec la commande :

```bash
docker compose-up --scale wordpress=2 -d
```

👉 Grâce au volume partagé `wordpress_data`, tous les containers WordPress utilisent **le même contenu** (un seul site web).  
👉 Traefik fait automatiquement le **load balancing** entre eux (round-robin).

---

## ✅ Résultat attendu

- Ton site est **accessible en HTTPS** avec un certificat valide
- Tout accès HTTP est **redirigé en HTTPS**
- Tu peux augmenter le nombre de containers WordPress pour gérer plus de charge
- Tu as une **infrastructure prête pour héberger d’autres sites** derrière Traefik
