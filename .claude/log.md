# Server Rebuild Log — wcma.work / testpozi.online
Last updated: 2026-04-08 (session 2)

## Purpose
This file documents every step taken to set up this server. If the server is lost, hand this file to a Claude agent to rebuild everything from scratch.

---

## Agent Instructions
- Read `/home/ssm-user/pref.txt` for communication preferences before starting.
- Address user as Guru.
- Be brief. One recommendation at a time. Numbered lists only.
- No comments in code.

---

## 1. EC2 Instance

- Provider: AWS EC2
- OS: Ubuntu 24.04.4 LTS
- Instance type: t3.medium
- Public IP: 16.176.28.146
- Admin access: AWS Systems Manager Session Manager only (no SSH)

---

## 2. DNS

| Domain | Points to |
|---|---|
| wcma.work | 16.176.28.146 |
| testpozi.online | 16.176.28.146 |
| xyz.wcma.work | 16.176.28.146 |
| wfml.wcma.work | 16.176.28.146 |

---

## 3. Host Hardening (already done — do not redo)

Steps completed in order:

1. `sudo apt update && sudo apt upgrade -y && sudo reboot`
2. UFW enabled: `sudo ufw enable`
3. SSH hardened:
   - root login disabled
   - password auth disabled
   - key-based auth only
4. Fail2ban installed: `sudo apt install fail2ban -y`
5. unattended-upgrades enabled: `sudo apt install unattended-upgrades -y`
6. IAM role `EC2-SSM-Role` created with policy `AmazonSSMManagedInstanceCore` and attached to EC2.
7. SSM Agent confirmed running.
8. Port 22 removed from EC2 security group.
9. IMDSv2 set to required.
10. Docker and Docker Compose installed.
11. `sudo systemctl disable ssh.service ssh.socket`
12. UFW rules: only ports 80 and 443 open publicly.

Current host listening ports: 80, 443 only.

---

## 4. Directory Structure

```
/home/ssm-user/
├── apps/
│   ├── reverse-proxy/
│   │   ├── compose.yaml
│   │   ├── nginx/
│   │   │   ├── nginx.conf
│   │   │   └── conf.d/
│   │   │       └── default.conf
│   │   ├── certbot/
│   │   │   ├── www/          (ACME challenge webroot)
│   │   │   └── conf/         (Let's Encrypt certs)
│   ├── mapnj2/
│   │   ├── compose.yaml
│   │   ├── data/             (SQLite database, owned by UID 1000)
│   │   └── app-src/          (git clone of arifwcma/mapnj2)
│   ├── xyz/
│   │   ├── compose.yaml
│   │   └── app-src/          (git clone of arifwcma/xyz)
│   ├── lizmap/
│   │   ├── compose.yaml
│   │   └── nginx/
│   │       └── lizmap.conf
│   └── qgis-server/
│       ├── compose.yaml
│       ├── projects/
│       │   ├── wimmera_parcels/   (git clone of arifwcma/wimmera_parcels)
│       │   └── wfml/              (wfml.qgs, wfml.qgs.cfg, wfml_attachments.zip)
│       └── data/                  (wfml raster data — pending upload)
├── secrets/
│   └── mapnj2/
│       └── service-account.json  (Google service account, permissions: 600, owner: UID 1000)
└── ubuntu_playground/
    ├── log.md                (this file)
    └── direction.md          (onboarding instructions for new agents)
```

---

## 5. Docker Network

One shared private network for all app containers:

```bash
docker network create internal-apps
```

Nginx is attached to both `reverse-proxy_default` and `internal-apps`.
App containers are attached to `internal-apps` only — no public ports.

---

## 6. Nginx (Reverse Proxy)

File: `/home/ssm-user/apps/reverse-proxy/compose.yaml`

```yaml
services:
  nginx:
    image: nginx:1.28-alpine
    container_name: nginx-reverse-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
      - /tmp
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/www:/var/www/certbot:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    networks:
      - default
      - internal-apps

networks:
  internal-apps:
    external: true
```

File: `/home/ssm-user/apps/reverse-proxy/nginx/nginx.conf`

