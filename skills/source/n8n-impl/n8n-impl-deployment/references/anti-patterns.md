# n8n Deployment Anti-Patterns

> Common deployment mistakes, security misconfigurations, and scaling pitfalls.
> Each anti-pattern includes the mistake, why it fails, and the correct approach.

---

## 1. Encryption Key Mistakes

### AP-001: Not Setting N8N_ENCRYPTION_KEY Explicitly

**Mistake:** Relying on the auto-generated encryption key without setting it as an environment variable.

**Why it fails:** The auto-generated key is stored inside the `/home/node/.n8n` volume. If the volume is lost, recreated, or you migrate to a new server, ALL stored credentials become permanently unrecoverable.

**Correct approach:**
```bash
# ALWAYS set explicitly
-e N8N_ENCRYPTION_KEY=your-secure-random-key
```
ALWAYS generate a strong random key, set it as an environment variable, and store a backup in a secure location (password manager, vault).

### AP-002: Different Encryption Keys Across Instances

**Mistake:** Each instance in a queue mode deployment generates its own encryption key.

**Why it fails:** Workers cannot decrypt credentials created by the main instance. Workflows fail silently or throw credential decryption errors.

**Correct approach:** ALWAYS use the EXACT same `N8N_ENCRYPTION_KEY` value on main, workers, and webhook processors. Set it in a shared `.env` file or secrets manager.

### AP-003: Storing Encryption Key in Version Control

**Mistake:** Committing `.env` files containing `N8N_ENCRYPTION_KEY` to Git.

**Why it fails:** Anyone with repository access can decrypt all stored credentials. This is a critical security breach.

**Correct approach:** ALWAYS add `.env` to `.gitignore`. Use Docker Secrets, Kubernetes Secrets, or a vault service for production. Use `_FILE` suffix:
```yaml
- N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key
```

---

## 2. Database Mistakes

### AP-004: Using SQLite in Production

**Mistake:** Running production n8n with the default SQLite database.

**Why it fails:**
- SQLite does NOT support queue mode
- SQLite does NOT handle concurrent access well
- SQLite cannot scale to multi-instance deployments
- Database corruption risk under heavy write load

**Correct approach:** ALWAYS use PostgreSQL (`DB_TYPE=postgresdb`) for any production deployment. SQLite is acceptable ONLY for single-user development or testing.

### AP-005: Using SQLite with Queue Mode

**Mistake:** Setting `EXECUTIONS_MODE=queue` while using SQLite.

**Why it fails:** Queue mode requires multiple processes to read/write the database concurrently. SQLite has file-level locking that causes deadlocks, data corruption, and execution failures.

**Correct approach:** Queue mode ALWAYS requires PostgreSQL. There is no workaround.

### AP-006: No Database Connection Pooling

**Mistake:** Using default `DB_POSTGRESDB_POOL_SIZE=2` with high-traffic deployments.

**Why it fails:** Connection exhaustion under load causes timeouts and execution failures.

**Correct approach:** Increase pool size based on expected concurrent executions:
```bash
-e DB_POSTGRESDB_POOL_SIZE=10  # Adjust based on load
```

---

## 3. Networking & Proxy Mistakes

### AP-007: Missing WEBHOOK_URL Behind Reverse Proxy

**Mistake:** Not setting `WEBHOOK_URL` when n8n is behind Traefik, nginx, or another reverse proxy.

**Why it fails:** Webhook URLs in the editor display `http://localhost:5678/webhook/...` instead of the public URL. External services cannot reach the webhooks.

**Correct approach:**
```bash
-e WEBHOOK_URL=https://n8n.example.com/
```
ALWAYS set `WEBHOOK_URL` to the public-facing URL when behind any reverse proxy.

### AP-008: Exposing n8n Directly Without TLS

**Mistake:** Running n8n on port 5678 directly exposed to the internet without HTTPS.

