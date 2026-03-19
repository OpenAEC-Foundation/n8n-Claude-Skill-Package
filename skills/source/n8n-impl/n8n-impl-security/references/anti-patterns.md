# Security Anti-Patterns

> Common n8n v1.x security mistakes with explanations and correct alternatives.

## AP-1: No Explicit Encryption Key

**WRONG:**
```yaml
# docker-compose.yml — no N8N_ENCRYPTION_KEY set
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    volumes:
      - n8n_data:/home/node/.n8n
```

**Problem:** n8n generates a random encryption key and stores it in `/home/node/.n8n/config`. If the volume is lost, corrupted, or recreated, ALL stored credentials become permanently unreadable. There is NO recovery mechanism.

**CORRECT:**
```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    secrets:
      - n8n_encryption_key
    environment:
      - N8N_ENCRYPTION_KEY_FILE=/run/secrets/n8n_encryption_key
```

**Rule:** ALWAYS set `N8N_ENCRYPTION_KEY` explicitly. ALWAYS back it up separately from the database.

---

## AP-2: Secrets in Docker Compose

**WRONG:**
```yaml
services:
  n8n:
    environment:
      - N8N_ENCRYPTION_KEY=abc123def456
      - DB_POSTGRESDB_PASSWORD=my-secret-password
```

**Problem:** Secrets are stored in plaintext in the compose file. If the file is committed to version control, anyone with repository access sees the secrets. Even without version control, the compose file is readable by any user on the host.

**CORRECT:**
```yaml
services:
  n8n:
    secrets:
      - n8n_encryption_key
      - db_password
    environment:
      - N8N_ENCRYPTION_KEY_FILE=/run/secrets/n8n_encryption_key
      - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  n8n_encryption_key:
    file: ./secrets/n8n_encryption_key.txt
  db_password:
    file: ./secrets/db_password.txt
```

**Rule:** NEVER put secrets directly in docker-compose.yml. ALWAYS use Docker Secrets with the `_FILE` suffix.

---

## AP-3: SQLite in Production

**WRONG:**
```yaml
services:
  n8n:
    environment:
      - DB_TYPE=sqlite  # or simply not setting DB_TYPE (default is sqlite)
```

**Problem:** SQLite does NOT support queue mode, has limited concurrent access, is prone to corruption under heavy load, and cannot scale to multiple instances.

**CORRECT:**
```yaml
services:
  n8n:
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password
```

**Rule:** ALWAYS use PostgreSQL for production deployments. SQLite is ONLY acceptable for local development or single-user testing.

---

## AP-4: Task Runners Disabled

**WRONG:**
```yaml
services:
  n8n:
    environment:
      # N8N_RUNNERS_ENABLED not set (defaults to false)
```

**Problem:** Without task runners, Code node JavaScript/Python executes directly in the main n8n process. Malicious or buggy code can access the n8n process memory, crash the entire instance, or perform unauthorized operations.

**CORRECT:**
```yaml
services:
  n8n:
    environment:
      - N8N_RUNNERS_ENABLED=true
      - N8N_RUNNERS_TASK_TIMEOUT=300
      - N8N_RUNNERS_MAX_CONCURRENCY=5
```

**Rule:** ALWAYS enable task runners in production. ALWAYS set a task timeout to prevent runaway code.

---

## AP-5: HTTP Without TLS

**WRONG:**
```yaml
services:
  n8n:
    ports:
      - "5678:5678"
    environment:
      - N8N_PROTOCOL=http
```

**Problem:** All traffic including credentials, API keys, and session cookies is transmitted in cleartext. Anyone on the network can intercept sensitive data.

**CORRECT:** Use a reverse proxy with TLS termination (see examples.md for Traefik and Nginx configurations).

```yaml
services:
  n8n:
    environment:
      - N8N_PROTOCOL=https
      - N8N_PROXY_HOPS=1
      - WEBHOOK_URL=https://n8n.example.com/
      - N8N_SECURE_COOKIE=true
```

**Rule:** ALWAYS use HTTPS in production. ALWAYS use a reverse proxy for TLS termination.

---

## AP-6: No Execution Data Pruning

**WRONG:**
```yaml
services:
  n8n:
    environment:
      - EXECUTIONS_DATA_PRUNE=false
```

**Problem:** Execution data accumulates indefinitely. The database grows without bound, eventually causing slow queries, disk exhaustion, and potential instance crashes.

**CORRECT:**
```yaml
services:
  n8n:
    environment:
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=336        # 14 days
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
```

**Rule:** ALWAYS enable execution data pruning in production. ALWAYS set a maximum age and count.

---

## AP-7: Unrestricted Code Node Modules

