
# Production Deployment Workflow — Laravel + Docker Compose + PostgreSQL + Nginx (Ubuntu on DigitalOcean)

> **Audience:** Beginners who want a clean, reproducible production deploy for a Laravel app.  
> **Stack:** Ubuntu (DigitalOcean Droplet) · Docker + Docker Compose · PHP-FPM · Nginx · PostgreSQL · optional Redis  
> **Outcome:** Your app runs at `https://your-domain.com` on a VPS with persistent database storage and a straightforward update process.

---

## 0) What you’ll build (high-level)

```
[ Internet ] ──HTTPS──> [ Nginx container ] ──FastCGI──> [ PHP-FPM container (Laravel) ]
                                      │
                                      ├──> [ PostgreSQL container (data volume) ]
                                      └──> [ Redis container (optional, for cache/queues) ]
```

- **Nginx** serves static files and forwards PHP requests to **PHP-FPM** (your Laravel code).
- **PostgreSQL** stores your data (persisted on the VPS disk via a Docker volume).
- (Optional) **Redis** for cache/sessions/queues.
- All services are isolated in containers and reproducible with `docker compose up -d`.

---

## 1) Prerequisites

- A **DigitalOcean Droplet** (Ubuntu 22.04/24.04 LTS) — 1–2 vCPU / 1–2 GB RAM is OK to start.
- A **domain name** you can point to your droplet’s public IP (A record).
- Your **Laravel project** on GitHub/GitLab **or** copied to the server.
- You have **SSH** access to the droplet as root or a sudoer.

> **Tip:** On small droplets, add a 2 GB **swap** file (see §4.5) to reduce out-of-memory issues during builds/composer installs.

---

## 2) Point your domain to the Droplet

In your DNS provider, create an **A** record:

- Host: `@` → **Droplet Public IP**
- (Optional) Host: `www` → **Droplet Public IP**

Wait for DNS to propagate (often minutes).

---

## 3) SSH into the server

From your local machine:

```bash
ssh root@YOUR_DROPLET_IP
# or, if you created a sudo user:
ssh deploy@YOUR_DROPLET_IP
```

---

## 4) One-time server setup (Ubuntu)

### 4.1 Update & basics

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg ufw
```

### 4.2 Install Docker Engine + Compose plugin (official repo)

```bash
# Add Docker’s official GPG key and repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu   $(. /etc/os-release && echo $VERSION_CODENAME) stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add your user to the `docker` group so you can run Docker without `sudo` (log out/in afterward):

```bash
sudo usermod -aG docker $USER
newgrp docker   # refresh group membership in current shell
```

### 4.3 Firewall (UFW)

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw --force enable
sudo ufw status verbose
```

### 4.4 Set timezone (optional)

```bash
sudo timedatectl set-timezone America/Mexico_City
timedatectl
```

### 4.5 Create swap (recommended on small droplets)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 5) Create a home for your app on the server

We’ll keep everything in **`/opt/laravel-app`**:

```bash
sudo mkdir -p /opt/laravel-app
sudo chown -R $USER:$USER /opt/laravel-app
cd /opt/laravel-app
```

- If your code is on GitHub:
  ```bash
  git clone https://github.com/your-user/your-laravel-repo.git .
  ```
- Or upload your project files (e.g., `scp`, SFTP, or Git pull from a private repo).

> Ensure your repository **does not** commit `.env` and other secrets.

---

## 6) Environment variables (`.env`)

Copy the example and edit:

```bash
cp .env.example .env
```

Update these **minimum** values:

```dotenv
APP_NAME="Laravel"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

# Database: use service name "db" as host
DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=CHANGE_ME_STRONG_PWD

# Cache/Session/Queue (start simple; you can switch to Redis later)
CACHE_STORE=file
SESSION_DRIVER=cookie
QUEUE_CONNECTION=database