**Why it fails:** All traffic including credentials, API keys, and session cookies are transmitted in plain text. Man-in-the-middle attacks can capture everything.

**Correct approach:** ALWAYS use a reverse proxy (Traefik, nginx, Caddy) with TLS termination. NEVER expose port 5678 directly to the internet.

### AP-009: Missing N8N_PROTOCOL=https

**Mistake:** Not setting `N8N_PROTOCOL=https` when TLS is handled by the reverse proxy.

**Why it fails:** n8n generates HTTP URLs for webhooks and redirects, causing mixed content warnings and broken functionality.

**Correct approach:**
```bash
-e N8N_PROTOCOL=https
```

### AP-010: Not Setting N8N_PROXY_HOPS

**Mistake:** Not configuring `N8N_PROXY_HOPS` when behind one or more reverse proxies.

**Why it fails:** n8n cannot determine the real client IP from `X-Forwarded-For` headers, affecting rate limiting, IP whitelisting, and security logging.

**Correct approach:**
```bash
-e N8N_PROXY_HOPS=1  # Number of proxies between client and n8n
```

---

## 4. Volume & Data Mistakes

### AP-011: No Persistent Volume for /home/node/.n8n

**Mistake:** Running n8n without a Docker volume mapped to `/home/node/.n8n`.

**Why it fails:** Container restart loses ALL data: SQLite database, encryption key, settings, and binary data. Everything is gone.

**Correct approach:**
```bash
-v n8n_data:/home/node/.n8n
```
ALWAYS mount a persistent volume. NEVER run n8n without persistent storage in any environment beyond quick tests.

### AP-012: Using Bind Mounts with Wrong Permissions

**Mistake:** Using `./data:/home/node/.n8n` bind mount where the host directory is owned by root.

**Why it fails:** n8n runs as user `node` (UID 1000) inside the container. Permission denied errors prevent reading/writing the database and settings.

**Correct approach:** Ensure the host directory is owned by UID 1000:
```bash
sudo chown -R 1000:1000 ./data
```
Or use Docker named volumes (preferred):
```bash
-v n8n_data:/home/node/.n8n
```

---

## 5. Queue Mode Mistakes

### AP-013: Version Mismatch Across Instances

**Mistake:** Running different n8n versions on main and worker instances.

**Why it fails:** Database schema differences cause migration conflicts. Workers may not understand workflow structures from a newer main instance. Executions silently fail or produce incorrect results.

**Correct approach:** ALWAYS pin the same Docker image tag across ALL services:
```yaml
n8n-main:
  image: docker.n8n.io/n8nio/n8n:1.81.0
n8n-worker:
  image: docker.n8n.io/n8nio/n8n:1.81.0
```

### AP-014: No S3 for Binary Data in Queue Mode

**Mistake:** Using filesystem or default binary data mode with queue mode.

**Why it fails:** Workers run in separate containers with separate filesystems. Binary data written by one instance is not accessible to another.

**Correct approach:** ALWAYS use S3-compatible storage for binary data in queue mode:
```bash
-e N8N_DEFAULT_BINARY_DATA_MODE=s3
-e N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=n8n-data
```

### AP-015: Updating Main Before Stopping Workers

**Mistake:** Updating the main instance while workers are still running the old version.

**Why it fails:** The main instance runs database migrations that change the schema. Old workers cannot read the new schema, causing execution failures.

**Correct approach:**
1. Stop workers and webhook processors
2. Update and restart main instance (runs migrations)
3. Wait for main to be healthy
4. Start workers with the new version

---

## 6. Security Mistakes

### AP-016: NODE_ENV Not Set to Production

**Mistake:** Running production n8n without `NODE_ENV=production`.

**Why it fails:** Development mode enables verbose error messages, debug endpoints, and relaxed security settings that expose internal information to attackers.

**Correct approach:**
```bash
-e NODE_ENV=production
```

### AP-017: Not Blocking Environment Variable Access

