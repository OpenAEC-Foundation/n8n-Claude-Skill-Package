# n8n Deployment Examples

> Production-ready Docker Compose configs, queue mode setup, CLI commands, and backup automation.

---

## 1. Docker Compose: Single Instance with PostgreSQL and Traefik

This is the recommended production setup for small-to-medium deployments.

### `.env` File

```env
# Domain
DOMAIN_NAME=example.com
SUBDOMAIN=n8n
SSL_EMAIL=admin@example.com

# Timezone
GENERIC_TIMEZONE=Europe/Amsterdam

# Database
POSTGRES_USER=n8n
POSTGRES_PASSWORD=change-me-strong-password
POSTGRES_DB=n8n

# n8n
N8N_ENCRYPTION_KEY=your-secure-random-key-here
```

### `docker-compose.yml`

```yaml
services:
  traefik:
    image: traefik:latest
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Execution pruning
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=336
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)"
      - "traefik.http.routers.n8n.tls=true"
      - "traefik.http.routers.n8n.tls.certresolver=mytlschallenge"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.middlewares.n8n.headers.SSLRedirect=true"
      - "traefik.http.middlewares.n8n.headers.STSSeconds=315360000"
      - "traefik.http.middlewares.n8n.headers.browserXSSFilter=true"
      - "traefik.http.middlewares.n8n.headers.contentTypeNosniff=true"
      - "traefik.http.middlewares.n8n.headers.forceSTSHeader=true"
      - "traefik.http.middlewares.n8n.headers.SSLHost=${SUBDOMAIN}.${DOMAIN_NAME}"
      - "traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true"
      - "traefik.http.middlewares.n8n.headers.STSPreload=true"

volumes:
  traefik_data:
  postgres_data:
  n8n_data:
```

### Launch Commands

```bash
# Start all services
docker compose up -d

# View logs
docker compose logs -f n8n

# Stop all services
docker compose down

# Stop and remove volumes (DESTRUCTIVE)
docker compose down -v
```

---

## 2. Docker Compose: Queue Mode with Workers

Full scaled deployment with main instance, workers, Redis, and PostgreSQL.

### `docker-compose-queue.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -U n8n -d n8n"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 10

  n8n-main:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "5678:5678"
    environment:
      - NODE_ENV=production
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      # Queue mode
      - EXECUTIONS_MODE=queue
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Redis
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      # Binary data (S3 required for queue mode)
      - N8N_DEFAULT_BINARY_DATA_MODE=s3
      - N8N_EXTERNAL_STORAGE_S3_HOST=${S3_HOST}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=${S3_BUCKET}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION=${S3_REGION}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY=${S3_ACCESS_KEY}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET=${S3_ACCESS_SECRET}
      # Execution pruning
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=336
    volumes:
      - n8n_data:/home/node/.n8n

  n8n-worker:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: worker
    deploy:
      replicas: 2
    environment:
      - NODE_ENV=production
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      # Queue mode
      - EXECUTIONS_MODE=queue
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Redis
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      # Binary data
      - N8N_DEFAULT_BINARY_DATA_MODE=s3
      - N8N_EXTERNAL_STORAGE_S3_HOST=${S3_HOST}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=${S3_BUCKET}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION=${S3_REGION}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY=${S3_ACCESS_KEY}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET=${S3_ACCESS_SECRET}
      # Worker health
      - QUEUE_HEALTH_CHECK_ACTIVE=true
      - N8N_GRACEFUL_SHUTDOWN_TIMEOUT=60

  n8n-webhook:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: webhook
    ports:
      - "5679:5678"
    environment:
      - NODE_ENV=production
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      # Queue mode
      - EXECUTIONS_MODE=queue
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Redis
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      # Binary data
      - N8N_DEFAULT_BINARY_DATA_MODE=s3
      - N8N_EXTERNAL_STORAGE_S3_HOST=${S3_HOST}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=${S3_BUCKET}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION=${S3_REGION}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY=${S3_ACCESS_KEY}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET=${S3_ACCESS_SECRET}

volumes:
  postgres_data:
  redis_data:
  n8n_data:
```

### Queue Mode `.env`

```env
N8N_ENCRYPTION_KEY=your-shared-encryption-key-all-instances
N8N_HOST=n8n.example.com
GENERIC_TIMEZONE=Europe/Amsterdam
POSTGRES_PASSWORD=strong-database-password

# S3 (required for binary data in queue mode)
S3_HOST=https://s3.amazonaws.com
S3_BUCKET=n8n-binary-data
S3_REGION=eu-west-1
S3_ACCESS_KEY=your-access-key
S3_ACCESS_SECRET=your-access-secret
```

### Scaling Workers

```bash
# Scale to 4 workers
docker compose -f docker-compose-queue.yml up -d --scale n8n-worker=4

# Check worker status
docker compose -f docker-compose-queue.yml ps
```

---

## 3. CLI Command Reference

### Workflow Management

```bash
# Execute workflow by ID
n8n execute --id 123

# Activate a workflow
n8n update:workflow --id=123 --active=true

# Deactivate a workflow
n8n update:workflow --id=123 --active=false

# Deactivate ALL workflows (maintenance mode)
n8n update:workflow --all --active=false

# Reactivate ALL workflows
n8n update:workflow --all --active=true
```

### Export Operations

```bash
# Export all workflows to a directory (one file per workflow)
n8n export:workflow --backup --output=/backups/workflows/

# Export single workflow
n8n export:workflow --id=123 --output=/backups/workflow-123.json

# Export all workflows as pretty-printed JSON
n8n export:workflow --all --pretty --output=/backups/all-workflows.json

# Export all credentials (encrypted)
n8n export:credentials --backup --output=/backups/credentials/