# Optional: if using Redis
# REDIS_HOST=redis
# REDIS_PORT=6379
```

Generate a strong DB password on the server:

```bash
openssl rand -base64 24
```

Paste that into `DB_PASSWORD`.

> We’ll generate the **APP_KEY** after containers are up.

---

## 7) Docker files

We’ll use this folder structure inside `/opt/laravel-app`:

```
/opt/laravel-app
├─ docker/
│  ├─ php/
│  │  ├─ Dockerfile
│  │  └─ php.ini
│  └─ nginx/
│     └─ default.conf
├─ docker-compose.yml
├─ .env
└─ (your Laravel app files: artisan, app/, bootstrap/, public/, etc.)
```

### 7.1 PHP-FPM Dockerfile — `docker/php/Dockerfile`

```dockerfile
# docker/php/Dockerfile
FROM php:8.3-fpm-alpine

# System deps
RUN apk add --no-cache bash git unzip libpq-dev icu-dev oniguruma-dev libzip-dev

# PHP extensions you typically need for Laravel + PostgreSQL
RUN docker-php-ext-configure intl   && docker-php-ext-install -j$(nproc) intl mbstring pdo pdo_pgsql zip bcmath opcache

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Opcache settings for production
RUN {   echo "opcache.enable=1";   echo "opcache.enable_cli=0";   echo "opcache.jit=tracing";   echo "opcache.jit_buffer_size=64M";   echo "opcache.memory_consumption=256";   echo "opcache.max_accelerated_files=20000"; } > /usr/local/etc/php/conf.d/opcache.ini

WORKDIR /var/www/html
```

### 7.2 PHP ini — `docker/php/php.ini`

```ini
; docker/php/php.ini
memory_limit = 512M
upload_max_filesize = 20M
post_max_size = 25M
max_execution_time = 120
date.timezone = America/Mexico_City
```

### 7.3 Nginx vhost — `docker/nginx/default.conf` (HTTP only first)

> We’ll enable HTTPS in §10. For now, keep it simple to boot the stack.

```nginx
# docker/nginx/default.conf
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    root /var/www/html/public;
    index index.php index.html;

    # Security headers (basic)
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    # Serve static files directly
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass   app:9000; # PHP-FPM service name:port
        fastcgi_index  index.php;
    }

    location ~* \.(jpg|jpeg|png|gif|css|js|ico|svg|woff|woff2|ttf)$ {
        expires 30d;
        access_log off;
    }

    client_max_body_size 25M;
}
```

### 7.4 Compose file — `docker-compose.yml`

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build:
      context: ./docker/php
    container_name: laravel_app
    working_dir: /var/www/html
    env_file: .env
    volumes:
      - ./:/var/www/html
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/custom.ini:ro
    depends_on:
      - db
    restart: unless-stopped
    networks: [internal, web]

  web:
    image: nginx:1.27-alpine
    container_name: laravel_nginx
    ports:
      - "80:80"
      # We'll map 443 later after issuing certs
    volumes:
      - ./:/var/www/html:ro
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      # Later for SSL:
      # - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - app
    restart: unless-stopped
    networks: [internal, web]

  db:
    image: postgres:16-alpine
    container_name: laravel_pg
    environment:
      POSTGRES_DB: ${DB_DATABASE}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped
    networks: [internal]
    # Optional: expose for remote admin (not recommended publicly)
    # ports:
    #   - "5432:5432"

  # Optional: Redis for cache/session/queues
  # redis:
  #   image: redis:7-alpine
  #   container_name: laravel_redis
  #   restart: unless-stopped
  #   networks: [internal]

volumes:
  pgdata:

networks:
  internal:
  web:
```

> **Why mount the code (`.:/var/www/html`) in production?**  
> It’s beginner-friendly for quick edits and updates. A more advanced/immutable approach builds the code *into* the image and avoids host bind-mounts—but this setup keeps learning simple.

---

## 8) First boot

From `/opt/laravel-app`:

```bash
docker compose pull        # fetch images (nginx, postgres)
docker compose build app   # build the PHP image
docker compose up -d       # start everything
```

Set Laravel file permissions (storage & cache):

```bash
docker compose exec -T app bash -lc "chown -R www-data:www-data storage bootstrap/cache && chmod -R ug+rwX storage bootstrap/cache"
```

Generate your **APP_KEY**:

```bash
docker compose exec -T app php artisan key:generate --ansi
```

Run migrations (and seeders if you have them):

