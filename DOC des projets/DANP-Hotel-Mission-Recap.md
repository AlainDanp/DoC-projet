# 📚 Récap complet — Mission DANP Hotel

> Documentation exhaustive de tous les sujets abordés lors du setup DevOps complet du projet.

---

## 📑 Sommaire

- [🏗️ PHASE 1 — Architecture & Setup initial](#️-phase-1--architecture--setup-initial)
- [🌐 PHASE 2 — Domaine, DNS et HTTPS](#-phase-2--domaine-dns-et-https)
- [⚙️ PHASE 3 — CI/CD Frontend (Jenkins)](#️-phase-3--cicd-frontend-jenkins)
- [🐳 PHASE 4 — Dockerisation du Backend](#-phase-4--dockerisation-du-backend)
- [🔐 PHASE 5 — Gestion des secrets](#-phase-5--gestion-des-secrets)
- [🛠️ PHASE 6 — Galère du dev local (debug session)](#️-phase-6--galère-du-dev-local-debug-session)
- [🚀 PHASE 7 — Déploiement Backend sur VPS](#-phase-7--déploiement-backend-sur-vps)
- [🔄 PHASE 8 — CI/CD Backend](#-phase-8--cicd-backend)
- [📱 PHASE 9 — Frontend Angular](#-phase-9--frontend-angular)
- [📋 PHASE 10 — Workflow opérationnel](#-phase-10--workflow-opérationnel)
- [💡 PHASE 11 — Bonnes pratiques et concepts abordés](#-phase-11--bonnes-pratiques-et-concepts-abordés)
- [🎯 État final du projet](#-état-final-du-projet)
- [📝 Suggestion de structure pour la doc](#-suggestion-de-structure-pour-la-doc)

---

## 🏗️ PHASE 1 — Architecture & Setup initial

### Décisions architecturales prises

- Stack technique : Angular 18 (frontend) + Spring Boot 4 Java 17 (backend) + MySQL 8
- Hébergement : VPS Hetzner CPX11 (Falkenstein, Ubuntu 24.04)
- Domaine : `danpohotel.com` chez OVH
- Architecture : monolithe applicatif sur 1 VPS, frontend statique servi par Nginx, backend Spring conteneurisé, MySQL en réseau Docker interne
- Choix : ne pas séparer en plusieurs serveurs *(coût + simplicité)*

### Infrastructure VPS Hetzner

- Création du serveur, première connexion SSH
- Sécurisation : utilisateur `deploy` créé, sudoers restreint, root SSH désactivé, UFW firewall, mises à jour système
- Configuration réseau

---

## 🌐 PHASE 2 — Domaine, DNS et HTTPS

### OVH - Gestion du domaine

- Achat de `danpohotel.com`
- Configuration DNS : records A pour `@`, `www`, et plus tard `api`
- Nettoyage des anciens records pointant vers le parking OVH

### Nginx

- Installation sur le VPS
- Configuration sites-available / sites-enabled
- Reverse proxy pour le backend
- Désactivation du site par défaut
- Configuration multi-domaines (`danpohotel.com` + `api.danpohotel.com`)

### Let's Encrypt / Certbot

- Installation de Certbot
- Génération des certificats SSL
- Redirection automatique HTTP → HTTPS
- Auto-renouvellement vérifié

---

## ⚙️ PHASE 3 — CI/CD Frontend (Jenkins)

### Jenkins en local Docker

- Installation via image `jenkins/jenkins:lts-jdk21`
- Configuration initiale (mot de passe, plugins suggérés)
- Installation du plugin NodeJS
- Configuration tool NodeJS-20

### Webhook GitHub → Jenkins

- Mise en place de ngrok pour exposer Jenkins
- Gestion de l'URL ngrok (problématique du free tier)
- Configuration du webhook GitHub
- Trigger automatique sur push

### Credentials Jenkins

- `GITHUB_CREDENTIALS` (Personal Access Token)
- `SSH_DEPLOY_DANPOHOTEL` (clé SSH pour déploiement)
- `DEPLOY_HOST` (IP du VPS)
- `DEPLOY_USER` (`deploy`)

### Pipeline Frontend (Jenkinsfile)

- Stages : Checkout → Install → Build → Deploy
- Build production Angular avec `--configuration production`
- Déploiement via rsync + ssh
- Reload Nginx automatique
- Gestion des artifacts

### Bugs et résolutions

- Budgets CSS Angular trop bas → ajustés
- Bug du tool NodeJS pointant sur mauvaise version
- Problème `known_hosts` non créé pour Jenkins
- Bug majeur SSH : credential `DEPLOY_USER` contenant `Global` au lieu de `deploy`
- Installation de `libatomic1`, `rsync`, `openssh-client` dans le container Jenkins

---

## 🐳 PHASE 4 — Dockerisation du Backend

### Refactoring du code Spring Boot

- Externalisation des secrets via variables d'env (`${...}`)
- Configuration CORS configurable
- Adaptation `SecurityConfig.java`
- Gestion des profils Spring (et pourquoi on a finalement supprimé `application-prod.properties`)

### Stratégie de gestion du schéma BDD

- Tentative avec Flyway (V1__initial_schema.sql + V2__initial_data.sql)
- Découverte des conflits FK et types de colonnes
- Choix final : Hibernate `ddl-auto=update` + `DataInitializer.java` pour les données initiales
- Refactoring du `DataInitializer` pour seed rôles + rooms + amenities
- Discussion sur "Hibernate vs Flyway" : trade-offs et choix selon le contexte

### Dockerfile

- Multi-stage build (builder + runtime)
- Image Eclipse Temurin Java 17
- User non-root pour la sécurité
- Optimisations JVM pour conteneurs (`UseContainerSupport`, `MaxRAMPercentage`)
- Optimisation du cache Maven via `dependency:go-offline`

### docker-compose.yml

- Services : `backend` + `mysql`
- Réseau interne `hotel-net`
- Volume persistant `db-data` pour MySQL
- Volume `uploads` pour les fichiers
- Healthcheck MySQL avec `mysqladmin ping`
- `depends_on` avec `condition: service_healthy`
- Variables d'env injectées via `.env`
- Port mapping `127.0.0.1:8081:8081` (binding localhost only)

### .dockerignore

- Exclusion `target/`, `.git/`, `.idea/`, `.env`, etc.
- Importance critique : ne JAMAIS embarquer le `.env` dans l'image

---

## 🔐 PHASE 5 — Gestion des secrets

### Incident de sécurité majeur (token GitHub leak)

- Fuite d'un Personal Access Token via screenshot
- Procédure de révocation et regénération
- Bonnes pratiques pour ne jamais avoir le token dans une URL/commande

### Structure `.env` / `.env.example`

- Différenciation rigoureuse : `.env` (secrets, jamais commité) vs `.env.example` (template avec placeholders)
- `.gitignore` avec règle `!.env.example`
- Permissions Linux `chmod 600 .env` sur le VPS

### Secrets gérés

- Mots de passe MySQL (root + user)
- JWT_SECRET (régénération aléatoire base64)
- MAIL_PASSWORD (Gmail app password)
- Bonnes pratiques de génération (commandes PowerShell, `openssl rand`)

---

## 🛠️ PHASE 6 — Galère du dev local (debug session)

### Problèmes rencontrés

- Variables d'env `.env` non chargées par Spring (différent de Node.js)
- Erreur CMD vs PowerShell (`$env:` syntax)
- Tentative création de fonction `Load-DotEnv` PowerShell
- MySQL Windows natif tournant en service système et squattant le port 3306
- Découverte que `mysql.exe` n'est pas dans le PATH

### Solutions trouvées

- Setup IntelliJ avec env vars
- Stratégie : MySQL via Docker + Spring via IntelliJ pour le hot reload
- OU : tout via Docker compose (recommandé)
- Wipe de la base via Docker + commandes SQL

### Conflits de ports Windows

- Port 8081 réservé par Windows (Hyper-V/WinNAT)
- `netsh interface ipv4 show excludedportrange`
- `net stop winnat / net start winnat`
- Identification des processus via `netstat -ano` + `tasklist`
- Multi-projets cohabitant : `big_shop_backend` squattant 8081

---

## 🚀 PHASE 7 — Déploiement Backend sur VPS

### Installation Docker sur VPS

- Méthode officielle Docker pour Ubuntu
- Configuration utilisateur `deploy` dans groupe `docker`
- Test `hello-world`

### Premier déploiement

- Clone du repo via Personal Access Token (avec piège de l'URL exposant le token)
- Création du `.env` de production avec nouveaux secrets
- `docker compose up -d --build`

### Configuration Nginx pour l'API

- Fichier `/etc/nginx/sites-available/api.danpohotel`
- Reverse proxy vers `http://127.0.0.1:8081`
- Headers de proxy importants (`X-Real-IP`, `X-Forwarded-For`, etc.)
- Support WebSocket
- Timeouts
- `client_max_body_size` pour les uploads

### HTTPS pour l'API

- Certbot pour `api.danpohotel.com`
- Redirection HTTP → HTTPS

---

## 🔄 PHASE 8 — CI/CD Backend

### SSH GitHub depuis le VPS

- Génération clé SSH dédiée `github_deploy`
- Ajout sur GitHub Settings → SSH keys
- Configuration `~/.ssh/config`
- Bascule URL git de HTTPS vers SSH
- Test `git pull` sans password

### Pipeline Backend (Jenkinsfile)

- Stages : Checkout → Deploy to VPS → Health Check
- Stratégie : Jenkins SSH au VPS et y exécute `git pull + docker compose up -d --build`
- Pas de build Maven côté Jenkins (délégué au VPS)
- `sshagent` pour gérer la clé

### Health Check adaptatif

- Première version avec `sleep 45` (problématique)
- Refactoring en polling toutes les 5s pendant 3 min max
- Bénéfices : adaptatif au temps de démarrage réel de Spring

### Bug du port (8082 vs 8081)

- Diagnostic : `Connection reset by peer`
- Tcp connect OK mais HTTP cassé
- Identification via `netstat` dans le conteneur : Spring sur 8082, Docker forward sur 8081
- Cause racine : `server.port=${SERVER_PORT:8082}` (mauvais défaut)
- Discussion sur la philosophie "Build once, deploy anywhere" et 12-factor app

---

## 📱 PHASE 9 — Frontend Angular

### Configuration des environments

- `environment.ts` (dev) et `environment.prod.ts` (prod)
- Harmonisation : `apiUrl` avec `/api` cohérent
- Switch automatique au build production

### Bug responsive mobile login form

- Test depuis téléphone réel (même Wi-Fi)
- IP correcte vs IP virtuelle VirtualBox
- Pare-feu Windows et port 4200
- `ng serve --host 0.0.0.0`

### Bug interactif : formulaire login non utilisable sur mobile

- Diagnostic via inspecteur DevTools mobile
- Cause : `transform: translateX` déplace visuel mais pas zone tactile
- Form Register superposé en pointer-events
- Solution : `pointer-events: none` + `opacity: 0` sur le form inactif

### Améliorations CSS mobile

- `min-height` au lieu de `height` fixe
- `overflow: visible` pour scroll natif
- `font-size: 16px` minimum (anti-zoom iOS)
- Tap targets ≥ 44px
- `100dvh` au lieu de `100vh`
- Media queries 992px / 780px / 380px / landscape

---

## 📋 PHASE 10 — Workflow opérationnel

### Routine de redémarrage après reboot PC

- Docker Desktop
- Vérification Jenkins + hotel-backend
- ngrok à relancer (et mise à jour webhook si URL changée)

### Workflow de dev quotidien

- `git push` déclenche le pipeline Jenkins
- Logs Jenkins et VPS pour debug
- Tests locaux avant push

### Commandes de diagnostic

- Docker (containers, logs, exec, ports)
- Réseau (netstat, curl, ports)
- Git (status, log, remote)
- SSH au VPS
- MySQL queries
- Nginx logs et reload

---

## 💡 PHASE 11 — Bonnes pratiques et concepts abordés

### Concepts DevOps

- Single source of truth pour les secrets
- Externalisation de la config (12-factor app)
- Idempotence des déploiements
- Healthchecks et observability
- Reverse proxy pattern
- Réseau Docker interne (isolation des services)
- Versioning des migrations BDD (Flyway) vs auto-generation (Hibernate)

### Concepts sécurité

- Principe du moindre privilège (sudoers, user MySQL dédié)
- Secrets hors du code source
- HTTPS obligatoire en prod
- Ports non exposés (`127.0.0.1` binding)
- SSH keys vs tokens
- Permissions fichiers `.env` à 600

### Bonnes pratiques Git

- Commits atomiques (un sujet par commit)
- Messages de commit descriptifs
- `.gitignore` rigoureux
- Distinction `.env` (secret) vs `.env.example` (template)

---

## 🎯 État final du projet

### Infrastructure opérationnelle

- ✅ Frontend prod sur `https://danpohotel.com`
- ✅ Backend API sur `https://api.danpohotel.com`
- ✅ HTTPS Let's Encrypt avec auto-renew sur les deux
- ✅ MySQL persisté et isolé (non exposé publiquement)
- ✅ Pipeline Jenkins frontend et backend opérationnels
- ✅ Webhooks GitHub configurés
- ✅ Authentification utilisateur fonctionnelle (login + register)
- ✅ Coût mensuel : ~4,50€

### Stack technique finale

- Frontend : Angular 18 + Nginx
- Backend : Spring Boot 4 + Java 17 + JWT + Spring Security
- Base : MySQL 8 en conteneur Docker
- Reverse proxy : Nginx (frontend host)
- CI/CD : Jenkins + GitHub webhooks via ngrok
- Hébergement : Hetzner CPX11 + OVH (domaine + DNS)

---

*Document créé pour le projet DANP Hotel — Récap exhaustif de la mission DevOps.*
