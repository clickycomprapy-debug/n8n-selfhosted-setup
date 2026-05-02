# n8n Self-Hosted with Traefik + Docker + Cloudflare
Production-ready n8n automation platform running on a hardened Ubuntu 24.04 VPS with Traefik as reverse proxy and Cloudflare for SSL termination.

## Stack
| Component | Version | Role |
|-----------|---------|------|
| Ubuntu | 24.04.4 LTS | Host OS |
| Docker | 29.0.4 | Container runtime |
| Traefik | latest | Reverse proxy + routing |
| n8n | latest | Workflow automation platform |
| PostgreSQL | 15 | Persistent database for n8n |
| Cloudflare | — | SSL termination + DDoS protection + DNS |

## Architecture
```
Internet (HTTPS)
      │
      ▼
 Cloudflare
 ├─ SSL/TLS termination
 ├─ DDoS protection
 ├─ WAF (country blocking + IP allowlist)
 └─ DNS proxy (orange cloud)
      │
      ▼ HTTP (internal)
 UFW Firewall
 └─ Ports 80/443 restricted to Cloudflare IPs / 5678 explicitly denied
      │
      ▼
 Traefik (network_mode: host)
 ├─ Listens on :80 and :443
 ├─ Routes by hostname
 └─ Reads Docker labels for dynamic config
      │
      ▼
 Docker Internal Network
 ├─ n8n (port 5678 internal only)
 └─ PostgreSQL 15 (port 5432 internal only)
```

SSL Strategy: Cloudflare manages TLS externally. The connection between Cloudflare and the server runs over HTTP internally. Port 5678 is blocked at UFW level — n8n is never directly accessible, all traffic must pass through Traefik.

## Project Structure
```
/docker/
├── traefik/
│   ├── docker-compose.yml
│   └── config/
│       └── web.yml
└── n8n-04wk/
    └── docker-compose.yml
```

## Traefik Configuration

### docker-compose.yml
```yaml
services:
  traefik:
    image: traefik:latest
    restart: unless-stopped
    network_mode: host
    command:
      - --api.dashboard=false
      - --api.insecure=false
      - --log.level=INFO
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.file.directory=/config
      - --providers.file.watch=true
      - --accesslog=true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /docker/traefik/config:/config:ro
      - /docker/traefik/certs:/certs:ro
```

Key decisions:
- `network_mode: host` — Traefik binds directly to host network
- `api.dashboard=false` — dashboard disabled in production
- `exposedbydefault=false` — no container exposed unless explicitly opted in
- Docker socket mounted read-only (`:ro`)
- `--accesslog=true` — logs all incoming requests

### config/web.yml
```yaml
http:
  middlewares:
    security-headers:
      headers:
        frameDeny: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "same-origin"

  routers:
    n8n:
      rule: "Host(`n8n.[YOUR-DOMAIN]`)"
      entrypoints: websecure
      tls: {}
      middlewares:
        - security-headers
      service: n8n

  services:
    n8n:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:5678"
```

## n8n + PostgreSQL Configuration

### docker-compose.yml
```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`n8n.[YOUR-DOMAIN]`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls=true"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
    environment:
      - N8N_SECURE_COOKIE=true
      - GENERIC_TIMEZONE=America/Asuncion
      - TZ=America/Asuncion
      - N8N_USER_MANAGEMENT_JWT_SECRET=${N8N_JWT_SECRET}
      - N8N_HOST=n8n.[YOUR-DOMAIN]
      - WEBHOOK_URL=https://n8n.[YOUR-DOMAIN]/
      - N8N_PROTOCOL=https
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=${DB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    container_name: postgres_n8n_prod
    restart: always
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```

## Environment Variables
```bash
# .env — never commit this file
N8N_JWT_SECRET=    # generate: openssl rand -base64 32
DB_USER=n8n
DB_PASSWORD=       # generate: openssl rand -base64 32
```

`.gitignore`:
```
.env
*.env
```

## Deployment Steps
```bash
# 1. Create directory structure
mkdir -p /docker/traefik/config
mkdir -p /docker/n8n-04wk

# 2. Start Traefik
cd /docker/traefik
docker compose up -d

# 3. Configure n8n environment
cd /docker/n8n-04wk
cp .env.example .env
nano .env

# 4. Start n8n + PostgreSQL
docker compose up -d

# 5. Verify
docker ps
```

## Verify Deployment
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
curl -I https://n8n.[YOUR-DOMAIN]
```

## Security Controls
| Control | Implementation |
|---------|---------------|
| n8n not directly exposed | Port 5678 denied at UFW level |
| No default container exposure | exposedbydefault=false |
| Docker socket read-only | :ro mount |
| Secrets not in compose | .env file excluded from version control |
| Dashboard disabled | api.dashboard=false |
| SSL/TLS | Cloudflare |
| Database not exposed | No ports mapping on PostgreSQL |
| Secure cookies | N8N_SECURE_COOKIE=true |
| Strong secrets | Generated with openssl rand -base64 32 |
| HTTP security headers | Via Traefik middleware |
| Cloudflare WAF | Country blocking + IP allowlist |
| Access logging | --accesslog=true |

## Useful Commands
```bash
# View containers
docker ps

# View logs
docker logs n8n-04wk-n8n-1 --tail 50 -f
docker logs traefik-traefik-1 --tail 50 -f

# Restart services
docker compose -f /docker/n8n-04wk/docker-compose.yml restart n8n
docker compose -f /docker/traefik/docker-compose.yml restart traefik

# Update n8n
cd /docker/n8n-04wk
docker compose pull
docker compose up -d

# Check pending updates
sudo apt list --upgradable 2>/dev/null
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.CreatedAt}}"
```

## Backup Strategy
```bash
# n8n volume
tar -zcf n8n-backup-$(date +%F).tar.gz /var/lib/docker/volumes/n8n_data

# Automated (cron root)
0 3 * * * tar -zcf /home/user/backups/n8n-backup-$(date +\%F).tar.gz /var/lib/docker/volumes/n8n_data
0 2 * * * tar -zcf /home/user/backups/docker-config-$(date +\%F).tar.gz /docker
30 3 * * * rclone copy /home/user/backups/ gdrive:backups-server/
0 5 * * * find /home/user/backups -name "*.tar.gz" -mtime +14 -delete
```

Backups follow the 3-2-1 rule — 3 copies, 2 different media, 1 offsite.

## Current Status
| Control | Status |
|---------|--------|
| n8n running | ✅ |
| PostgreSQL running | ✅ |
| Traefik routing | ✅ |
| HTTPS active | ✅ Cloudflare |
| Automated backups | ✅ Daily + Google Drive |
| HTTP security headers | ✅ |
| Cloudflare WAF | ✅ |
| n8n auto-update | ✅ Weekly via cron |
| Access logging | ✅ |
| Secure cookies | ✅ |
| Strong secrets | ✅ |

## Work in Progress
- [ ] WireGuard VPN for remote access
- [ ] Cloudflare Access (Zero Trust) authentication layer
- [ ] mTLS in Traefik
- [ ] Rate limiting middleware

## Related
- linux-server-hardening — Full Ubuntu server hardening guide

---
Maintained by M.M.
