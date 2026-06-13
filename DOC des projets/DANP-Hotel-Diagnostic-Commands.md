# 📚 MEMO — Commandes de diagnostic DANP Hotel

> Boîte à outils pour vérifier l'état de ton projet : ports, Docker, services, réseau, etc.

---

## 📑 Sommaire

- [🐳 Docker — Conteneurs](#-docker--conteneurs)
- [🐙 Docker Compose](#-docker-compose)
- [🌐 Réseau — Ports / Connectivité](#-réseau--ports--connectivité)
- [⚙️ Windows — Process](#️-windows--process)
- [🔐 SSH / VPS Hetzner](#-ssh--vps-hetzner)
- [🧪 MySQL](#-mysql)
- [🌐 Nginx (sur le VPS)](#-nginx-sur-le-vps)
- [🔒 Certificats SSL (Let's Encrypt / Certbot)](#-certificats-ssl-lets-encrypt--certbot)
- [🔍 Git](#-git)
- [🏗️ Jenkins](#️-jenkins)
- [📡 ngrok](#-ngrok)
- [🩺 Diagnostic complet](#-diagnostic-complet--tout-va-bien-)
- [🚨 Patterns d'erreurs courantes](#-patterns-derreurs-courantes)
- [📋 Workflow type "Un truc cloche"](#-workflow-type-un-truc-cloche)
- [💡 Pour aller plus loin](#-pour-aller-plus-loin)

---

## 🐳 Docker — Conteneurs

### Voir ce qui tourne

```powershell
docker ps                           # conteneurs en cours d'exécution
docker ps -a                        # tous les conteneurs (même arrêtés/crashés)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"   # vue compacte
```

### Voir un conteneur en détail

```powershell
docker inspect hotel-backend        # toutes les infos (JSON)
docker port hotel-backend           # mapping de ports
docker stats hotel-backend          # CPU/RAM/réseau en temps réel
```

### Logs

```powershell
docker logs hotel-backend                    # tous les logs
docker logs hotel-backend --tail 50          # 50 dernières lignes
docker logs hotel-backend -f                 # suivi temps réel (Ctrl+C pour sortir)
docker logs hotel-backend --since 5m         # logs des 5 dernières minutes
docker logs hotel-backend 2>&1 | grep -i error           # filtrer les erreurs
docker logs hotel-backend 2>&1 | grep -iE "tomcat|port"  # chercher Tomcat/port
```

### Inspecter l'intérieur d'un conteneur

```powershell
docker exec -it hotel-backend sh             # ouvrir un shell dans le conteneur
docker exec hotel-backend ps aux             # voir les processus tournant dedans
docker exec hotel-backend env                # voir les variables d'env du conteneur
docker exec hotel-backend ls /app            # voir les fichiers
```

### Tester depuis l'intérieur du conteneur

```powershell
docker exec hotel-backend wget -qO- http://localhost:8081/api/rooms
docker exec hotel-backend sh -c "netstat -tlnp 2>/dev/null || ss -tlnp"
```

### Redémarrer / Arrêter

```powershell
docker restart hotel-backend                 # redémarre sans rebuild
docker stop hotel-backend                    # arrête
docker start hotel-backend                   # démarre
docker rm -f hotel-backend                   # supprime de force
```

---

## 🐙 Docker Compose

### Cycle de vie

```powershell
docker compose up -d                # démarre tout (avec image existante)
docker compose up -d --build        # rebuild et démarre (après modif code)
docker compose down                 # arrête + supprime conteneurs (garde volumes)
docker compose down -v              # ⚠️ arrête + WIPE volumes (perd la BDD !)
docker compose restart              # restart tous les services
docker compose restart backend      # restart un service spécifique
docker compose stop                 # arrête sans supprimer
docker compose start                # redémarre
```

### Voir l'état

```powershell
docker compose ps                   # services et leur état
docker compose logs -f              # logs de tous les services
docker compose logs -f backend      # logs d'un service
docker compose logs --tail 30 backend
```

### Inspecter sans démarrer

```powershell
docker compose config               # affiche la config résolue (avec .env appliqué)
docker compose ps --services        # liste les noms de services
```

---

## 🌐 Réseau — Ports / Connectivité

### Qui occupe un port sur ta machine ?

**Windows :**

```powershell
netstat -ano | findstr ":8081"     # qui écoute sur 8081 ?
netstat -ano | findstr "LISTENING" # tous les ports en écoute
tasklist /FI "PID eq 12345"        # quel programme est ce PID ?
```

**Linux (VPS) :**

```bash
sudo netstat -tlnp | grep 8081     # qui écoute sur 8081 ?
sudo ss -tlnp | grep 8081          # version moderne (ss au lieu de netstat)
sudo lsof -i :8081                 # processus qui occupe 8081
```

### Tester si un port répond

```powershell
curl.exe -v http://localhost:8081/api/rooms       # PC
curl -v http://localhost:8081/api/rooms 2>&1 | head -30  # VPS (Linux)
Test-NetConnection localhost -Port 8081           # PowerShell : test TCP
```

### Tester un service HTTPS

```powershell
curl.exe https://api.danpohotel.com/api/rooms
curl.exe -I https://api.danpohotel.com/api/rooms  # juste les headers
```

### Voir le certificat SSL

```powershell
curl.exe -v https://api.danpohotel.com 2>&1 | Select-String "issuer|subject|expire"
```

---

## ⚙️ Windows — Process

```powershell
tasklist                                                # tous les processus
tasklist /FI "PID eq 12345"                            # un PID spécifique
Get-Process | Where-Object {$_.Name -like "*java*"}    # filtrer par nom
Stop-Process -Id 12345 -Force                          # tuer un process
Stop-Process -Name "java" -Force                       # tuer tous les java.exe
```

### Si port Windows réservé (erreur "An attempt was made...")

```powershell
# En PowerShell admin
net stop winnat
net start winnat

# Voir les plages réservées
netsh interface ipv4 show excludedportrange protocol=tcp
```

---

## 🔐 SSH / VPS Hetzner

### Connexion

```powershell
ssh deploy@178.105.82.18
```

### Tester rapidement la connectivité

```powershell
ssh -o ConnectTimeout=5 deploy@178.105.82.18 "echo OK"
```

### Copier des fichiers

```powershell
# Depuis ton PC vers le VPS
scp .\fichier.txt deploy@178.105.82.18:~/

# Depuis le VPS vers ton PC
scp deploy@178.105.82.18:~/fichier.txt .\
```

### Sur le VPS — Vérifier la config GitHub SSH

```bash
ssh -T git@github.com   # doit afficher "Hi AlainDanp! You've successfully authenticated"
cat ~/.ssh/config        # voir la config SSH
```

---

## 🧪 MySQL

### Sur le VPS *(via Docker)*

```bash
# Se connecter à MySQL via le conteneur
docker exec -it hotel-mysql mysql -u root -p

# Une fois dans MySQL :
SHOW DATABASES;
USE danp_hotel;
SHOW TABLES;
DESCRIBE rooms;
SELECT * FROM rooms;
SELECT * FROM users;
SELECT * FROM role;
EXIT;
```

### One-shot queries (sans rentrer dans mysql)

```bash
docker exec hotel-mysql mysql -uroot -p"PASSWORD" -e "USE danp_hotel; SELECT * FROM rooms;"
docker exec hotel-mysql mysql -uroot -p"PASSWORD" -e "SHOW PROCESSLIST;"
docker exec hotel-mysql mysql -uroot -p"PASSWORD" -e "USE danp_hotel; SHOW TABLES;"
```

### Backup BDD

```bash
docker exec hotel-mysql mysqldump -uroot -p"PASSWORD" danp_hotel > backup-$(date +%Y%m%d).sql
```

### Restore BDD

```bash
docker exec -i hotel-mysql mysql -uroot -p"PASSWORD" danp_hotel < backup-20250101.sql
```

### Voir la taille de la BDD

```bash
docker exec hotel-mysql mysql -uroot -p"PASSWORD" -e "
  SELECT table_schema 'Database',
         ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) 'Size (MB)'
  FROM information_schema.tables
  GROUP BY table_schema;"
```

---

## 🌐 Nginx (sur le VPS)

### Statut

```bash
sudo systemctl status nginx
sudo systemctl restart nginx
sudo systemctl reload nginx       # recharge la config sans interruption
sudo systemctl stop nginx
sudo systemctl start nginx
```

### Tester la config

```bash
sudo nginx -t                     # validation syntaxique
sudo nginx -T                     # affiche TOUTE la config résolue
```

### Logs

```bash
sudo tail -50 /var/log/nginx/error.log     # erreurs récentes
sudo tail -50 /var/log/nginx/access.log    # accès récents
sudo tail -f /var/log/nginx/error.log      # suivi temps réel
```

### Voir les fichiers de config

```bash
ls /etc/nginx/sites-enabled/
cat /etc/nginx/sites-enabled/api.danpohotel
cat /etc/nginx/sites-enabled/danpohotel
```

---

## 🔒 Certificats SSL (Let's Encrypt / Certbot)

```bash
sudo certbot certificates              # liste les certificats et leur validité
sudo certbot renew --dry-run           # simule un renouvellement
sudo certbot renew                     # force le renouvellement (rare, c'est auto)
sudo certbot delete                    # supprimer un certificat (rare)
```

### Voir un certificat depuis n'importe où

```powershell
curl.exe -vI https://api.danpohotel.com 2>&1 | Select-String "issuer|subject|expire"
```

---

## 🔍 Git

### État du repo

```powershell
git status                              # fichiers modifiés/à commit
git log --oneline -10                   # 10 derniers commits
git log --oneline --all -10             # 10 derniers de toutes branches
git branch                              # liste les branches
git remote -v                           # URLs des remotes
```

### Comparaisons

```powershell
git diff                                # changements non-stagés
git diff --cached                       # changements stagés
git diff HEAD~1                         # comparer avec le commit précédent
```

### Annuler / Revenir en arrière

```powershell
git restore <fichier>                   # annule les modifs locales d'un fichier
git restore --staged <fichier>          # unstage
git reset --hard HEAD                   # ⚠️ ANNULE TOUTES les modifs locales
git reset --hard HEAD~1                 # ⚠️ Supprime le dernier commit (local)
```

### Synchronisation

```powershell
git pull                                # récupérer les derniers changements
git push                                # envoyer les commits locaux
git fetch --all --prune                 # synchroniser les refs sans merge
```

---

## 🏗️ Jenkins

### URLs

- Interface : http://localhost:8080
- Job hotel-frontend : http://localhost:8080/job/hotel-frontend/
- Job hotel-backend : http://localhost:8080/job/hotel-backend/

### Conteneur Jenkins

```powershell
docker ps | findstr jenkins
docker logs jenkins --tail 100
docker restart jenkins                 # redémarre Jenkins
```

### Sur le VPS — Voir le code que Jenkins a déployé

```bash
ssh deploy@178.105.82.18
cd ~/hotel-backend
git log --oneline -5                    # derniers commits déployés
ls -la
# ⚠️ Ne JAMAIS afficher .env dans un screenshot/log public
```

---

## 📡 ngrok

### Démarrer

```powershell
ngrok http 8080
```

### Voir le statut

- Interface web locale : http://localhost:4040
- Tu y vois les requêtes entrantes en temps réel

### Si l'URL change

1. Note la nouvelle URL ngrok
2. Va sur GitHub → repo backend → Settings → Webhooks → Edit → nouvelle URL
3. Idem pour le repo frontend

---

## 🩺 Diagnostic complet — "Tout va bien ?"

Voici un script PowerShell qui te fait un check rapide. Sauvegarde-le dans un fichier `health-check.ps1` :

```powershell
# health-check.ps1
Write-Host "=== Local Backend ===" -ForegroundColor Cyan
$local = curl.exe -s -o NUL -w "%{http_code}" http://localhost:8081/api/rooms
Write-Host "http://localhost:8081/api/rooms → $local" -ForegroundColor $(if ($local -eq "200") {"Green"} else {"Red"})

Write-Host ""
Write-Host "=== Prod Backend ===" -ForegroundColor Cyan
$prod = curl.exe -s -o NUL -w "%{http_code}" https://api.danpohotel.com/api/rooms
Write-Host "https://api.danpohotel.com/api/rooms → $prod" -ForegroundColor $(if ($prod -eq "200") {"Green"} else {"Red"})

Write-Host ""
Write-Host "=== Prod Frontend ===" -ForegroundColor Cyan
$front = curl.exe -s -o NUL -w "%{http_code}" https://danpohotel.com
Write-Host "https://danpohotel.com → $front" -ForegroundColor $(if ($front -eq "200") {"Green"} else {"Red"})

Write-Host ""
Write-Host "=== Docker containers ===" -ForegroundColor Cyan
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Lance avec :

```powershell
& "C:\chemin\health-check.ps1"
```

---

## 🚨 Patterns d'erreurs courantes

| Erreur | Diagnostic | Solution |
|---|---|---|
| `Connection refused` | Rien n'écoute sur ce port | Vérifier que le service est UP |
| `Connection reset by peer` | Le service écoute mais réponse cassée | Vérifier la **vraie** port d'écoute du service *(netstat dans le conteneur)* |
| `502 Bad Gateway` (Nginx) | Nginx ne peut pas joindre l'upstream | Backend down ou mauvais port |
| `504 Gateway Timeout` | Backend trop lent | Augmenter timeout Nginx |
| `Empty reply from server` | Le serveur accepte la connexion puis la coupe | Service en pleine initialisation OU mauvais port côté serveur |
| `An attempt was made to access a socket...` | Port réservé Windows | `net stop winnat ; net start winnat` |
| `Port is already allocated` | Port déjà utilisé par un autre conteneur | `docker ps` puis stopper le conteneur qui squatte |

---

## 📋 Workflow type "Un truc cloche"

Quand quelque chose ne marche pas, suis cet ordre :

### 1. Le service est-il vivant ?

```powershell
docker ps                           # le conteneur tourne ?
docker logs hotel-backend --tail 50 # quoi dans les logs ?
```

### 2. Le bon port est-il bien écouté ?

```powershell
docker exec hotel-backend sh -c "netstat -tlnp 2>/dev/null || ss -tlnp"
docker port hotel-backend
```

### 3. Le service répond-il de l'intérieur ?

```powershell
docker exec hotel-backend wget -qO- http://localhost:8081/api/rooms
```

### 4. Le port est-il joignable de l'extérieur ?

```powershell
curl.exe -v http://localhost:8081/api/rooms
```

### 5. Le proxy/load balancer est-il OK ?

```bash
# sur le VPS
sudo nginx -t
sudo tail -30 /var/log/nginx/error.log
```

### 6. Le DNS pointe-t-il au bon endroit ?

```powershell
nslookup api.danpohotel.com
nslookup danpohotel.com
```

---

## 💡 Pour aller plus loin

Outils utiles à installer :

- **Postman** ou **Insomnia** : tester des API graphiquement
- **DBeaver** : se connecter aux bases MySQL/Postgres avec une interface
- **VS Code REST Client extension** : créer des fichiers `.http` avec tes requêtes
- **Lens** ou **Portainer** : interface graphique pour gérer Docker
- **Wireshark** : analyser le trafic réseau si problème complexe

---

## 📍 Infos clés du projet

| Élément | Valeur |
|---|---|
| **Domaine frontend** | https://danpohotel.com |
| **Domaine backend** | https://api.danpohotel.com |
| **VPS IP** | 178.105.82.18 |
| **VPS user** | deploy |
| **VPS hostname** | hotel-frontend-prod |
| **Backend port (conteneur)** | 8081 |
| **Backend port (host VPS)** | 127.0.0.1:8081 |
| **MySQL port (interne)** | 3306 (non exposé) |
| **Jenkins port (local)** | 8080 |
| **ngrok URL** | https://reload-ranged-halogen.ngrok-free.dev |
| **GitHub repo frontend** | https://github.com/AlainDanp/Hotel_danp_Frontend |
| **GitHub repo backend** | https://github.com/AlainDanp/Hotel_danp_Backend |

---

## 🎯 Conventions de noms importantes

| Conteneur Docker | Service |
|---|---|
| `hotel-backend` | Spring Boot backend |
| `hotel-mysql` | MySQL database |
| `jenkins` | Jenkins CI/CD (local PC uniquement) |

---

*Document créé pour le projet DANP Hotel — Garde-le précieusement, mets-le à jour si tu changes ton infra.*
