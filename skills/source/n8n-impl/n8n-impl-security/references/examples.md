# Security Configuration Examples

> Production-ready examples for n8n v1.x security hardening, reverse proxy setup, SSL/TLS, and Docker Secrets.

## Complete Hardened Docker Compose

This example shows a fully hardened production deployment with Traefik, PostgreSQL, and Docker Secrets.

```yaml
# docker-compose.yml — Production hardened n8n
services:
  traefik:
    image: traefik:latest
    restart: always
    command:
      - "--api=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"
    secrets:
      - n8n_encryption_key
      - db_password
    environment:
      # Core
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      # Security — secrets via _FILE suffix
      - N8N_ENCRYPTION_KEY_FILE=/run/secrets/n8n_encryption_key
      - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password
      # Security — hardening
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=true
      - N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES=true
      - N8N_RESTRICT_FILE_ACCESS_TO=/files
      - N8N_SECURE_COOKIE=true
      - N8N_SAMESITE_COOKIE=strict
      - N8N_PROXY_HOPS=1
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_SCHEMA=public
      # Execution pruning
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=336
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
      # Task runners
      - N8N_RUNNERS_ENABLED=true
      - N8N_RUNNERS_TASK_TIMEOUT=300
      - N8N_RUNNERS_MAX_CONCURRENCY=5
      # Disable unnecessary features
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_PUBLIC_API_SWAGGERUI_DISABLED=true
      - N8N_TEMPLATES_ENABLED=false
      - N8N_HIRING_BANNER_ENABLED=false
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    depends_on:
      - postgres

  postgres:
    image: postgres:16-alpine
    restart: always
    secrets:
      - db_password
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 10s
      timeout: 5s
      retries: 5

secrets:
  n8n_encryption_key:
    file: ./secrets/n8n_encryption_key.txt
  db_password:
    file: ./secrets/db_password.txt

volumes:
  traefik_data:
  n8n_data:
  postgres_data:
```

### Environment File (.env)

```env
# .env — NEVER commit this file to version control
DOMAIN_NAME=example.com
SUBDOMAIN=n8n
GENERIC_TIMEZONE=Europe/Amsterdam
SSL_EMAIL=admin@example.com
```

### Secret Files Setup

```bash
# Create secrets directory
mkdir -p ./secrets
chmod 700 ./secrets

# Generate encryption key
openssl rand -hex 32 > ./secrets/n8n_encryption_key.txt

# Generate database password
openssl rand -base64 32 > ./secrets/db_password.txt

# Lock down permissions
chmod 600 ./secrets/*.txt

# CRITICAL: Back up encryption key to a SEPARATE secure location
cp ./secrets/n8n_encryption_key.txt /secure-backup/n8n_encryption_key.txt
```

---

## Reverse Proxy: Nginx Configuration

```nginx
# /etc/nginx/sites-available/n8n
server {
    listen 80;
    server_name n8n.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_cache off;

        # Chunked transfer encoding for streaming
        chunked_transfer_encoding on;
    }
}
```

**Required n8n environment variables with Nginx:**
```bash
N8N_PROTOCOL=https
N8N_HOST=n8n.example.com
N8N_PROXY_HOPS=1
WEBHOOK_URL=https://n8n.example.com/
N8N_SECURE_COOKIE=true
```

---

## Reverse Proxy: Traefik Configuration

Traefik is the recommended reverse proxy for n8n Docker deployments because it provides automatic Let's Encrypt certificate management.

See the complete Docker Compose example above for the full Traefik integration. Key Traefik labels for n8n:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.n8n.rule=Host(`n8n.example.com`)"
  - "traefik.http.routers.n8n.entrypoints=websecure"
  - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
  - "traefik.http.services.n8n.loadbalancer.server.port=5678"
```

---

## Direct SSL/TLS (Without Reverse Proxy)

For simple deployments without a reverse proxy:

```bash
N8N_PROTOCOL=https
N8N_SSL_KEY=/path/to/private.key
N8N_SSL_CERT=/path/to/certificate.crt
```

**Rules:**
- NEVER use self-signed certificates in production
- ALWAYS set up automatic certificate renewal (e.g., certbot cron)
- Prefer reverse proxy over direct SSL for production deployments

---

## Queue Mode Security

When running n8n in queue mode, additional security considerations apply:

```bash
# Main instance
EXECUTIONS_MODE=queue
N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key  # MUST match workers
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PASSWORD_FILE=/run/secrets/redis_password

# Worker instance — MUST use same encryption key
EXECUTIONS_MODE=queue
N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key  # SAME key as main
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PASSWORD_FILE=/run/secrets/redis_password
```

**Rules:**
- ALWAYS share the same `N8N_ENCRYPTION_KEY` across main and ALL workers
- ALWAYS use `_FILE` suffix for Redis password
- ALWAYS enable Redis authentication in production
- NEVER expose Redis to the public internet

---

## Node Restriction Examples

### Minimal Permissions (High Security)

```bash
# Block environment variable access
N8N_BLOCK_ENV_ACCESS_IN_NODE=true

# Restrict file access
N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES=true
N8N_RESTRICT_FILE_ACCESS_TO=/files;/tmp

# Exclude dangerous nodes
NODES_EXCLUDE=["n8n-nodes-base.executeCommand","n8n-nodes-base.localFileTrigger","n8n-nodes-base.readWriteFile"]

# Restrict Code node
NODE_FUNCTION_ALLOW_BUILTIN=crypto,path,url
NODE_FUNCTION_ALLOW_EXTERNAL=lodash,moment

# Disable community packages
N8N_COMMUNITY_PACKAGES_ENABLED=false
```

### Standard Permissions (Balanced)

```bash
N8N_BLOCK_ENV_ACCESS_IN_NODE=true
N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES=true
N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
N8N_RUNNERS_ENABLED=true
NODES_EXCLUDE=["n8n-nodes-base.executeCommand"]
```

---

## Backup Script with Encryption Key Safety

```bash
#!/bin/bash
# backup-n8n.sh — Run daily via cron
set -euo pipefail

BACKUP_DIR="/backups/n8n/$(date +%Y-%m-%d)"
mkdir -p "$BACKUP_DIR"

# Database backup (PostgreSQL)
docker exec postgres pg_dump -U n8n n8n > "$BACKUP_DIR/database.sql"

# Workflow export
docker exec n8n n8n export:workflow --backup --output=/tmp/workflows/
docker cp n8n:/tmp/workflows/ "$BACKUP_DIR/workflows/"

# Credential export (encrypted — requires same encryption key to restore)
docker exec n8n n8n export:credentials --backup --output=/tmp/credentials/
docker cp n8n:/tmp/credentials/ "$BACKUP_DIR/credentials/"

# Encryption key backup (CRITICAL)
cp ./secrets/n8n_encryption_key.txt "$BACKUP_DIR/encryption_key.txt"
chmod 600 "$BACKUP_DIR/encryption_key.txt"

# Compress and secure
tar czf "$BACKUP_DIR.tar.gz" "$BACKUP_DIR"
chmod 600 "$BACKUP_DIR.tar.gz"
rm -rf "$BACKUP_DIR"

echo "Backup complete: $BACKUP_DIR.tar.gz"
```

**Rules:**
- ALWAYS include the encryption key in backups
- ALWAYS store backups in a different location than the running instance
- NEVER export credentials with `--decrypted` unless absolutely necessary
- ALWAYS test restore procedures periodically
