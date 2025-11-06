Objectif du projet

Mettre en place une chaîne de déploiement complète pour une application Node.js + React (Vite) hébergée sur un serveur VPS Debian, avec automatisation via GitHub Actions (CI/CD) et gestion des processus avec PM2.

vps-projet/
├── admin-dashboard/      
│   ├── dist/             
│   └── src/              
│
├── backend-API/          
│   ├── index.js          
│   └── .env              
│
└── .github/workflows/    
    └── deploy.yml


Environnement serveur

Serveur : Debian 12
Utilisateur déploiement : deploy
Dossier de travail : /var/www/vps-projet

Outils installés :

Node.js v24.x

npm

git

pm2

nginx

tapes principales du déploiement
1. Connexion et configuration du serveur

ssh root@<ip_vps>
adduser deploy
usermod -aG sudo deploy

Installation de Node, npm, git et PM2 :

sudo apt update && sudo apt install -y nodejs npm git
sudo npm install -g pm2

Création de l’arborescence :

sudo mkdir -p /var/www
sudo mv /root/.ssh/vps-projet /var/www/
sudo chown -R deploy:deploy /var/www/vps-projet

2. Lancement des applications

Backend :

cd /var/www/vps-projet/backend-API
npm install
pm2 start index.js --name backend

Frontend :

cd /var/www/vps-projet/admin-dashboard
npm install
npm run build
pm2 serve dist 8080 --spa --name frontend

3. Configuration de Nginx

Fichier /etc/nginx/sites-available/frontend.conf :

server {
    listen 80;
    server_name 185.98.137.86;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

sudo ln -s /etc/nginx/sites-available/frontend.conf /etc/nginx/sites-enabled/
sudo systemctl reload nginx

Sécurisation SSH & GitHub Actions

Génération d’une clé SSH locale :
ssh-keygen -t rsa -b 4096 -C "github-deploy"

Ajout de la clé publique sur le VPS :

type $env:USERPROFILE\.ssh\id_rsa.pub | ssh deploy@<ip_vps> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

Ajout des secrets GitHub :

VPS_HOST → IP du serveur

VPS_USER → deploy

VPS_SSH_KEY → contenu de la clé privée

VPS_PATH → /var/www/vps-projet

5. Déploiement automatique avec GitHub Actions

.github/workflows/deploy.yml :

name: Deploy Frontend & Backend to VPS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Build frontend
        run: |
          cd admin-dashboard
          npm install
          npm run build

      - name: Deploy to VPS via rsync
        run: |
          echo "$SSH_KEY" > private_key.pem
          chmod 600 private_key.pem
          rsync -avz -e "ssh -i private_key.pem -o StrictHostKeyChecking=no" \
          admin-dashboard/dist/ $VPS_USER@$VPS_HOST:$VPS_PATH/admin-dashboard/dist/
        env:
          SSH_KEY: ${{ secrets.VPS_SSH_KEY }}
          VPS_USER: ${{ secrets.VPS_USER }}
          VPS_HOST: ${{ secrets.VPS_HOST }}
          VPS_PATH: ${{ secrets.VPS_PATH }}

      - name: Restart services on VPS
        run: |
          ssh -i private_key.pem -o StrictHostKeyChecking=no $VPS_USER@$VPS_HOST "
            pm2 restart all
          "


Résultat final

 Frontend : http://185.98.137.86

 Backend : http://185.98.137.86:3000

 Déploiement automatique à chaque push sur main
 PM2 redémarre automatiquement les services après chaque déploiement

Points techniques maîtrisés

Gestion d’un serveur Linux Debian

Configuration Nginx en reverse proxy

Déploiement d’applications Node.js & React

Utilisation de PM2 pour la mise en production

Automatisation via GitHub Actions et SSH sécurisé

Gestion des variables d’environnement (.env)

 Auteur

Clément Boscher
B3 Développement – MDS
Déploiement réalisé dans le cadre du projet “Theo BackFront”
Date : Novembre 2025