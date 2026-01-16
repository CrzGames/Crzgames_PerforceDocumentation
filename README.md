# Crzgames - Perforce Documentation

## Setup l'environnement pour HELIX SWARM (UI WEB) :
1. Go to connect SSH to VPS.
2. Download and Install Docker in VPS : https://docs.docker.com/engine/install/debian/ (for debian)
3. Download and Install NGINX in VPS :
   ```bash
   sudo apt update
   # module stream à ajouté pour TCP pur important !
   sudo apt install -y nginx libnginx-mod-stream
   ```
4. Go to directory : /etc/nginx/sites-available
5. Modify conf in NGINX (/etc/nginx/sites-available/default) :
   ```bash
    server {
       # après avoir générer la commande certbot/nginx remplacer "_" par le nom de domaine
       server_name _;
    
       location / {
        proxy_pass http://localhost:3000/;
       }
    
       listen [::]:443 ssl ipv6only=on; # managed by Certbot
       listen 443 ssl; # managed by Certbot
       http2 on;
    }
    
    server {
      server_name swarm.crzcommon.com;
    
      listen 80;
      listen [::]:80;
    
      location ^~ /.well-known/acme-challenge {
        allow all;
        default_type "text/plain";
        root /var/www/html;
      }
    
      location / {
        return 301 https://$host$request_uri;
      }
    }
   ```
