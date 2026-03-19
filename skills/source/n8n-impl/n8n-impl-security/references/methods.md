# Security Environment Variables & Methods

> Complete reference for all n8n v1.x security-related environment variables, encryption settings, and audit API.

## Credential Encryption

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ENCRYPTION_KEY` | random | Encryption key for all credentials stored in database |

**Behavior:**
- If not set, n8n generates a random key and stores it in `~/.n8n/config`
- The key encrypts ALL credential data before writing to the database
- Changing or losing the key makes ALL stored credentials unreadable
- In queue mode, ALL instances (main + workers) MUST share the SAME key
- The key is NOT recoverable — ALWAYS back it up separately from the database

**Generation:**
```bash
# Generate a cryptographically secure key
openssl rand -hex 32
```

---

## Security Environment Variables

### Core Security

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Block `$env` access in expressions and Code node |
| `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES` | `true` | Block file operations targeting `.n8n` directory |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` | `false` | Set 0600 permissions on settings file at startup |
| `N8N_RESTRICT_FILE_ACCESS_TO` | — | Semicolon-separated list of allowed directories for file operations |
| `N8N_SECURE_COOKIE` | `true` | Set Secure flag on cookies (HTTPS only) |
| `N8N_SAMESITE_COOKIE` | `lax` | SameSite cookie policy: `strict`, `lax`, or `none` |
| `N8N_CONTENT_SECURITY_POLICY` | `{}` | CSP headers in helmet.js format |

### Git Node Security

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_GIT_NODE_DISABLE_BARE_REPOS` | `false` | Prevent Git node from using bare repositories |
| `N8N_GIT_NODE_ENABLE_HOOKS` | `false` | Allow Git node to execute hooks (DANGEROUS) |

### Node Restrictions

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE_FUNCTION_ALLOW_BUILTIN` | — | Comma-separated allowed Node.js built-in modules in Code node. `*` allows all |
| `NODE_FUNCTION_ALLOW_EXTERNAL` | — | Comma-separated allowed npm packages in Code node |
| `NODES_EXCLUDE` | `["n8n-nodes-base.executeCommand", "n8n-nodes-base.localFileTrigger"]` | Array of node types to exclude from loading |
| `NODES_INCLUDE` | — | Whitelist of node types to include (overrides exclude) |
| `N8N_COMMUNITY_PACKAGES_ENABLED` | `true` | Allow installation of community nodes |
| `N8N_COMMUNITY_PACKAGES_PREVENT_LOADING` | `false` | Prevent loading already-installed community nodes |

### SSL/TLS

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_SSL_KEY` | — | Path to SSL private key file |
| `N8N_SSL_CERT` | — | Path to SSL certificate file |
| `N8N_PROTOCOL` | `http` | Set to `https` when behind TLS-terminating proxy |
| `N8N_PROXY_HOPS` | `0` | Number of reverse proxies in front of n8n (for correct IP detection) |

### Task Runners (Sandboxed Execution)

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_RUNNERS_ENABLED` | `false` | Enable task runners for sandboxed Code node execution |
| `N8N_RUNNERS_MODE` | `internal` | `internal` (same process) or `external` (separate process) |
| `N8N_RUNNERS_AUTH_TOKEN` | — | Shared authentication secret for external runners |
| `N8N_RUNNERS_BROKER_PORT` | `5679` | Port for runner broker communication |
| `N8N_RUNNERS_BROKER_LISTEN_ADDRESS` | `127.0.0.1` | Address the broker listens on |
| `N8N_RUNNERS_MAX_PAYLOAD` | `1073741824` | Maximum communication payload in bytes (1GB) |
| `N8N_RUNNERS_MAX_OLD_SPACE_SIZE` | — | Node.js memory allocation for runner (MB) |
| `N8N_RUNNERS_MAX_CONCURRENCY` | `5` | Maximum concurrent tasks per runner |
| `N8N_RUNNERS_TASK_TIMEOUT` | `300` | Task execution timeout in seconds |
| `N8N_RUNNERS_HEARTBEAT_INTERVAL` | `30` | Health check interval in seconds |