**WRONG:**
```bash
NODE_FUNCTION_ALLOW_BUILTIN=*
NODE_FUNCTION_ALLOW_EXTERNAL=*
```

**Problem:** The Code node can import ANY Node.js built-in module (including `child_process`, `fs`, `net`) and ANY installed npm package. This allows arbitrary command execution, file system access, and network operations from within workflows.

**CORRECT:**
```bash
NODE_FUNCTION_ALLOW_BUILTIN=crypto,path,url,querystring
NODE_FUNCTION_ALLOW_EXTERNAL=lodash,moment,date-fns
```

**Rule:** NEVER use wildcard `*` for module access in production. ALWAYS whitelist only the specific modules your workflows need.

---

## AP-8: API Playground Exposed in Production

**WRONG:**
```yaml
services:
  n8n:
    environment:
      # N8N_PUBLIC_API_SWAGGERUI_DISABLED not set (defaults to false)
```

**Problem:** The Swagger/Scalar UI at `/api/v1/docs` exposes the complete API documentation and an interactive playground. This provides attackers with a detailed map of all available endpoints.

**CORRECT:**
```yaml
services:
  n8n:
    environment:
      - N8N_PUBLIC_API_SWAGGERUI_DISABLED=true
```

**Rule:** ALWAYS disable the API playground in production unless actively needed for development.

---

## AP-9: Mismatched Encryption Keys in Queue Mode

**WRONG:**
```yaml
# Main instance
n8n-main:
  environment:
    - N8N_ENCRYPTION_KEY=key-for-main

# Worker instance
n8n-worker:
  environment:
    - N8N_ENCRYPTION_KEY=different-key-for-worker
```

**Problem:** Workers cannot decrypt credentials encrypted by the main instance. Workflow executions fail silently or with cryptic decryption errors. This is one of the most common queue mode issues.

**CORRECT:**
```yaml
# Both instances use the SAME secret
n8n-main:
  secrets:
    - n8n_encryption_key
  environment:
    - N8N_ENCRYPTION_KEY_FILE=/run/secrets/n8n_encryption_key

n8n-worker:
  secrets:
    - n8n_encryption_key
  environment:
    - N8N_ENCRYPTION_KEY_FILE=/run/secrets/n8n_encryption_key
```

**Rule:** ALWAYS use the exact same `N8N_ENCRYPTION_KEY` across ALL instances (main, workers, webhook processors). Use Docker Secrets to ensure consistency.

---

## AP-10: Exporting Decrypted Credentials

**WRONG:**
```bash
n8n export:credentials --all --decrypted --output=./backup/credentials.json
```

**Problem:** Credentials are exported in plaintext. If this file is stored insecurely, committed to version control, or left on a shared filesystem, all API keys, passwords, and tokens are exposed.

**CORRECT:**
```bash
# Export encrypted (default behavior)
n8n export:credentials --all --output=./backup/credentials.json
# Ensure encryption key is backed up separately to allow restore
```

**Rule:** NEVER use `--decrypted` in production backup scripts. ALWAYS export credentials in their encrypted form and back up the encryption key separately.

---

## AP-11: Missing N8N_PROXY_HOPS

**WRONG:**
```yaml
services:
  n8n:
    environment:
      - N8N_PROTOCOL=https
      # N8N_PROXY_HOPS not set (defaults to 0)
```

**Problem:** Without `N8N_PROXY_HOPS`, n8n does not trust the `X-Forwarded-For` header from the reverse proxy. This means IP-based webhook restrictions, rate limiting, and audit logging use the proxy's IP address instead of the client's real IP.

**CORRECT:**
```yaml
services:
  n8n:
    environment:
      - N8N_PROTOCOL=https
      - N8N_PROXY_HOPS=1  # Set to number of proxies in front of n8n
```

**Rule:** ALWAYS set `N8N_PROXY_HOPS` to the number of reverse proxies in front of n8n when using a proxy.

---

## AP-12: Forgetting to Back Up Encryption Key

**WRONG:**
```bash
# Backup script that only backs up the database
pg_dump -U n8n n8n > backup.sql
```

**Problem:** The database backup contains encrypted credentials. Without the encryption key, these credentials are permanently unreadable. Restoring from this backup on a new instance with a different key means losing ALL credentials.

**CORRECT:**
```bash
# Back up BOTH database AND encryption key
pg_dump -U n8n n8n > backup.sql
cp /run/secrets/n8n_encryption_key /secure-backup/encryption_key.txt
chmod 600 /secure-backup/encryption_key.txt
```

**Rule:** ALWAYS back up the encryption key alongside (but separately from) database backups. ALWAYS test restore procedures to verify the key works.