6. Download and Install certbot for certificat HTTPS (il faudra surement retourné au fichier /etc/nginx/sites-availables/conf pour checker si c'est ok) :
   ```bash
   sudo apt-get update
   sudo apt-get install certbot python3-certbot-nginx

   sudo certbot --nginx -d swarm.crzcommon.com

   sudo apt-get install cron
   sudo systemctl enable cron
   sudo systemctl start cron
   
   sudo crontab -e
   0 */12 * * * certbot renew --quiet

   sudo nginx -t
   sudo systemctl reload nginx
   ```
7. Fichier docker-compose.yaml (helix swarm + redis) pour Perforce :
  ```bash
   services:
     helix-swarm:
       image: perforce/helix-swarm:latest
       container_name: helix-swarm
       ports:
         - "3000:80"
       volumes:
         - swarm-data:/opt/perforce/swarm/data
         - swarm-www:/var/www
       environment:
         # Configuration Helix Swarm
         - SWARM_HOST=swarm.crzcommon.com
         - SWARM_PORT=80
         - SWARM_USER=swarm
         - SWARM_PASSWD=xxxxxx (récupérer le mot de passe sur 1Password)
   
         # Identifiants superuser du serveur Perforce (obligatoire pour setup Helix Swarm)
         - P4D_SUPER=crzgames
         - P4D_SUPER_PASSWD=xxxxxx (récupérer le mot de passe sur 1Password)
   
         # Connexion au serveur Perforce
         - P4D_PORT=ssl:perforce.crzcommon.com:1667 # (ATTENTION: P4D_PORT = P4PORT COMPLET)
   
         # Connexion à Redis
         - SWARM_REDIS=helix-redis
         - SWARM_REDIS_PORT=6379
       depends_on:
         - helix-redis
       restart: unless-stopped
   
     helix-redis:
       image: redis:alpine
       container_name: helix-redis
       volumes:
         - redis-data:/data
       restart: unless-stopped
   
   volumes:
     swarm-data:
     swarm-www:
     redis-data:
  ```
8. Run le docker compose :
```bash
# -d = pour lancer en arrière plan
sudo docker compose up -d
```

<br /><br /><br /><br />

## Setup l'environnement pour le serveur PERFORCE :
1. Go to connect SSH to VPS.
2. Générer les certificat TLS/SSL pour le domaine "perforce.crzcommon.com" pour le serveur Perforce qui attends du TCP pur :
   ```bash
   sudo apt-get update
   sudo apt-get install -y certbot
   sudo certbot certonly --standalone -d perforce.crzcommon.com

   # Pour renouveller les certificat automatiquement
   sudo apt-get install cron
   sudo systemctl enable cron
   sudo systemctl start cron

   # Ajouter au fichier crontab la commande 
   sudo crontab -e
   0 */12 * * * certbot renew --quiet
   ```
3. Ajouter cela en bas du fichier (/etc/nginx/nginx.conf) :
   ```bash
   stream {
     server {
          listen 1667 ssl;  # TLS-terminated (clients connectent avec ssl:domaine:1667)
          server_name perforce.crzcommon.com;  # Optionnel, mais utile pour SNI si multi-domaines
  
          # Certs Let's Encrypt (même que pour HTTP, mais réutilisables pour stream)
          ssl_certificate /etc/letsencrypt/live/perforce.crzcommon.com/fullchain.pem;
          ssl_certificate_key /etc/letsencrypt/live/perforce.crzcommon.com/privkey.pem;
          ssl_protocols TLSv1.2 TLSv1.3;
          ssl_ciphers HIGH:!aNULL:!MD5;
          ssl_session_cache shared:SSL:10m;  # Cache sessions pour perf
          ssl_session_timeout 10m;
  
          # Proxy vers backend (clair par défaut)
          proxy_pass localhost:1667; # port du serveur perforce
      }
    }
   ```
4. Restart le serveur nginx :
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```
5. Installer et lancer le serveur PERFORCE (activé l'option unicode a true lors de l'installation, si utilisé avec helix swarm) : https://help.perforce.com/helix-core/quickstart/current/Content/quickstart/admin-install-linux.html

<br /><br /><br /><br />

# 📘 Perforce (Helix Core) – Connexion Unreal Engine

Ce guide explique **comment se connecter au serveur Perforce du projet Unreal Engine**, depuis zéro.

---

## 🔧 Prérequis

### Installer le client Perforce (OBLIGATOIRE)

Télécharger et installer **Helix Command-Line Client (P4)** :

https://portal.perforce.com/s/downloads?product=Helix%20Command-Line%20Client%20%28P4%29

Après installation, vérifier dans un terminal (PowerShell / CMD) :
```bash
p4 -V
```
Si une version s’affiche → ✅ OK

---

## 👤 Création d’un compte Perforce (ADMIN UNIQUEMENT)

❗ Les utilisateurs **ne peuvent PAS se créer un compte eux-mêmes**.  
Un **administrateur Perforce (superuser)** doit créer chaque compte.

### Exemple avec l’admin/superuser `crzgames` du serveur PEFORCE
```bash
# Se connecter au compte superuser
p4 -p ssl:perforce.crzcommon.com:1667 -u crzgames login
# Saisir le mot de passe du superuser

# Choisir un nom pour le nouvelle utilisateur
p4 -p ssl:perforce.crzcommon.com:1667 -u crzgames user -f prenom_du_collegue

# Choisir un mot de passe pour le nouvelle utilisateur (en spécifiant le nom du user choisi juste avant)
p4 -p ssl:perforce.crzcommon.com:1667 -u crzgames passwd prenom_du_collegue  
```

---

## 🛠️ Création d’un groupe à timeout illimité et ajout des nouveaux utilisateurs au groupe (ADMIN UNIQUEMENT)

Ces commandes doivent être exécutées par un **superuser Perforce**  
(exemple : `crzgames`).

### 1️⃣ Créer ou éditer le groupe `always_on`

```bash
p4 -p ssl:perforce.crzcommon.com:1667 -u crzgames group always_on
```
Un éditeur texte s’ouvre.

### 2️⃣ Configurer le groupe
Dans l’éditeur, remplir ou modifier comme suit :
```bash
Group: always_on

Timeout: unlimited

Users:
        toto
        corentin
```
- Timeout: unlimited → ticket sans expiration
- Users → liste des utilisateurs (un par ligne)

Sauvegarder puis fermer l’éditeur.

---

## 🔐 Première connexion (UTILISATEUR)

### Accepter le certificat SSL (une seule fois)
```bash
p4 -p ssl:perforce.crzcommon.com:1667 trust
```
Répondre `yes`.

### Se connecter
```bash
p4 -p ssl:perforce.crzcommon.com:1667 -u nom_du_user_choisi_auparavant login -s
# Puis il vous demanderas de saisir votre mot de passe juste après cette commande.
# SI JAMAIS il vous renvoie : "Perforce password (P4PASSWD) invalid or unset", c'est sûrement que le nom d'utilisateur n'est pas bon ou jamais créer.
```
Vérifier :
```bash
p4 login -s
```

---

## 🎮 Connexion dans Unreal Engine

Source Control (en bas droite de l'éditeur) → chosir le type de controle de version : Perforce

Saisir : 
- Server : ssl:perforce.crzcommon.com:1667
- User : Prenom

Cliquer **Connect**.

---

## 🧠 Bonnes pratiques

- Admin (crzgames) → uniquement pour gérer users / groupes
- User (Corentin) → travailler dans UE
- 1 workspace = 1 projet UE