### Execution Data Pruning

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_DATA_PRUNE` | `true` | Enable automatic execution data pruning |
| `EXECUTIONS_DATA_MAX_AGE` | `336` | Maximum execution age in hours (336 = 14 days) |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` | Maximum stored executions before pruning |
| `EXECUTIONS_DATA_HARD_DELETE_BUFFER` | `1` | Hours to wait before hard deletion |
| `EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL` | `15` | Hard delete check interval in minutes |
| `EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL` | `60` | Soft delete check interval in minutes |
| `EXECUTIONS_DATA_SAVE_ON_ERROR` | `all` | Save execution data on error: `all` or `none` |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `all` | Save execution data on success: `all` or `none` |
| `EXECUTIONS_DATA_SAVE_ON_PROGRESS` | `false` | Save progress data per node execution |
| `EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS` | `true` | Save data from manual test executions |

### Security Audit

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_SECURITY_AUDIT_DAYS_ABANDONED_WORKFLOW` | `90` | Days of inactivity before a workflow is flagged as abandoned |

### API Security

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_PUBLIC_API_DISABLED` | `false` | Disable the public REST API entirely |
| `N8N_PUBLIC_API_SWAGGERUI_DISABLED` | `false` | Disable the Swagger/Scalar API playground |

---

## Docker Secrets (_FILE Suffix)

n8n supports loading sensitive values from files using the `_FILE` suffix convention. This integrates with Docker Secrets and Kubernetes Secrets.

**How it works:**
1. Create a secret file containing ONLY the secret value (no trailing newline)
2. Mount the file into the container (via Docker Secrets or volume mount)
3. Set the environment variable with `_FILE` suffix pointing to the file path

**Supported variables** (any credential/connection variable):
- `N8N_ENCRYPTION_KEY_FILE`
- `DB_POSTGRESDB_PASSWORD_FILE`
- `DB_POSTGRESDB_USER_FILE`
- `QUEUE_BULL_REDIS_PASSWORD_FILE`
- Any variable ending in `_PASSWORD`, `_KEY`, `_SECRET`, etc.

**Docker Secrets example:**
```bash
# Create secret files
echo -n "my-encryption-key-value" > ./secrets/encryption_key.txt
echo -n "my-db-password" > ./secrets/db_password.txt
chmod 600 ./secrets/*.txt
```

---

## Security Audit API

### CLI

```bash
n8n audit
```

### REST API

```bash
POST /api/v1/audit
```

**Request:**
```bash
curl -X POST 'https://n8n.example.com/api/v1/audit' \
  -H 'X-N8N-API-KEY: <your-api-key>' \
  -H 'Content-Type: application/json'
```

**Audit checks include:**
- Abandoned workflows (configurable via `N8N_SECURITY_AUDIT_DAYS_ABANDONED_WORKFLOW`)
- Insecure credential configurations
- Risky node configurations (execute command, file system access)
- Workflow vulnerability patterns

---

## Configuration Methods

n8n supports three methods for setting configuration:

### 1. Environment Variables (Primary)

```bash
# Shell
export N8N_ENCRYPTION_KEY=your-key-here

# Docker CLI
docker run -e N8N_ENCRYPTION_KEY=your-key-here ...

# Docker Compose
environment:
  - N8N_ENCRYPTION_KEY=your-key-here
```

### 2. Docker Compose File

Variables defined in the `environment:` section of `docker-compose.yml`.

### 3. _FILE Suffix (Docker/Kubernetes Secrets)

```yaml
environment:
  - N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key
```

**Priority:** `_FILE` suffix takes precedence over the plain variable if both are set.

---

## External Secrets

n8n supports integration with external secret providers for centralized secret management. This allows credentials to reference secrets stored in external vaults rather than in the n8n database.

**Supported providers** (Enterprise feature):
- AWS Secrets Manager
- Azure Key Vault
- HashiCorp Vault
- Infisical

**How it works:**
1. Configure the external secrets provider in n8n Settings
2. Credentials reference external secret paths instead of storing values directly
3. n8n resolves the secret at runtime from the provider

**Benefits:**
- Centralized secret rotation without updating n8n credentials
- Audit trail for secret access
- Compliance with organizational secret management policies
