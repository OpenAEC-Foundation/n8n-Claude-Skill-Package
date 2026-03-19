# n8n Environment Variable Reference

> Complete reference of 100+ environment variables organized by category.
> Source: https://docs.n8n.io/hosting/configuration/environment-variables/

---

## 1. Deployment & Core

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_HOST` | `localhost` | Hostname for n8n instance |
| `N8N_PORT` | `5678` | HTTP port |
| `N8N_PROTOCOL` | `http` | Protocol: `http` or `https` |
| `N8N_PATH` | `/` | Deployment base path |
| `N8N_LISTEN_ADDRESS` | `::` | IP address to bind to |
| `N8N_ENCRYPTION_KEY` | random | **CRITICAL**: Encryption key for credentials. ALWAYS set explicitly. ALWAYS share across all instances. |
| `N8N_USER_FOLDER` | `user-folder` | Path for `.n8n` data directory |
| `N8N_EDITOR_BASE_URL` | — | Public URL for editor access |
| `N8N_DISABLE_UI` | `false` | Disable the web editor UI |
| `N8N_PUSH_BACKEND` | `websocket` | UI update transport: `sse` or `websocket` |
| `NODE_ENV` | — | ALWAYS set to `production` for production deployments |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Shutdown grace period in seconds |
| `N8N_PROXY_HOPS` | `0` | Number of reverse proxies in front of n8n |
| `GENERIC_TIMEZONE` | — | Timezone for schedule/cron nodes (e.g., `Europe/Amsterdam`) |
| `TZ` | — | System timezone |

---

## 2. Database

### PostgreSQL

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_TYPE` | `sqlite` | ALWAYS set to `postgresdb` for production |
| `DB_TABLE_PREFIX` | — | Table name prefix |
| `DB_PING_INTERVAL_SECONDS` | `2` | Database connection health check interval |
| `DB_POSTGRESDB_DATABASE` | `n8n` | Database name |
| `DB_POSTGRESDB_HOST` | `localhost` | PostgreSQL host |
| `DB_POSTGRESDB_PORT` | `5432` | PostgreSQL port |
| `DB_POSTGRESDB_USER` | `postgres` | Database user |
| `DB_POSTGRESDB_PASSWORD` | — | Database password. Use `_FILE` suffix for secrets. |
| `DB_POSTGRESDB_SCHEMA` | `public` | Database schema |
| `DB_POSTGRESDB_POOL_SIZE` | `2` | Connection pool size |
| `DB_POSTGRESDB_CONNECTION_TIMEOUT` | `20000` | Connection timeout (ms) |
| `DB_POSTGRESDB_IDLE_CONNECTION_TIMEOUT` | `30000` | Idle connection timeout (ms) |

### PostgreSQL SSL

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_POSTGRESDB_SSL_ENABLED` | `false` | Enable SSL for database connection |
| `DB_POSTGRESDB_SSL_CA` | — | SSL CA certificate path |
| `DB_POSTGRESDB_SSL_CERT` | — | SSL client certificate path |
| `DB_POSTGRESDB_SSL_KEY` | — | SSL client key path |
| `DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED` | `true` | Reject unauthorized SSL connections |

### SQLite

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_SQLITE_POOL_SIZE` | `0` | Pool size: 0 = journal mode, >0 = WAL mode |
| `DB_SQLITE_VACUUM_ON_STARTUP` | `false` | Optimize SQLite database on startup |

---

## 3. Execution

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_MODE` | `regular` | `regular` (single instance) or `queue` (scaled) |
| `EXECUTIONS_TIMEOUT` | `-1` | Default workflow timeout in seconds (-1 = disabled) |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Maximum timeout users can set per workflow (seconds) |
| `EXECUTIONS_DATA_SAVE_ON_ERROR` | `all` | Save execution data on error: `all` or `none` |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `all` | Save execution data on success: `all` or `none` |
| `EXECUTIONS_DATA_SAVE_ON_PROGRESS` | `false` | Save progress after each node execution |
| `EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS` | `true` | Save manual (test) execution data |
| `EXECUTIONS_DATA_PRUNE` | `true` | Auto-delete old execution data |
| `EXECUTIONS_DATA_MAX_AGE` | `336` | Max execution age in hours (336 = 14 days) |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` | Max stored executions before pruning |
| `EXECUTIONS_DATA_HARD_DELETE_BUFFER` | `1` | Hours before hard deletion after soft delete |
| `EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL` | `15` | Hard delete check interval (minutes) |
| `EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL` | `60` | Soft delete check interval (minutes) |
| `N8N_CONCURRENCY_PRODUCTION_LIMIT` | `-1` | Max concurrent production executions (-1 = unlimited) |
| `N8N_AI_TIMEOUT_MAX` | `3600000` | AI/LLM node HTTP timeout in ms (1 hour default) |

---

## 4. Queue Mode & Redis

