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
      environment:
        - SWARM_HOST=helix-swarm
        - SWARM_PORT=80
  
        # IMPORTANT: mot de passe admin du service : Helix Swarm
        - SWARM_PASSWD=CHANGE_ME_STRONG
  
        # Identifiants superuser du serveur Perforce (obligatoire pour setup Helix Swarm)
        - P4D_SUPER=crzgames
        - P4D_SUPER_PASSWD=Marylene59!!!
  
        # Connexion au serveur Perforce
        - P4D_PORT=ssl:perforce.crzcommon.com:1667 # (ATTENTION: P4D_PORT = P4PORT COMPLET)
  
        # Redis
        - REDIS_HOST=redis
        - REDIS_PORT=6379
  
        # Token (un UUID)
        - SWARM_TOKEN=CHANGE_ME_UUID
      depends_on:
        - redis
      restart: unless-stopped
  
    redis:
      image: redis:alpine
      container_name: redis
      volumes:
        - redis-data:/data
      restart: unless-stopped
  
  volumes:
    swarm-data:
    redis-data
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