**Mistake:** Leaving `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (default) in production.

**Why it fails:** Any user with workflow edit access can read ALL environment variables (including database passwords, API keys, encryption key) via the Code node or expressions using `$env`.

**Correct approach:**
```bash
-e N8N_BLOCK_ENV_ACCESS_IN_NODE=true
```
ALWAYS enable this in multi-user production environments.

### AP-018: Exporting Credentials Decrypted

**Mistake:** Using `n8n export:credentials --all --decrypted` and storing the output.

**Why it fails:** All credential secrets (API keys, passwords, tokens) are written in plain text. If this file is compromised, every integrated service is exposed.

**Correct approach:** NEVER use `--decrypted` in production. Export encrypted credentials and ensure the `N8N_ENCRYPTION_KEY` is backed up separately to enable future restoration.

### AP-019: Not Enforcing Settings File Permissions

**Mistake:** Not setting `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`.

**Why it fails:** The settings file (which may contain the encryption key) is readable by other users/processes in the container.

**Correct approach:**
```bash
-e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
```

---

## 7. Backup Mistakes

### AP-020: No Backup Strategy

**Mistake:** Running production n8n without any backup automation.

**Why it fails:** Database corruption, accidental deletion, or server failure results in total data loss — workflows, credentials, and execution history.

**Correct approach:** Implement automated daily backups:
1. PostgreSQL database dump (`pg_dump`)
2. CLI workflow/credential export
3. `N8N_ENCRYPTION_KEY` stored in a secure vault
4. Off-site backup storage with retention policy

### AP-021: Backing Up Workflows Without Encryption Key

**Mistake:** Exporting credentials via CLI but not backing up the `N8N_ENCRYPTION_KEY`.

**Why it fails:** Exported credentials are encrypted. Without the matching encryption key, they cannot be imported or decrypted on a new instance. The backup is useless.

**Correct approach:** ALWAYS back up `N8N_ENCRYPTION_KEY` alongside credential exports. Store it separately in a secure vault or password manager.

### AP-022: Not Testing Restore Procedure

**Mistake:** Having backups but never testing whether they can be restored.

**Why it fails:** Backup files may be corrupted, incomplete, or incompatible. You discover this during an actual emergency when it is too late.

**Correct approach:** ALWAYS test restore procedures quarterly:
1. Spin up a fresh n8n instance
2. Restore database from backup
3. Set the backed-up `N8N_ENCRYPTION_KEY`
4. Verify workflows load and credentials decrypt correctly

---

## 8. Performance Mistakes

### AP-023: No Execution Data Pruning

**Mistake:** Not configuring `EXECUTIONS_DATA_PRUNE=true` or setting unreasonable retention.

**Why it fails:** Execution data accumulates indefinitely, consuming disk space and slowing database queries. Eventually the database becomes unmanageably large.

**Correct approach:**
```bash
-e EXECUTIONS_DATA_PRUNE=true
-e EXECUTIONS_DATA_MAX_AGE=336           # 14 days
-e EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000  # Max stored executions
```

### AP-024: Unlimited Concurrent Executions

**Mistake:** Not setting `N8N_CONCURRENCY_PRODUCTION_LIMIT` on resource-constrained servers.

**Why it fails:** Burst webhook traffic triggers hundreds of concurrent executions, exhausting CPU, memory, and database connections. The entire instance becomes unresponsive.

**Correct approach:**
```bash
-e N8N_CONCURRENCY_PRODUCTION_LIMIT=20  # Adjust to server capacity
```

### AP-025: Not Enabling Task Runners

**Mistake:** Not setting `N8N_RUNNERS_ENABLED=true`.

**Why it fails:** Code node executions run in the main n8n process. A misbehaving script can crash or hang the entire instance.

**Correct approach:**
```bash
-e N8N_RUNNERS_ENABLED=true
```
Task runners isolate Code node execution in separate processes, preventing crashes from affecting the main instance.