# Export single credential
n8n export:credentials --id=456 --output=/backups/cred-456.json

# DANGER: Export credentials decrypted (NEVER in production)
n8n export:credentials --all --decrypted --output=/backups/decrypted-creds.json

# Export all entities with execution history
n8n export:entities --outputDir=/backups/full/ --includeExecutionHistoryDataTables=true
```

### Import Operations

```bash
# Import workflows from file
n8n import:workflow --input=/backups/all-workflows.json

# Import workflows from directory (separate files)
n8n import:workflow --separate --input=/backups/workflows/

# Import credentials from file
n8n import:credentials --input=/backups/credentials.json

# Import from directory
n8n import:credentials --separate --input=/backups/credentials/

# Import all entities
n8n import:entities --inputDir=/backups/full/ --truncateTables true
```

### Administration

```bash
# Run security audit
n8n audit

# Display license info
n8n license:info

# Clear license
n8n license:clear

# Reset user management (emergency access recovery)
n8n user-management:reset

# Disable MFA for a user
n8n mfa:disable --email=user@example.com

# Reset LDAP settings
n8n ldap:reset

# Uninstall community node
n8n community-node --uninstall --package n8n-nodes-some-package
```

### Running CLI in Docker

```bash
# Execute CLI commands inside running container
docker exec -it n8n n8n export:workflow --all --pretty

# Or run a one-off container for CLI operations
docker run --rm \
  -v n8n_data:/home/node/.n8n \
  -e DB_TYPE=postgresdb \
  -e DB_POSTGRESDB_HOST=postgres-host \
  -e DB_POSTGRESDB_DATABASE=n8n \
  -e DB_POSTGRESDB_USER=n8n \
  -e DB_POSTGRESDB_PASSWORD=secret \
  -e N8N_ENCRYPTION_KEY=your-key \
  docker.n8n.io/n8nio/n8n \
  n8n export:workflow --backup --output=/home/node/.n8n/backups/
```

---

## 4. Backup Automation

### Daily Backup Script (PostgreSQL)

```bash
#!/bin/bash
# n8n-backup.sh — Run daily via cron
set -euo pipefail

BACKUP_DIR="/backups/n8n/$(date +%Y%m%d-%H%M%S)"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# 1. Database dump
docker exec postgres pg_dump -U n8n n8n | gzip > "$BACKUP_DIR/database.sql.gz"

# 2. Workflow export (CLI)
docker exec n8n n8n export:workflow --backup --output=/home/node/.n8n/backup-tmp/
docker cp n8n:/home/node/.n8n/backup-tmp/ "$BACKUP_DIR/workflows/"
docker exec n8n rm -rf /home/node/.n8n/backup-tmp/

# 3. Credential export (encrypted)
docker exec n8n n8n export:credentials --backup --output=/home/node/.n8n/backup-tmp/
docker cp n8n:/home/node/.n8n/backup-tmp/ "$BACKUP_DIR/credentials/"
docker exec n8n rm -rf /home/node/.n8n/backup-tmp/

# 4. Cleanup old backups
find /backups/n8n/ -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

echo "Backup completed: $BACKUP_DIR"
```

### Cron Entry

```bash
# Daily at 02:00 AM
0 2 * * * /opt/scripts/n8n-backup.sh >> /var/log/n8n-backup.log 2>&1
```

### Restore Procedure

```bash
# 1. Stop n8n
docker compose down

# 2. Restore database
gunzip < /backups/n8n/20260319-020000/database.sql.gz | docker exec -i postgres psql -U n8n n8n

# 3. Start n8n
docker compose up -d

# 4. Verify restoration
docker exec n8n n8n export:workflow --all | head -20
```

---

## 5. Updating n8n

### Standard Update (Docker Compose)

```bash
# 1. Backup first (ALWAYS)
./n8n-backup.sh

# 2. Pull new image
docker compose pull n8n

# 3. Recreate container with new image
docker compose up -d n8n

# 4. Verify
docker compose logs -f n8n
```

### Update to Specific Version

```bash
# Edit docker-compose.yml to pin version
# image: docker.n8n.io/n8nio/n8n:1.81.0

docker compose pull n8n
docker compose up -d n8n
```

### Queue Mode Update

When updating queue mode deployments, ALL instances MUST run the same version:

```bash
# 1. Backup
./n8n-backup.sh

# 2. Pull new image
docker compose -f docker-compose-queue.yml pull

# 3. Stop workers first
docker compose -f docker-compose-queue.yml stop n8n-worker n8n-webhook

# 4. Update and restart main (runs database migrations)
docker compose -f docker-compose-queue.yml up -d n8n-main

# 5. Wait for main to be healthy, then start workers
docker compose -f docker-compose-queue.yml up -d n8n-worker n8n-webhook
```

---

## 6. Docker Secrets Integration

### Using `_FILE` Suffix for Sensitive Values

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    environment:
      - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password
      - N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key
    secrets:
      - db_password
      - encryption_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  encryption_key:
    file: ./secrets/encryption_key.txt
```

ALWAYS use Docker Secrets or `_FILE` suffix instead of plain-text passwords in environment variables for production.

---

## 7. Health Check Configuration

### Health Endpoint

n8n exposes a health check at `GET /healthz` (configurable via `N8N_ENDPOINT_HEALTH`).

### Docker Compose Health Check

```yaml
n8n:
  image: docker.n8n.io/n8nio/n8n
  healthcheck:
    test: ["CMD-SHELL", "wget -qO- http://localhost:5678/healthz || exit 1"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 30s
```

### Worker Health Check

```yaml
n8n-worker:
  image: docker.n8n.io/n8nio/n8n
  command: worker
  environment:
    - QUEUE_HEALTH_CHECK_ACTIVE=true
```