### Core Queue

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_MODE` | `regular` | MUST be `queue` to enable queue mode |
| `N8N_ENCRYPTION_KEY` | random | MUST be identical across ALL instances |

### Redis Connection

| Variable | Default | Description |
|----------|---------|-------------|
| `QUEUE_BULL_REDIS_HOST` | `localhost` | Redis host |
| `QUEUE_BULL_REDIS_PORT` | `6379` | Redis port |
| `QUEUE_BULL_REDIS_USERNAME` | — | Redis username (optional) |
| `QUEUE_BULL_REDIS_PASSWORD` | — | Redis password (optional) |
| `QUEUE_BULL_REDIS_DB` | `0` | Redis database number |
| `QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD` | `10000` | Redis unavailability tolerance (ms) |

### Queue Process Control

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_DISABLE_PRODUCTION_MAIN_PROCESS` | `false` | Disable webhook processing on main instance |
| `N8N_MULTI_MAIN_SETUP_ENABLED` | `false` | Enable multi-main HA deployment |
| `QUEUE_HEALTH_CHECK_ACTIVE` | — | Enable health check endpoints for workers |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Worker shutdown grace period (seconds) |

---

## 5. Credentials

| Variable | Default | Description |
|----------|---------|-------------|
| `CREDENTIALS_OVERWRITE_DATA` | — | JSON string to overwrite credential data |
| `CREDENTIALS_OVERWRITE_ENDPOINT` | — | API endpoint to fetch credential overwrites |
| `CREDENTIALS_OVERWRITE_PERSISTENCE` | `false` | Persist overwrites in DB (required for queue mode) |
| `CREDENTIALS_DEFAULT_NAME` | `My credentials` | Default name for new credentials |

---

## 6. Endpoints & Webhooks

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ENDPOINT_REST` | `rest` | REST API path prefix |
| `N8N_ENDPOINT_WEBHOOK` | `webhook` | Production webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_TEST` | `webhook-test` | Test webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_WAIT` | `webhook-waiting` | Waiting webhook path prefix |
| `N8N_ENDPOINT_HEALTH` | `healthz` | Health check endpoint path |
| `WEBHOOK_URL` | — | Manual webhook URL. ALWAYS set when behind reverse proxy. |
| `N8N_PAYLOAD_SIZE_MAX` | — | Maximum request payload size |
| `N8N_FORMDATA_FILE_SIZE_MAX` | — | Maximum form data file size |

---

## 7. Security

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Block env var access in expressions/Code node |
| `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES` | `true` | Block Code node access to `.n8n` directory |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` | `false` | Set 0600 permissions on settings file. ALWAYS enable in Docker. |
| `N8N_RESTRICT_FILE_ACCESS_TO` | — | Limit file access to specified dirs (semicolon-separated) |
| `N8N_SECURITY_AUDIT_DAYS_ABANDONED_WORKFLOW` | `90` | Days before workflow considered abandoned |
| `N8N_CONTENT_SECURITY_POLICY` | `{}` | CSP headers in helmet.js format |
| `N8N_SECURE_COOKIE` | `true` | HTTPS-only cookies |
| `N8N_SAMESITE_COOKIE` | `lax` | Cross-site cookie policy: `strict`, `lax`, `none` |
| `N8N_GIT_NODE_DISABLE_BARE_REPOS` | `false` | Prevent Git node from using bare repos |
| `N8N_GIT_NODE_ENABLE_HOOKS` | `false` | Allow Git node hooks (security risk) |

---

## 8. SSL/TLS

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_SSL_KEY` | — | Path to SSL private key file |
| `N8N_SSL_CERT` | — | Path to SSL certificate file |

---

## 9. Node Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_COMMUNITY_PACKAGES_ENABLED` | `true` | Allow community node installation |
| `N8N_COMMUNITY_PACKAGES_PREVENT_LOADING` | `false` | Prevent loading installed community nodes |
| `N8N_COMMUNITY_PACKAGES_REGISTRY` | `https://registry.npmjs.org` | NPM registry URL |
| `N8N_CUSTOM_EXTENSIONS` | — | Path to custom node directories |
| `N8N_PYTHON_ENABLED` | `true` | Enable Python in Code node |
| `NODE_FUNCTION_ALLOW_BUILTIN` | — | Allowed built-in Node.js modules in Code node (`*` for all) |
| `NODE_FUNCTION_ALLOW_EXTERNAL` | — | Allowed external npm modules in Code node |
| `NODES_ERROR_TRIGGER_TYPE` | `n8n-nodes-base.errorTrigger` | Error trigger node type |
| `NODES_EXCLUDE` | `["n8n-nodes-base.executeCommand", ...]` | Nodes to exclude from loading |
| `NODES_INCLUDE` | — | Nodes to include (whitelist mode) |

---

