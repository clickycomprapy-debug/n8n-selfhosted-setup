Self-Hosted n8n with Traefik + Docker + Cloudflare
> Production-ready n8n automation platform running on a hardened Ubuntu 24.04 VPS with Traefik as reverse proxy and Cloudflare for SSL termination.
This setup powers True Recovery — a remote automation studio helping small and mid-sized businesses in Paraguay modernize their collections and operational workflows.
---
Stack
Component	Version	Role
Ubuntu	24.04.4 LTS	Host OS
Docker	29.0.4	Container runtime
Traefik	latest	Reverse proxy + routing
n8n	latest	Workflow automation platform
PostgreSQL	15	Persistent database for n8n
Cloudflare	—	SSL termination + DDoS protection + DNS
---
Architecture
```
Internet (HTTPS)
      │
      ▼
 Cloudflare
 ├─ SSL/TLS termination
 ├─ DDoS protection
 └─ DNS proxy (orange cloud)
      │
      ▼ HTTP (internal — Cloudflare to server)
 UFW Firewall
 └─ Ports 80, 443 open / 5678 explicitly denied
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
SSL strategy: Cloudflare handles TLS externally. The connection between Cloudflare and the server is HTTP — this is intentional. n8n is configured with `N8N_PROTOCOL=https` because from the end user's perspective, the connection is always HTTPS via Cloudflare. Port 5678 is blocked at the UFW level so n8n is never directly accessible — all traffic must go through Traefik.
---
Project Structure
```
/docker/
├── traefik/
│   ├── docker-compose.yml
│   └── config/
│       └── web.yml              # Static file-based routing config
└── n8n-04wk/
    └── docker-compose.yml       # n8n + PostgreSQL stack
```
---
Traefik Configuration
docker-compose.yml
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
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /docker/traefik/config:/config:ro
```
Key decisions:
`network_mode: host` — Traefik binds directly to the host network, required for routing to work cleanly with Docker services
`api.dashboard=false` and `api.insecure=false` — Dashboard disabled in production
`exposedbydefault=false` — No container is exposed unless it explicitly opts in via labels
Docker socket mounted as read-only (`:ro`) — Traefik can watch Docker events but cannot modify containers
---
n8n + PostgreSQL Configuration
docker-compose.yml
```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n-04wk.rule=Host(`n8n.[YOUR-DOMAIN]`)
      - traefik.http.routers.n8n-04wk.entrypoints=web
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
Note: Sensitive values (JWT secret, DB credentials) are stored in a `.env` file that is never committed to version control. See `.env.example` below.
---
Environment Variables
Create a `.env` file in the same directory as `docker-compose.yml`:
```bash
# .env — never commit this file
N8N_JWT_SECRET=your-long-random-secret-here
DB_USER=n8n
DB_PASSWORD=your-strong-db-password-here
```
.env.example (safe to commit)
```bash
# Copy this to .env and fill in your values
N8N_JWT_SECRET=
DB_USER=n8n
DB_PASSWORD=
```
.gitignore
```
.env
*.env
```
---
Deployment
Prerequisites
Ubuntu 24.04 server with Docker installed
Domain pointed to your server via Cloudflare (proxy enabled — orange cloud)
UFW configured (ports 80, 443 open — port 5678 denied)
SSH access configured (see linux-server-hardening)
Steps
```bash
# 1. Clone or create directory structure
mkdir -p /docker/traefik/config
mkdir -p /docker/n8n-04wk

# 2. Create Traefik compose file
nano /docker/traefik/docker-compose.yml
# (paste Traefik config from above)

# 3. Start Traefik
cd /docker/traefik
docker compose up -d

# 4. Create n8n environment file
cd /docker/n8n-04wk
cp .env.example .env
nano .env
# (fill in your values)

# 5. Start n8n + PostgreSQL
docker compose up -d

# 6. Verify all containers running
docker ps
```
Verify deployment
```bash
# All 3 containers should be running
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Expected output:
# traefik-traefik-1      Up X days
# n8n-04wk-n8n-1         Up X days    5678/tcp
# postgres_n8n_prod      Up X days    5432/tcp

# Note: 5678 shows as internal only — not published to host
```
---
Security Considerations
Control	Implementation
n8n not directly exposed	Port 5678 denied at UFW level
No default container exposure	`exposedbydefault=false` in Traefik
Read-only Docker socket	`:ro` mount prevents container modification
Secrets not in compose file	`.env` file excluded from version control
Dashboard disabled	`api.dashboard=false`
SSL/TLS	Cloudflare Full mode
Database not exposed	No `ports:` mapping on PostgreSQL
---
Useful Commands
```bash
# View running containers
docker ps

# View n8n logs
docker logs n8n-04wk-n8n-1 --tail 50 -f

# View Traefik logs
docker logs traefik-traefik-1 --tail 50 -f

# View PostgreSQL logs
docker logs postgres_n8n_prod --tail 50 -f

# Restart n8n
docker compose -f /docker/n8n-04wk/docker-compose.yml restart n8n

# Restart Traefik
docker compose -f /docker/traefik/docker-compose.yml restart traefik

# Check disk usage
docker system df

# Clean unused images/containers
docker system prune -f
```
---
Backup Strategy
n8n data backup
```bash
# Backup n8n volume
docker run --rm \
  -v n8n_data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/n8n_backup_$(date +%Y%m%d).tar.gz /data
```
PostgreSQL backup
```bash
# Dump PostgreSQL database
docker exec postgres_n8n_prod \
  pg_dump -U n8n n8n > n8n_db_backup_$(date +%Y%m%d).sql
```
---
Work in Progress
[ ] Automated daily backups with cron
[ ] HTTP security headers middleware in Traefik
[ ] Rate limiting middleware for public routes
[ ] Monitoring with Wazuh or Uptime Kuma
[ ] Staging environment for workflow testing
---
Related Repositories
linux-server-hardening — Complete Ubuntu server hardening guide applied to this server
---
About True Recovery
True Recovery is a remote automation studio based in Paraguay, helping small and mid-sized businesses modernize their collections, customer communication, and operational workflows using n8n and the infrastructure documented in this repository.
---
Maintained by Matheo M. | True Recovery | Paraguay
Stack: Ubuntu 24.04 · Docker · Traefik · n8n · PostgreSQL · Cloudflare