```bash
docker compose exec -T app php artisan migrate --force
# docker compose exec -T app php artisan db:seed --force  # if applicable
```

Optimize for production:

```bash
docker compose exec -T app php artisan config:cache
docker compose exec -T app php artisan route:cache
docker compose exec -T app php artisan view:cache
```

Visit: `http://your-domain.com` (HTTP only for now).

---

## 9) (Optional) Use Redis for sessions/cache/queues

1) Start the `redis` service (uncomment in `docker-compose.yml` and `docker compose up -d redis`).  
2) In `.env` set:
```dotenv
CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis
REDIS_HOST=redis
```
3) Clear/refresh caches:
```bash
docker compose exec -T app php artisan optimize:clear
docker compose exec -T app php artisan config:cache
```

---

## 10) Enable HTTPS with Let’s Encrypt (simple & reliable)

**Approach:** Use `certbot` on the host (standalone), obtain certs into `/etc/letsencrypt`, then mount them into the Nginx container and enable the TLS server block.

### 10.1 Obtain certificates (temporarily free port 80)

Stop Nginx so certbot can bind to `:80`:

```bash
docker compose stop web
sudo apt-get install -y certbot
sudo certbot certonly --standalone -d your-domain.com -d www.your-domain.com --agree-tos -m you@example.com --non-interactive
```

Certs will be saved under `/etc/letsencrypt`. Confirm with:

```bash
sudo ls -l /etc/letsencrypt/live/your-domain.com
```

### 10.2 Enable TLS in Nginx

Edit `docker/nginx/default.conf` to add a TLS server and redirect HTTP → HTTPS:

```nginx
# HTTP → redirect to HTTPS
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    return 301 https://$host$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    ssl_certificate     /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    # Basic TLS settings; you can harden further later
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /var/www/html/public;
    index index.php index.html;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include        fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass   app:9000;
        fastcgi_index  index.php;
    }

    location ~* \.(jpg|jpeg|png|gif|css|js|ico|svg|woff|woff2|ttf)$ {
        expires 30d;
        access_log off;
    }

    client_max_body_size 25M;
}
```

Now **mount** the certs directory in `docker-compose.yml` for `web` and expose 443:

```yaml
web:
  image: nginx:1.27-alpine
  container_name: laravel_nginx
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./:/var/www/html:ro
    - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    - /etc/letsencrypt:/etc/letsencrypt:ro   # <── add this
  depends_on:
    - app
  restart: unless-stopped
  networks: [internal, web]
```

Restart Nginx:

```bash
docker compose up -d web
```

Visit: `https://your-domain.com` ✅

### 10.3 Auto-renew

Add a cron for renewals with a reload hook:

```bash
sudo crontab -e
# Add this line (twice daily check):
0 3,15 * * * certbot renew --quiet --deploy-hook "docker compose -f /opt/laravel-app/docker-compose.yml exec -T web nginx -s reload"
```

> If your code lives elsewhere, adjust the compose file path in the hook.

---

## 11) Scheduler (cron) & Queues

### 11.1 Laravel Scheduler

Run the scheduler every minute from the **host** crontab:

```bash
crontab -e
# Add:
* * * * * docker compose -f /opt/laravel-app/docker-compose.yml exec -T app php artisan schedule:run >> /opt/laravel-app/storage/logs/scheduler.log 2>&1
```

### 11.2 Queue Worker (optional)

Add a **queue** service to `docker-compose.yml`:

```yaml
  queue:
    build:
      context: ./docker/php
    container_name: laravel_queue
    working_dir: /var/www/html
    env_file: .env
    command: php artisan queue:work --tries=3 --sleep=1
    volumes:
      - ./:/var/www/html
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/custom.ini:ro
    depends_on:
      - app
      - db
      # - redis   # if you use Redis queue
    restart: unless-stopped
    networks: [internal]
```

Start it:

```bash
docker compose up -d queue
```

---

## 12) Backups

### 12.1 PostgreSQL dump

Create a folder and a simple script:

```bash
sudo mkdir -p /opt/backups && sudo chown $USER:$USER /opt/backups
cat > /opt/laravel-app/backup-db.sh <<"EOF"
#!/usr/bin/env bash
set -euo pipefail
STAMP=$(date +%F_%H-%M-%S)
docker compose -f /opt/laravel-app/docker-compose.yml exec -T db   pg_dump -U ${DB_USERNAME} ${DB_DATABASE} | gzip > /opt/backups/db_$STAMP.sql.gz
echo "Backup done: /opt/backups/db_$STAMP.sql.gz"
EOF
chmod +x /opt/laravel-app/backup-db.sh
```

Run it:

```bash
/opt/laravel-app/backup-db.sh
```

Add a cron (daily at 02:00):

```bash
crontab -e
0 2 * * * /opt/laravel-app/backup-db.sh >> /opt/laravel-app/storage/logs/backup.log 2>&1
```

### 12.2 Restore example

```bash
gunzip -c /opt/backups/db_2025-09-27_12-00-00.sql.gz |   docker compose -f /opt/laravel-app/docker-compose.yml exec -T db psql -U ${DB_USERNAME} -d ${DB_DATABASE}
```

---

## 13) Logs & troubleshooting

- Nginx logs:
  ```bash
  docker compose logs -f web
  ```
- Laravel log:
  ```bash
  docker compose exec -T app tail -f storage/logs/laravel.log
  ```
- PHP-FPM container shell:
  ```bash
  docker compose exec -it app bash
  ```
- Container status:
  ```bash
  docker compose ps
  ```

Common fixes:
- **502 Bad Gateway** → Nginx can’t reach PHP-FPM (`fastcgi_pass app:9000`). Check `app` is running.
- **Permission error** → ensure `storage` and `bootstrap/cache` are writable by `www-data` (see §8).
- **DB connection error** → in `.env`, use `DB_HOST=db`, and confirm the `db` service is healthy.

---

## 14) Updating your app (zero-downtime-ish)

On the server in `/opt/laravel-app`:

```bash
git pull origin main             # or your branch
docker compose exec -T app composer install --no-dev --prefer-dist --no-interaction --optimize-autoloader
docker compose exec -T app php artisan migrate --force
docker compose exec -T app php artisan optimize
docker compose up -d web app     # recreate if configs changed
```

> For very high-traffic apps, consider a blue–green strategy or rolling deployments with a registry. This guide keeps it simple.

---

## 15) Hardening checklist (later, when ready)

- Use a **non-root** sudo user (e.g., `deploy`) and disable root SSH login; use SSH keys.
- Keep Ubuntu & Docker patched (`apt-get upgrade`, rebuild images regularly).
- Don’t expose Postgres to the internet; remove `ports: 5432:5432` from `db` in compose.
- Add **Fail2ban**, **Unattended Upgrades**, strict SSH config (`AllowUsers`, `PasswordAuthentication no`).
- Add a **WAF/CDN** (e.g., Cloudflare) and rate-limiting at Nginx if under attack.
- Store secrets in **dotenv** or a secrets manager; never commit them to Git.

---

## 16) Quick command cheat-sheet

```bash
# Start / stop / restart
docker compose up -d
docker compose stop
docker compose restart web

# Rebuild PHP image after changing Dockerfile
docker compose build app && docker compose up -d app

# Artisan shortcuts
docker compose exec -T app php artisan tinker
docker compose exec -T app php artisan migrate --force
docker compose exec -T app php artisan optimize

# Database shell
docker compose exec -it db psql -U ${DB_USERNAME} -d ${DB_DATABASE}

# View container logs
docker compose logs -f web
docker compose logs -f app
docker compose logs -f db
```

---

## 17) Done!

You have a working **production** deploy of Laravel on a DigitalOcean Ubuntu droplet with Docker Compose, Nginx, and PostgreSQL — with HTTPS, backups, and an update path.

**Next steps (nice-to-haves):**
- Move to **immutable builds** (build the app into the PHP image, no bind-mount).
- Add **CI/CD** (GitHub Actions) to build & deploy on push.
- Add **healthchecks** in `docker-compose.yml` and monitoring (Do Agent, UptimeRobot).
- Use **Redis** for better cache/session/queue performance.