## 10. Workflow Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ONBOARDING_FLOW_DISABLED` | `false` | Disable onboarding tips |
| `N8N_WORKFLOW_ACTIVATION_BATCH_SIZE` | `1` | Workflows activated concurrently at startup |
| `N8N_WORKFLOW_CALLER_POLICY_DEFAULT_OPTION` | `workflowsFromSameOwner` | Default sub-workflow caller policy |
| `N8N_WORKFLOW_TAGS_DISABLED` | `false` | Disable workflow tags feature |
| `WORKFLOWS_DEFAULT_NAME` | `My workflow` | Default workflow name |
| `N8N_WORKFLOW_AUTODEACTIVATION_ENABLED` | `false` | Auto-unpublish after repeated crashes |
| `N8N_WORKFLOW_AUTODEACTIVATION_MAX_LAST_EXECUTIONS` | `3` | Crash count before auto-deactivation |

---

## 11. Proxy

| Variable | Default | Description |
|----------|---------|-------------|
| `HTTP_PROXY` | — | Proxy URL for HTTP requests |
| `HTTPS_PROXY` | — | Proxy URL for HTTPS requests |
| `ALL_PROXY` | — | Proxy URL for all requests |
| `NO_PROXY` | — | Comma-separated hostnames to bypass proxy |

---

## 12. Binary Data & External Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_DEFAULT_BINARY_DATA_MODE` | `default` | Storage mode: `default` (memory), `filesystem`, `s3` |
| `N8N_AVAILABLE_BINARY_DATA_MODES` | `filesystem` | Comma-separated available modes |
| `N8N_BINARY_DATA_STORAGE_PATH` | `<N8N_USER_FOLDER>/binaryData` | Filesystem storage path |
| `N8N_EXTERNAL_STORAGE_S3_HOST` | — | S3-compatible host URL |
| `N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME` | — | S3 bucket name |
| `N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION` | — | S3 bucket region |
| `N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY` | — | S3 access key |
| `N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET` | — | S3 access secret |
| `N8N_EXTERNAL_STORAGE_S3_AUTH_AUTO_DETECT` | — | Use AWS credential provider chain |

---

## 13. Task Runners

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_RUNNERS_ENABLED` | `false` | Enable task runners. ALWAYS set `true` in production. |
| `N8N_RUNNERS_MODE` | `internal` | Runner mode: `internal` or `external` |
| `N8N_RUNNERS_AUTH_TOKEN` | — | Shared auth secret for external runners |
| `N8N_RUNNERS_BROKER_PORT` | `5679` | Broker connection port |
| `N8N_RUNNERS_BROKER_LISTEN_ADDRESS` | `127.0.0.1` | Broker listen address |
| `N8N_RUNNERS_MAX_PAYLOAD` | `1073741824` | Max communication payload in bytes (1 GB) |
| `N8N_RUNNERS_MAX_OLD_SPACE_SIZE` | — | Node.js V8 heap size in MB |
| `N8N_RUNNERS_MAX_CONCURRENCY` | `5` | Max concurrent tasks per runner |
| `N8N_RUNNERS_TASK_TIMEOUT` | `300` | Task execution limit in seconds |
| `N8N_RUNNERS_HEARTBEAT_INTERVAL` | `30` | Health check frequency in seconds |

---

## 14. API

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_PUBLIC_API_DISABLED` | `false` | Disable public REST API |
| `N8N_PUBLIC_API_ENDPOINT` | `api` | Public API path prefix |
| `N8N_PUBLIC_API_SWAGGERUI_DISABLED` | `false` | Disable Swagger/Scalar UI |

---

## 15. Telemetry & Miscellaneous

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_TEMPLATES_ENABLED` | `false` | Enable workflow templates |
| `N8N_TEMPLATES_HOST` | `https://api.n8n.io` | Custom template library endpoint |
| `N8N_DIAGNOSTICS_ENABLED` | `true` | Enable anonymous telemetry |
| `N8N_VERSION_NOTIFICATIONS_ENABLED` | `true` | Version update notifications |
| `N8N_PERSONALIZATION_ENABLED` | `true` | Personalization survey |
| `N8N_HIRING_BANNER_ENABLED` | `true` | Hiring banner in UI |
| `N8N_REINSTALL_MISSING_PACKAGES` | `false` | Auto-reinstall missing community packages |
| `N8N_TUNNEL_SUBDOMAIN` | — | Custom tunnel subdomain (dev only) |
| `N8N_DEV_RELOAD` | `false` | Auto-reload during development |
| `N8N_PREVIEW_MODE` | `false` | Preview mode |
| `VUE_APP_URL_BASE_API` | `http://localhost:5678/` | Frontend API endpoint |

---

## 16. Metrics

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_METRICS` | — | Enable Prometheus metrics endpoint |
| `N8N_METRICS_PREFIX` | — | Custom prefix for metric names |
| `N8N_METRICS_INCLUDE_*` | — | Various flags to include specific metric types |

---

## Secret Management with `_FILE` Suffix

Most credential/connection variables support the `_FILE` suffix to load values from files instead of environment variables. This is the recommended approach for Docker Secrets and Kubernetes Secrets.

**Example:**
```yaml
environment:
  - DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password
  - N8N_ENCRYPTION_KEY_FILE=/run/secrets/encryption_key
```

ALWAYS prefer `_FILE` suffix over plain-text environment variables for sensitive values in production.
