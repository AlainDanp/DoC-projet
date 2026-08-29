# 📙 Guide Spring Boot — Déploiement & CI/CD (Docker, GitHub Actions, Jenkins)

> Guide de référence : passer de « ça marche sur ma machine » à une application déployée automatiquement, du commit jusqu'à la production.
> Version cible : **Spring Boot 3.x** / **Java 17+**

---

## Table des matières

1. [Vue d'ensemble : le pipeline complet](#1-vue-densemble)
2. [Préparer l'application au déploiement](#2-préparer-lapplication)
3. [Docker — conteneuriser l'application](#3-docker)
4. [Docker Compose — orchestrer app + BDD + services](#4-docker-compose)
5. [Migrations BDD avec Flyway (indispensable avant la prod)](#5-flyway)
6. [Git & GitHub — le workflow d'équipe](#6-git--github)
7. [CI/CD avec GitHub Actions](#7-github-actions)
8. [CI/CD avec Jenkins](#8-jenkins)
9. [Déployer sur un serveur (VPS)](#9-déployer-sur-un-serveur)
10. [Gestion des secrets & environnements](#10-secrets--environnements)
11. [Checklist de mise en production](#11-checklist)

---

## 1. Vue d'ensemble

### Définitions

| Terme | Signification |
|---|---|
| **CI** (Continuous Integration) | À chaque push : compiler + tester automatiquement. Détecter les régressions immédiatement. |
| **CD** (Continuous Delivery/Deployment) | Si la CI passe : construire l'image Docker, la publier, et déployer (automatiquement ou sur validation). |
| **Pipeline** | La chaîne d'étapes automatisées : build → test → package → deploy |
| **Artefact** | Le produit du build : le `.jar`, puis l'image Docker |
| **Registry** | Entrepôt d'images Docker (Docker Hub, GitHub Container Registry, AWS ECR...) |

### Le pipeline cible de ce guide

```
   Développeur                    CI (GitHub Actions ou Jenkins)              Serveur
       │                                      │                                 │
  git push ──────▶ ① Checkout du code        │                                 │
       │           ② mvn verify (tests)      │                                 │
       │           ③ Build image Docker      │                                 │
       │           ④ Push image → Registry ──┼──▶ ⑤ Pull de l'image           │
       │                                      │    ⑥ docker compose up -d      │
       │                                      │    ⑦ Healthcheck ✅            │
```

---

## 2. Préparer l'application

Avant de déployer, l'application doit respecter quelques règles.

### 2.1 Actuator — endpoints de santé (obligatoire)

Le déploiement a besoin de savoir si l'app est vivante :

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      probes:
        enabled: true    # active /actuator/health/liveness et /readiness
```

| Endpoint | Rôle |
|---|---|
| `/actuator/health` | L'app est-elle en bonne santé ? (`{"status":"UP"}`) |
| `/actuator/health/liveness` | L'app est-elle vivante ? (sinon → redémarrer le conteneur) |
| `/actuator/health/readiness` | L'app est-elle prête à recevoir du trafic ? |

### 2.2 Externaliser TOUTE la configuration

Le même jar/image doit fonctionner en dev, staging et prod. **Rien d'environnemental en dur** :

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
app:
  jwt:
    secret: ${JWT_SECRET}
```

> 🧠 **Principe "build once, deploy everywhere"** : une seule image Docker, configurée par variables d'environnement selon l'endroit où elle tourne.

### 2.3 Construire le jar

```bash
mvn clean package                 # compile + tests + génère target/mon-app-1.0.0.jar
mvn clean package -DskipTests     # sans les tests (uniquement pour du debug local !)
java -jar target/mon-app-1.0.0.jar   # vérifier qu'il démarre
```

---

## 3. Docker

### 3.1 Concepts en 30 secondes

| Terme | Définition |
|---|---|
| **Image** | Modèle figé contenant l'app + son environnement (JRE, OS minimal) |
| **Conteneur** | Instance en cours d'exécution d'une image |
| **Dockerfile** | Recette de construction de l'image |
| **Registry** | Là où on publie les images (`docker push` / `docker pull`) |
| **Tag** | Version d'une image : `mon-app:1.2.0`, `mon-app:latest` |

### 3.2 Le Dockerfile multi-stage (LA bonne pratique)

Un build en 2 étapes : la 1ère compile avec Maven (lourde), la 2ème ne garde que le nécessaire pour exécuter (légère). Crée ce fichier `Dockerfile` à la racine du projet :

```dockerfile
# ─────────── ÉTAPE 1 : BUILD ───────────
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app

# Copier d'abord le pom seul → les dépendances sont mises en cache
# et ne se retéléchargent que si le pom change (build BEAUCOUP plus rapide)
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Puis copier le code et compiler
COPY src ./src
RUN mvn clean package -DskipTests

# ─────────── ÉTAPE 2 : RUNTIME ───────────
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Sécurité : ne jamais tourner en root dans un conteneur
RUN addgroup -S spring && adduser -S spring -G spring
USER spring

# Récupérer uniquement le jar depuis l'étape de build
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

# Healthcheck intégré au conteneur
HEALTHCHECK --interval=30s --timeout=5s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health | grep -q UP || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

Et un fichier `.dockerignore` (comme `.gitignore`, pour ne pas copier l'inutile dans l'image) :

```
target/
.git/
.idea/
*.md
.env
```

### 3.3 Construire et lancer

```bash
# Construire l'image (le . = contexte de build, dossier courant)
docker build -t mon-app:1.0.0 .

# Lancer un conteneur
docker run -d \
  --name mon-app \
  -p 8080:8080 \
  -e DB_URL=jdbc:postgresql://host.docker.internal:5432/mondb \
  -e DB_USER=postgres \
  -e DB_PASSWORD=postgres \
  -e JWT_SECRET=un-secret-de-32-caracteres-mini \
  mon-app:1.0.0

# Commandes du quotidien
docker ps                    # conteneurs en cours
docker logs -f mon-app       # suivre les logs
docker exec -it mon-app sh   # entrer dans le conteneur
docker stop mon-app && docker rm mon-app
docker images                # lister les images
```

> 💡 `-p 8080:8080` = port machine:port conteneur. `host.docker.internal` permet au conteneur d'atteindre ta machine hôte (utile en dev quand la BDD n'est pas encore dans Docker).

### 3.4 Publier sur un registry

```bash
# Docker Hub
docker login
docker tag mon-app:1.0.0 monpseudo/mon-app:1.0.0
docker push monpseudo/mon-app:1.0.0

# GitHub Container Registry (ghcr.io) — recommandé si ton code est sur GitHub
echo $GITHUB_TOKEN | docker login ghcr.io -u mon-user --password-stdin
docker tag mon-app:1.0.0 ghcr.io/mon-user/mon-app:1.0.0
docker push ghcr.io/mon-user/mon-app:1.0.0
```

### 3.5 Stratégie de tags

| Tag | Usage |
|---|---|
| `1.2.0` | Version précise (immuable) — pour la prod |
| `sha-a1b2c3d` | Commit exact — traçabilité parfaite en CI |
| `latest` | Dernière version — pratique en dev, **à éviter en prod** (non reproductible) |

---

## 4. Docker Compose

Pour lancer **plusieurs conteneurs ensemble** (app + BDD + autres services) avec un seul fichier. Crée `docker-compose.yml` à la racine :

```yaml
services:
  app:
    build: .                       # utilise le Dockerfile local (dev)
    # image: ghcr.io/mon-user/mon-app:1.2.0   # ou une image publiée (prod)
    container_name: mon-app
    ports:
      - "8080:8080"
    environment:
      DB_URL: jdbc:postgresql://postgres:5432/mondb   # 'postgres' = nom du service !
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
      SPRING_PROFILES_ACTIVE: prod
    depends_on:
      postgres:
        condition: service_healthy   # attend que la BDD soit VRAIMENT prête
    restart: unless-stopped

  postgres:
    image: postgres:16
    container_name: mon-postgres
    environment:
      POSTGRES_DB: mondb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data   # les données survivent aux redémarrages
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d mondb"]
      interval: 5s
      timeout: 5s
      retries: 10
    restart: unless-stopped

volumes:
  pgdata:
```

Les variables `${...}` sont lues depuis un fichier `.env` à côté (jamais commité !) :

```bash
# .env  (ajouté au .gitignore !)
DB_USER=postgres
DB_PASSWORD=un-mot-de-passe-fort
JWT_SECRET=une-chaine-secrete-de-32-caracteres-minimum
```

### Commandes Docker Compose

```bash
docker compose up -d          # tout démarrer en arrière-plan
docker compose up -d --build  # reconstruire l'image puis démarrer
docker compose logs -f app    # logs du service 'app'
docker compose ps             # état des services
docker compose down           # tout arrêter
docker compose down -v        # ⚠️ arrêter ET supprimer les volumes (perte des données !)
docker compose pull && docker compose up -d   # mise à jour (nouvelle image)
```

> 🔑 **Point clé réseau** : dans Compose, les conteneurs se joignent **par leur nom de service**. L'app atteint la BDD via `postgres:5432`, pas via `localhost`.

---

## 5. Flyway

### Pourquoi c'est indispensable avant la prod

`spring.jpa.hibernate.ddl-auto: update` laisse Hibernate modifier le schéma tout seul : imprévisible, non versionné, dangereux (il ne supprime jamais rien, gère mal les renommages, et peut corrompre des données). **En production, le schéma doit être versionné comme du code** : c'est le rôle de Flyway.

### Mise en place

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate    # Hibernate VÉRIFIE que le schéma correspond aux entités, sans rien modifier
  flyway:
    enabled: true
    locations: classpath:db/migration
```

### Les scripts de migration

Dans `src/main/resources/db/migration/`, nommage strict : `V<numéro>__<description>.sql` (double underscore !)

```sql
-- V1__create_products_table.sql
CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    description TEXT,
    price       NUMERIC(10,2) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);
```

```sql
-- V2__create_otp_codes_table.sql
CREATE TABLE otp_codes (
    phone       VARCHAR(20) PRIMARY KEY,
    hashed_code VARCHAR(100) NOT NULL,
    expires_at  TIMESTAMP NOT NULL,
    attempts    INT NOT NULL DEFAULT 0
);
```

```sql
-- V3__add_category_to_products.sql
ALTER TABLE products ADD COLUMN category VARCHAR(50);
CREATE INDEX idx_products_category ON products(category);
```

### Comment ça marche

Au démarrage, Flyway regarde la table `flyway_schema_history` (qu'il crée lui-même), compare avec tes scripts, et **applique uniquement les nouveaux**, dans l'ordre. Chaque environnement (dev, staging, prod) converge ainsi vers le même schéma.

### Règles d'or Flyway

- ❌ **Ne JAMAIS modifier un script déjà appliqué** (Flyway le détecte via un checksum et refuse de démarrer) → toujours créer un nouveau `V<n+1>__...sql`
- ✅ Une migration = un changement cohérent et petit
- ✅ Tester chaque migration sur une copie des données avant la prod
- ✅ Penser "compatible ascendant" : ajouter une colonne nullable d'abord, la remplir, puis la rendre NOT NULL dans une migration suivante

---

## 6. Git & GitHub

### Workflow de branches recommandé (GitHub Flow, simple et efficace)

```
main ────●─────────●──────────●────▶   (toujours déployable, protégée)
          \       ↗ \        ↗
           ●──●──●   ●──●───●
         feature/otp  fix/email-bug
```

1. `main` est **toujours stable et déployable**
2. Chaque tâche = une branche : `feature/auth-otp`, `fix/prix-negatif`
3. Travail terminé → **Pull Request** vers `main`
4. La **CI se lance automatiquement** sur la PR (tests)
5. Revue de code par un collègue → merge
6. Le merge sur `main` déclenche le **déploiement**

### Commandes du quotidien

```bash
git checkout -b feature/auth-otp     # créer et basculer sur la branche
git add . && git commit -m "feat(auth): ajout du flux OTP par SMS"
git push -u origin feature/auth-otp  # pousser la branche
# → ouvrir la Pull Request sur GitHub
git checkout main && git pull        # revenir à jour après le merge
```

### Convention de commits (Conventional Commits)

```
feat(auth): ajout de la vérification OTP
fix(products): correction du calcul de remise
refactor(mail): extraction du port NotificationPort
test(otp): tests d'expiration du code
chore(ci): mise à jour de l'action docker/build-push
docs(readme): instructions de lancement
```

### Protéger la branche main

Sur GitHub : **Settings → Branches → Add branch protection rule** sur `main` :
- ✅ Require a pull request before merging
- ✅ Require status checks to pass (la CI doit être verte)
- ✅ Require at least 1 approval

---

## 7. GitHub Actions

GitHub Actions = le CI/CD **intégré à GitHub**. Les pipelines (« workflows ») sont des fichiers YAML dans `.github/workflows/`.

### Vocabulaire

| Terme | Définition |
|---|---|
| **Workflow** | Un pipeline (1 fichier YAML) |
| **Trigger (`on:`)** | Ce qui déclenche le workflow (push, PR, planning...) |
| **Job** | Groupe d'étapes exécuté sur une machine (« runner ») |
| **Step** | Une étape : commande shell ou action réutilisable |
| **Action** | Brique prête à l'emploi du marketplace (`actions/checkout`, `docker/build-push-action`...) |
| **Secret** | Variable chiffrée (Settings → Secrets and variables → Actions) |

### 7.1 Workflow de CI — tests à chaque push et PR

Crée `.github/workflows/ci.yml` :

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    # Une vraie BDD PostgreSQL pour les tests d'intégration
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U test"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10

    steps:
      - name: Checkout du code
        uses: actions/checkout@v4

      - name: Installer Java 21
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven          # cache des dépendances → builds bien plus rapides

      - name: Build + tests
        run: mvn -B clean verify
        env:
          DB_URL: jdbc:postgresql://localhost:5432/testdb
          DB_USER: test
          DB_PASSWORD: test

      - name: Publier le rapport de tests
        if: always()            # même si les tests échouent
        uses: actions/upload-artifact@v4
        with:
          name: rapports-tests
          path: target/surefire-reports/
```

### 7.2 Workflow de CD — build de l'image + déploiement sur merge dans main

Crée `.github/workflows/cd.yml` :

```yaml
name: CD

on:
  push:
    branches: [main]     # uniquement quand main avance (après merge de PR)

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}   # ex: mon-user/mon-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write     # autorise le push vers ghcr.io

    steps:
      - uses: actions/checkout@v4

      - name: Login au registry GitHub
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # fourni automatiquement

      - name: Extraire les tags (sha + latest)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=raw,value=latest

      - name: Build & push de l'image Docker
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha        # cache de build entre les exécutions
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push             # ne se lance que si le build a réussi
    runs-on: ubuntu-latest
    environment: production           # permet d'exiger une validation manuelle

    steps:
      - name: Déployer sur le serveur via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/mon-app
            docker compose pull
            docker compose up -d
            docker image prune -f
```

### 7.3 Les secrets à créer sur GitHub

**Settings → Secrets and variables → Actions → New repository secret** :

| Secret | Contenu |
|---|---|
| `SERVER_HOST` | IP ou domaine du serveur |
| `SERVER_USER` | Utilisateur SSH (ex: `deploy`) |
| `SERVER_SSH_KEY` | Clé SSH **privée** (la clé publique est sur le serveur dans `~/.ssh/authorized_keys`) |

> 💡 **Validation manuelle avant la prod** : dans Settings → Environments → `production`, coche « Required reviewers ». Le job `deploy` attendra alors un clic d'approbation. C'est ça, la différence entre Continuous *Delivery* (validation humaine) et Continuous *Deployment* (100% auto).

---

## 8. Jenkins

Jenkins est l'alternative **auto-hébergée** : même logique que GitHub Actions, mais le serveur de CI t'appartient. Courant en entreprise.

### 8.1 Installer Jenkins (avec Docker, le plus simple)

```bash
docker run -d --name jenkins \
  -p 8090:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk21

# Mot de passe initial :
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Ouvre `http://localhost:8090`, installe les plugins suggérés + **Docker Pipeline**, **Git**, **Pipeline**.

> ⚠️ Le montage de `/var/run/docker.sock` permet à Jenkins de lancer des commandes Docker. Simple pour débuter ; en entreprise on préfère des agents dédiés.

### 8.2 Le Jenkinsfile — pipeline as code

Comme GitHub Actions, le pipeline vit **dans le dépôt** : fichier `Jenkinsfile` à la racine.

```groovy
pipeline {
    agent any

    tools {
        maven 'maven-3.9'        // configuré dans Manage Jenkins → Tools
        jdk   'jdk-21'
    }

    environment {
        REGISTRY   = 'ghcr.io'
        IMAGE_NAME = 'mon-user/mon-app'
        // Credentials stockés dans Jenkins (Manage Jenkins → Credentials) :
        REGISTRY_CREDS = credentials('ghcr-credentials')   // crée REGISTRY_CREDS_USR et _PSW
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm     // clone la branche qui a déclenché le build
            }
        }

        stage('Build & Tests') {
            steps {
                sh 'mvn -B clean verify'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'   // rapport de tests dans Jenkins
                }
            }
        }

        stage('Build image Docker') {
            steps {
                sh "docker build -t ${REGISTRY}/${IMAGE_NAME}:${env.GIT_COMMIT.take(7)} ."
            }
        }

        stage('Push image') {
            when { branch 'main' }        // seulement sur main
            steps {
                sh """
                    echo ${REGISTRY_CREDS_PSW} | docker login ${REGISTRY} -u ${REGISTRY_CREDS_USR} --password-stdin
                    docker push ${REGISTRY}/${IMAGE_NAME}:${env.GIT_COMMIT.take(7)}
                    docker tag ${REGISTRY}/${IMAGE_NAME}:${env.GIT_COMMIT.take(7)} ${REGISTRY}/${IMAGE_NAME}:latest
                    docker push ${REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Validation avant prod') {
            when { branch 'main' }
            steps {
                input message: 'Déployer en production ?', ok: 'Oui, déployer'
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sshagent(credentials: ['server-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no deploy@mon-serveur.com '
                            cd /opt/mon-app &&
                            docker compose pull &&
                            docker compose up -d &&
                            docker image prune -f
                        '
                    """
                }
            }
        }
    }

    post {
        failure {
            echo '❌ Pipeline en échec'
            // mail to: 'equipe@monentreprise.com', subject: "Échec build ${env.JOB_NAME}"
        }
        success {
            echo '✅ Pipeline réussi'
        }
    }
}
```

### 8.3 Connecter Jenkins à GitHub

1. **Créer le job** : New Item → *Multibranch Pipeline* → source Git = URL du dépôt. Jenkins détecte automatiquement le `Jenkinsfile` sur chaque branche et chaque PR.
2. **Déclencher à chaque push** : dans GitHub → Settings du repo → Webhooks → Add webhook → URL : `http://ton-jenkins:8090/github-webhook/` → événements : push + pull request.
3. **Credentials** : Manage Jenkins → Credentials → ajouter le token GitHub, les identifiants du registry, la clé SSH du serveur.

### 8.4 GitHub Actions vs Jenkins — lequel choisir ?

| Critère | GitHub Actions | Jenkins |
|---|---|---|
| Hébergement | Cloud GitHub (zéro maintenance) | Ton serveur (maintenance à ta charge) |
| Mise en route | Minutes | Heures (install, plugins, agents) |
| Coût | Gratuit (quota généreux, illimité en public) | Serveur à payer, logiciel gratuit |
| Intégration GitHub | Native | Via webhooks + plugins |
| Personnalisation | Marketplace d'actions | Immense écosystème de plugins |
| Cas d'usage type | Projets sur GitHub, équipes petites/moyennes | Entreprises, infra interne, besoins spécifiques |

> 🏆 **Recommandation** : si ton code est sur GitHub, commence par **GitHub Actions**. Apprends Jenkins car tu le croiseras en entreprise — et le `Jenkinsfile` te permet de traduire facilement un pipeline de l'un vers l'autre.

---

## 9. Déployer sur un serveur

Scénario le plus courant pour débuter : un **VPS Linux** (OVH, Hetzner, DigitalOcean, AWS EC2...).

### 9.1 Préparation du serveur (une seule fois)

```bash
# Se connecter
ssh root@ip-du-serveur

# Créer un utilisateur de déploiement (jamais travailler en root)
adduser deploy
usermod -aG docker deploy      # après avoir installé Docker

# Installer Docker
curl -fsSL https://get.docker.com | sh

# Copier ta clé publique SSH pour la CI
mkdir -p /home/deploy/.ssh
echo "ssh-ed25519 AAAA... ta-cle-publique" >> /home/deploy/.ssh/authorized_keys

# Pare-feu minimal
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### 9.2 Structure sur le serveur

```
/opt/mon-app/
├── docker-compose.yml     # version prod (image du registry, pas de build)
└── .env                   # secrets de prod (chmod 600, jamais dans Git)
```

Le `docker-compose.yml` de prod référence l'image publiée :

```yaml
services:
  app:
    image: ghcr.io/mon-user/mon-app:latest
    ports:
      - "8080:8080"
    env_file: .env
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
  postgres:
    # ... identique au §4
```

Le déploiement (fait par la CI) se résume alors à :

```bash
cd /opt/mon-app && docker compose pull && docker compose up -d
```

### 9.3 Reverse proxy + HTTPS avec Nginx et Let's Encrypt

Ne jamais exposer l'app directement : un reverse proxy gère le HTTPS et masque le port 8080.

```bash
apt install nginx certbot python3-certbot-nginx
```

```nginx
# /etc/nginx/sites-available/mon-app
server {
    server_name api.mondomaine.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/mon-app /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d api.mondomaine.com    # HTTPS automatique + renouvellement
```

Ton API est maintenant sur `https://api.mondomaine.com` 🎉

### 9.4 Et au-delà du VPS ?

| Option | Quand |
|---|---|
| **PaaS** (Railway, Render, Fly.io, Heroku) | Prototype/side-project : tu pushes, ils déploient. Zéro serveur à gérer. |
| **Kubernetes** (EKS, GKE, AKS) | Nombreux microservices, scaling automatique, grande équipe. Grosse marche d'apprentissage — n'y va que quand Docker Compose ne suffit plus. |

---

## 10. Secrets & environnements

### Règles absolues

1. ❌ Jamais de secret dans le code, le yml versionné, le Dockerfile ou l'historique Git
2. ✅ `.env` dans le `.gitignore`, `chmod 600` sur le serveur
3. ✅ Secrets de CI dans le coffre de l'outil (GitHub Secrets / Jenkins Credentials)
4. ✅ Des secrets **différents par environnement** (le JWT_SECRET de dev ≠ celui de prod)
5. ✅ Rotation en cas de fuite (et purge de l'historique Git avec `git filter-repo` si commité par erreur)

### Cartographie des environnements

| Environnement | Où | Config | Déclenchement |
|---|---|---|---|
| **dev** (local) | Ta machine | `.env` local + profil `dev` | manuel |
| **CI** | Runner GitHub/Jenkins | secrets CI + services éphémères | chaque push/PR |
| **staging** | Serveur de test | `.env` staging | merge sur `main` |
| **production** | Serveur prod | `.env` prod | validation manuelle |

---

## 11. Checklist

### Avant le premier déploiement

- [ ] Actuator branché, `/actuator/health` répond UP (§2.1)
- [ ] Toute la config externalisée en variables d'environnement (§2.2)
- [ ] `ddl-auto: validate` + migrations Flyway versionnées (§5)
- [ ] `Dockerfile` multi-stage + `.dockerignore` (§3.2)
- [ ] `docker compose up -d` fonctionne en local avec la BDD (§4)
- [ ] `.env` dans le `.gitignore`, aucun secret dans Git (§10)

### Le pipeline

- [ ] CI sur chaque PR : build + tests (avec BDD de test) (§7.1)
- [ ] Branche `main` protégée : PR + CI verte obligatoires (§6)
- [ ] CD sur merge dans `main` : build image → push registry → deploy (§7.2 ou §8)
- [ ] Validation manuelle avant la production
- [ ] Healthcheck vérifié après déploiement

### Le serveur

- [ ] Utilisateur `deploy` dédié, connexion par clé SSH uniquement (§9.1)
- [ ] Pare-feu actif (SSH + 80 + 443 seulement)
- [ ] Nginx en reverse proxy + HTTPS Let's Encrypt (§9.3)
- [ ] `restart: unless-stopped` sur les services + volumes pour les données
- [ ] Sauvegardes régulières de la BDD (`pg_dump` planifié)

---

*Guide n°3 de la série. Complète les guides « API & Clean Architecture » et « Annotations & Tests ».*