```nginx
user nginx;
worker_processes auto;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    server_tokens off;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    limit_req_zone $binary_remote_addr zone=general:10m rate=20r/m;
    limit_req_zone $binary_remote_addr zone=qgis:10m rate=20r/m;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    include /etc/nginx/conf.d/*.conf;
}
```

File: `/home/ssm-user/apps/reverse-proxy/nginx/conf.d/default.conf`

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name wcma.work;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name testpozi.online;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name xyz.wcma.work;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name wfml.wcma.work;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name wfml.wcma.work;

    ssl_certificate /etc/letsencrypt/live/wfml.wcma.work/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wfml.wcma.work/privkey.pem;

    location /ows/ {
        limit_req zone=qgis burst=5 nodelay;
        if ($arg_MAP !~ "^/var/www/qgis_projects/wfml/") {
            return 403;
        }
        proxy_pass http://qgis-server:80/ows/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        limit_req zone=general burst=10 nodelay;
        proxy_pass http://lizmap-nginx:80;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name xyz.wcma.work;

    ssl_certificate /etc/letsencrypt/live/xyz.wcma.work/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/xyz.wcma.work/privkey.pem;

    location / {
        limit_req zone=general burst=10 nodelay;
        proxy_pass http://xyz:3002;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name wcma.work;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name testpozi.online;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name xyz.wcma.work;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name xyz.wcma.work;

    ssl_certificate /etc/letsencrypt/live/xyz.wcma.work/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/xyz.wcma.work/privkey.pem;

    location / {
        limit_req zone=general burst=10 nodelay;
        proxy_pass http://xyz:3002;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name testpozi.online;

    ssl_certificate /etc/letsencrypt/live/testpozi.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/testpozi.online/privkey.pem;

    location /cgi-bin/qgis_mapserv.fcgi {
        limit_req zone=qgis burst=5 nodelay;
        if ($arg_MAP != "/var/www/qgis_projects/wimmera_parcels/wimmera_parcels.qgz") {
            return 403;
        }
        rewrite ^/cgi-bin/qgis_mapserv.fcgi$ /ows/ break;
        proxy_pass http://qgis-server:80;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name wcma.work;

    ssl_certificate /etc/letsencrypt/live/wcma.work/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wcma.work/privkey.pem;

    location / {
        limit_req zone=general burst=10 nodelay;
        proxy_pass http://mapnj2:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Start Nginx:
```bash
cd /home/ssm-user/apps/reverse-proxy && docker compose up -d
```

---

## 7. HTTPS Certificates (Let's Encrypt)

Issue certs (run once per domain, HTTP server block must be live first):

```bash
docker run --rm \
  -v /home/ssm-user/apps/reverse-proxy/certbot/conf:/etc/letsencrypt \
  -v /home/ssm-user/apps/reverse-proxy/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email arif@wcma.work \
  --agree-tos \
  --no-eff-email \
  -d wcma.work

docker run --rm \
  -v /home/ssm-user/apps/reverse-proxy/certbot/conf:/etc/letsencrypt \
  -v /home/ssm-user/apps/reverse-proxy/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email arif@wcma.work \
  --agree-tos \
  --no-eff-email \
  -d testpozi.online
```

Current expiry: 2026-07-07 (both domains).

Auto-renew cron (already configured as ssm-user):
```
0 3 * * * docker run --rm -v /home/ssm-user/apps/reverse-proxy/certbot/conf:/etc/letsencrypt -v /home/ssm-user/apps/reverse-proxy/certbot/www:/var/www/certbot certbot/certbot renew --quiet && docker exec nginx-reverse-proxy nginx -s reload
```

To verify cron is set: `crontab -l`

---

## 8. mapnj2 App (Next.js)

Repo: `git@github.com:arifwcma/mapnj2.git`
Location: `/home/ssm-user/apps/mapnj2/app-src`

Clone:
```bash
mkdir -p /home/ssm-user/apps/mapnj2
cd /home/ssm-user/apps/mapnj2
git clone git@github.com:arifwcma/mapnj2.git app-src
```

Create data directory (must be owned by UID 1000 = node user in container):
```bash
mkdir -p /home/ssm-user/apps/mapnj2/data
sudo chown -R 1000:1000 /home/ssm-user/apps/mapnj2/data
```

### Secret

Place Google service account JSON at:
```
/home/ssm-user/secrets/mapnj2/service-account.json
```

Set permissions:
```bash
sudo chmod 700 /home/ssm-user/secrets
sudo chmod 700 /home/ssm-user/secrets/mapnj2
sudo chmod 600 /home/ssm-user/secrets/mapnj2/service-account.json
sudo chown 1000:1000 /home/ssm-user/secrets/mapnj2/service-account.json
```

### Source code fixes applied (already in repo)

1. `app/lib/db.js` line 5 — SQLite path changed to absolute:
   ```js
   const dbPath = '/app/data/database.db'
   ```

2. `app/lib/earthengine.js` — use env var for service account path:
   ```js
   const serviceAccountPath = process.env.GOOGLE_APPLICATION_CREDENTIALS || path.join(process.cwd(), "sensitive_resources", "service-account.json")
   ```

### compose.yaml

File: `/home/ssm-user/apps/mapnj2/compose.yaml`

```yaml
services:
  mapnj2:
    build:
      context: ./app-src
      dockerfile: Dockerfile
    container_name: mapnj2
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: "3001"
      GOOGLE_APPLICATION_CREDENTIALS: /run/secrets/service-account.json
    read_only: true
    tmpfs:
      - /tmp
      - /app/.next/cache
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    volumes:
      - /home/ssm-user/secrets/mapnj2/service-account.json:/run/secrets/service-account.json:ro
      - /home/ssm-user/apps/mapnj2/data:/app/data
    networks:
      - internal-apps

networks:
  internal-apps:
    external: true
```

Build and start:
```bash
cd /home/ssm-user/apps/mapnj2 && docker compose up -d --build
```

---

## 9. xyz App (Next.js)

Repo: `git@github.com:arifwcma/xyz.git`
Location: `/home/ssm-user/apps/xyz/app-src`

Clone:
```bash
mkdir -p /home/ssm-user/apps/xyz
cd /home/ssm-user/apps/xyz
git clone git@github.com:arifwcma/xyz.git app-src
```

Domain: `xyz.wcma.work` → container `xyz` on port 3002.

TLS cert issued for `xyz.wcma.work` (same certbot method as other domains).

### compose.yaml

File: `/home/ssm-user/apps/xyz/compose.yaml`

```yaml
services:
  xyz:
    build:
      context: ./app-src
      dockerfile: Dockerfile
    container_name: xyz
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: "3002"
      GOOGLE_APPLICATION_CREDENTIALS: /run/secrets/service-account.json
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    volumes:
      - /home/ssm-user/secrets/mapnj2/service-account.json:/run/secrets/service-account.json:ro
    networks:
      - internal-apps

networks:
  internal-apps:
    external: true
```

Build and start:
```bash
cd /home/ssm-user/apps/xyz && docker compose up -d --build
```

---

## 10. QGIS Server (testpozi.online)

Repo: `git@github.com:arifwcma/wimmera_parcels.git`
Location: `/home/ssm-user/apps/qgis-server/projects/wimmera_parcels`

Clone:
```bash
mkdir -p /home/ssm-user/apps/qgis-server/projects
cd /home/ssm-user/apps/qgis-server/projects
git clone https://github.com/arifwcma/wimmera_parcels.git wimmera_parcels
```

The `.qgz` project file is in the root of the repo.

### compose.yaml

File: `/home/ssm-user/apps/qgis-server/compose.yaml`

```yaml
services:
  qgis-server:
    image: qgis/qgis-server:ltr
    container_name: qgis-server
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
      - /run
      - /var/cache/nginx
      - /var/lib/nginx
      - /var/log/nginx
    security_opt:
      - no-new-privileges:true
    environment:
      QGIS_SERVER_LOG_LEVEL: "2"
    volumes:
      - ./projects:/var/www/qgis_projects:ro
      - ./data:/data:ro
    networks:
      - internal-apps

networks:
  internal-apps:
    external: true
```

**Important:** `cap_drop: ALL` was removed because QGIS Server's bundled Nginx needs privilege at startup to chown temp dirs. This is a known limitation of the `qgis/qgis-server:ltr` image.

Start:
```bash
cd /home/ssm-user/apps/qgis-server && docker compose up -d
```

### How QGIS routing works

The `qgis/qgis-server:ltr` image does NOT support `/cgi-bin/qgis_mapserv.fcgi` directly.
It uses `/ows/` as the entry point internally.

Our Nginx rewrites the legacy URL to `/ows/` before proxying:
```nginx
location /cgi-bin/qgis_mapserv.fcgi {
    if ($arg_MAP != "/var/www/qgis_projects/wimmera_parcels/wimmera_parcels.qgz") {
        return 403;
    }
    rewrite ^/cgi-bin/qgis_mapserv.fcgi$ /ows/ break;
    proxy_pass http://qgis-server:80;
    ...
}
```

The MAP parameter whitelist is a security measure — it prevents path traversal attacks.

Test WFS endpoint:
```bash
curl -I "https://testpozi.online/cgi-bin/qgis_mapserv.fcgi?MAP=/var/www/qgis_projects/wimmera_parcels/wimmera_parcels.qgz&SERVICE=WFS&VERSION=1.1.0&REQUEST=GetCapabilities"
```

---

## 11. Startup Order (on fresh rebuild)

1. Create Docker network: `docker network create internal-apps`
2. Start Nginx: `cd /home/ssm-user/apps/reverse-proxy && docker compose up -d`
3. Issue certs (see Section 7)
4. Update Nginx config with HTTPS blocks, reload: `docker exec nginx-reverse-proxy nginx -s reload`
5. Start QGIS: `cd /home/ssm-user/apps/qgis-server && docker compose up -d`
6. Build and start mapnj2: `cd /home/ssm-user/apps/mapnj2 && docker compose up -d --build`
7. Build and start xyz: `cd /home/ssm-user/apps/xyz && docker compose up -d --build`

---

## 12. Useful Commands

```bash
docker logs mapnj2 -f
docker logs xyz -f
docker logs qgis-server -f
docker logs nginx-reverse-proxy -f

docker compose restart                        # from app directory
docker compose up -d --build                  # rebuild and restart

docker exec nginx-reverse-proxy nginx -t      # test nginx config
docker exec nginx-reverse-proxy nginx -s reload

crontab -l                                    # view cron jobs
docker network ls
docker ps
```

---

## 13. Security State

1. No public SSH — SSM Session Manager only.
2. UFW enabled, ports 80 and 443 only.
3. IMDSv2 required.
4. Fail2ban installed.
5. unattended-upgrades enabled.
6. Nginx is sole public entry point.
7. App containers have no public ports.
8. mapnj2: read-only filesystem, tmpfs, cap_drop ALL, no-new-privileges.
9. QGIS: read-only mounts, no-new-privileges, internal only.
10. Secrets outside repo, mounted read-only at runtime.
11. MAP parameter whitelisted in Nginx for QGIS endpoint.
12. Auto-renew cron configured for Let's Encrypt.

---

## 14. wfml Project (wfml.wcma.work)

### What was done
- wfml QGIS project files placed at `/home/ssm-user/apps/qgis-server/projects/wfml/`
- TLS cert issued for `wfml.wcma.work` (expiry 2026-07-07)
- Nginx blocks added: HTTP redirect + HTTPS with `/ows/` proxying to qgis-server
- MAP whitelist: `^/var/www/qgis_projects/wfml/`
- Lizmap stack deployed at `/home/ssm-user/apps/lizmap/` (containers: `lizmap`, `lizmap-nginx`) — fate TBD
- Nginx `/` on wfml.wcma.work currently proxies to `lizmap-nginx`

### WMS endpoint (for wfmc Android app)
```
https://wfml.wcma.work/ows/?MAP=/var/www/qgis_projects/wfml/wfml.qgs&SERVICE=WMS&...
```

**Reminder:** Update wfmc base URL from `https://wimmera.xyz/qgis/` to `https://wfml.wcma.work/ows/` and MAP path from `/srv/data/wfml/wfml.qgs` to `/var/www/qgis_projects/wfml/wfml.qgs`.

### Raster data — DONE
Uploaded `data.zip` to S3 bucket `wcma-wfml-data`, then on server:
```bash
aws s3 cp s3://wcma-wfml-data/data.zip /tmp/data.zip
unzip /tmp/data.zip -d /home/ssm-user/apps/qgis-server/projects/wfml/
rm /tmp/data.zip
```
Extracted folders (`depth/`, `wcma_boundary/`) moved into `projects/wfml/data/` to match relative paths in `wfml.qgs`.
WMS endpoint confirmed 200 OK.

---

## 15. SSL Migration — DONE (2026-04-09)

### Goal
Replace all individual per-domain certs with a single wildcard cert `*.wcma.work` (plus apex `wcma.work`).
Also replace `testpozi.online` with `parcels.wcma.work`.
All apps to serve on both HTTP and HTTPS (no redirects).

### IAM permissions added to EC2-SSM-Role
- `AmazonS3ReadOnlyAccess` — for S3 data download
- Inline policy for Route 53 DNS-01 challenge:
  - `route53:ListHostedZones`
  - `route53:GetChange`
  - `route53:ChangeResourceRecordSets`

### Steps completed
- AWS CLI v2 installed on host
- `certbot` and `python3-certbot-dns-route53` installed on host
- Wildcard cert issued via DNS-01 (Route 53): `wcma.work` + `*.wcma.work`, expiry 2026-07-08
- Cert stored at: `certbot/conf/live/wcma.work-0001/`
- Nginx config rewritten: all domains serve HTTP + HTTPS, wildcard cert, testpozi.online → parcels.wcma.work, no redirects
- Old individual certs deleted (wcma.work, testpozi.online, xyz.wcma.work, wfml.wcma.work)
- Old Docker-based certbot cron removed
- Auto-renew via systemd certbot.timer (active + enabled)
- Deploy hook at `/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh` reloads nginx after renewal
- All endpoints confirmed 200 (HTTP + HTTPS)

### Active URLs
| Domain | App |
|---|---|
| wcma.work | mapnj2 (Next.js) |
| xyz.wcma.work | xyz (Next.js) |
| wfml.wcma.work | Lizmap + QGIS Server |
| parcels.wcma.work | QGIS Server (wimmera_parcels) |

---

## 16. Lizmap (wfml.wcma.work) — DONE (2026-04-09)

- Repository `wfml` configured in `lizmapConfig.ini.php` pointing to `/srv/projects/wfml/`
- Public (anonymous) access granted to wfml repository via `jacl2_rights` in `jauth.db`
- Named Docker volumes replaced with bind mounts:
  - `./var` → `/www/lizmap/var`
  - `./www` → `/www/lizmap/www`
- Config is now on-disk and reproducible
- Runtime dirs (db/, log/, sessions/) excluded from git via `var/.gitignore`
- Map URL: `https://wfml.wcma.work/index.php/view/map?repository=wfml&project=wfml`
- Admin panel: `https://wfml.wcma.work/admin.php`
- Admin password changed from default.

### Lizmap plugin fix attempt (2026-04-09) — DID NOT RESOLVE "map cannot be displayed"

Attempted fixes (both applied, both still active):
1. Mounted custom `nginx.conf` at `/home/ssm-user/apps/qgis-server/nginx/nginx.conf` -- changed rewrite from `^/ows/$` to `^/ows/(.*)$` so sub-paths like `/ows/lizmap/server.json` route through FastCGI instead of falling back to static file (was returning 404).
2. Added `QGIS_SERVER_LIZMAP_REVEAL_SETTINGS=true` to qgis-server env vars in `compose.yaml`.

Result: `http://qgis-server:80/ows/lizmap/server.json` now returns 200 with valid JSON (plugin loads, LIZMAP service listed). But the Lizmap UI still shows "This map cannot be displayed."

Do NOT retry these two fixes -- they are already applied and confirmed working. The problem is elsewhere.

---

## 17. Pending Tasks (as of 2026-04-09)

1. Update wfmc Android app URLs (`lib/services/settings_store.dart`): base URL → `https://wfml.wcma.work/ows/`, MAP path → `/var/www/qgis_projects/wfml/wfml.qgs`
2. ~~Change Lizmap admin password from default (admin/admin) at `https://wfml.wcma.work/admin.php`~~ DONE (2026-04-09)
3. AWS Parameter Store for secrets (optional)
4. Terraform — codify full infrastructure (mandatory)
5. Security: Add authentication on QGIS Server
