# n8n REST API, Deployment & Operations Research

> Research date: 2026-03-19
> Sources: Official n8n documentation (docs.n8n.io), n8n GitHub repository (github.com/n8n-io/n8n)
> Scope: REST API, Docker deployment, queue mode, webhooks, error handling, environment variables, CLI

---

## 1. REST API

### 1.1 Overview

The n8n Public REST API enables programmatic management of workflows, executions, credentials, users, and more. The API follows REST conventions with JSON request/response bodies.

- **Base URL**: `<N8N_HOST>:<N8N_PORT>/<N8N_PATH>/api/v1/`
- **Authentication**: API key via `X-N8N-API-KEY` header
- **Pagination**: Cursor-based (NOT offset-based)
- **Content-Type**: `application/json`
- **Availability**: Requires a paid plan (not available during free trial)

### 1.2 Authentication

API keys are created in **Settings > n8n API**:

1. Navigate to Settings > n8n API
2. Select "Create API Key"
3. Provide a label and set an expiration period
4. On enterprise plans, select specific scopes for resource access
5. Copy the generated key

**Header format:**
```
X-N8N-API-KEY: <your-api-key>
```

**Example request (self-hosted):**
```bash
curl -X 'GET' \
  '<N8N_HOST>:<N8N_PORT>/<N8N_PATH>/api/v1/workflows?active=true' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <your-api-key>'
```

Non-enterprise keys grant full access to all account resources. Enterprise instances support scope-based restrictions.

### 1.3 Pagination

- **Type**: Cursor-based pagination
- **Default page size**: 100 results
- **Maximum page size**: 250 results
- **Parameters**: `limit` (page size), `cursor` (encoded string for next page)
- **Response fields**: `data` (array of results), `nextCursor` (encoded string for next page)

**Example:**
```
GET /api/v1/workflows?active=true&limit=150
→ Response includes { data: [...], nextCursor: "MTIzNA==" }

GET /api/v1/workflows?active=true&limit=150&cursor=MTIzNA==
→ Next page of results
```

### 1.4 Complete API Endpoint Reference

All endpoints are under `/api/v1/`. Authentication required for all endpoints.

#### Workflows

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/workflows` | `workflow:list` | List workflows (paginated). Query: `offset`, `limit`, `active`, `tags`, `name`, `projectId`, `excludePinnedData` |
| POST | `/workflows` | `workflow:create` | Create a new workflow |
| GET | `/workflows/:id` | `workflow:read` | Get single workflow. Query: `excludePinnedData` |
| PUT | `/workflows/:id` | `workflow:update` | Update workflow content |
| DELETE | `/workflows/:id` | `workflow:delete` | Delete workflow permanently |
| GET | `/workflows/:id/versions/:versionId` | `workflow:read` | Get specific workflow version |
| POST | `/workflows/:id/activate` | `workflow:activate` | Activate a workflow |
| POST | `/workflows/:id/deactivate` | `workflow:deactivate` | Deactivate a workflow |
| POST | `/workflows/:id/transfer` | `workflow:move` | Transfer workflow to another project. Body: `{ destinationProjectId }` |
| GET | `/workflows/:id/tags` | `workflowTags:list` | Get workflow tags |
| PUT | `/workflows/:id/tags` | `workflowTags:update` | Update workflow tags. Body: array of tag IDs |

#### Executions

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/executions` | `execution:list` | List executions (paginated). Query: `lastId`, `limit`, `status`, `workflowId`, `projectId`, `includeData` |
| GET | `/executions/:id` | `execution:read` | Get single execution. Query: `includeData` |
| DELETE | `/executions/:id` | `execution:delete` | Delete execution (cannot delete running) |
| POST | `/executions/:id/retry` | `execution:retry` | Retry a failed execution |
| POST | `/executions/:id/stop` | `execution:stop` | Stop a running execution |
| POST | `/executions/stop` | `execution:stop` | Stop multiple executions. Body: `{ status[], workflowId?, startedAfter?, startedBefore? }` |
| GET | `/executions/:id/tags` | `executionTags:list` | Get execution tags |
| PATCH | `/executions/:id/tags` | `executionTags:update` | Update execution tags |

#### Credentials

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/credentials` | `credential:list` | List credentials (paginated). Query: `offset`, `limit` |
| POST | `/credentials` | `credential:create` | Create new credential |
| PUT | `/credentials/:id` | `credential:update` | Update credential |
| DELETE | `/credentials/:id` | `credential:delete` | Delete credential |
| POST | `/credentials/:id/transfer` | `credential:move` | Transfer credential to another project |
| GET | `/credential-types/:credentialTypeName` | — | Get credential type schema |

#### Users

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users` | List users |
| GET | `/users/:id` | Get single user |
| — | `/users/:id/role` | Manage user role |

#### Tags

| Method | Path | Description |
|--------|------|-------------|
| GET | `/tags` | List tags |
| POST | `/tags` | Create tag |
| GET | `/tags/:id` | Get single tag |
| PUT/DELETE | `/tags/:id` | Update/delete tag |

#### Variables

| Method | Path | Description |
|--------|------|-------------|
| GET | `/variables` | List variables |
| POST | `/variables` | Create variable |
| GET | `/variables/:id` | Get single variable |
| PUT/DELETE | `/variables/:id` | Update/delete variable |

#### Projects

| Method | Path | Description |
|--------|------|-------------|
| GET | `/projects` | List projects |
| POST | `/projects` | Create project |
| GET | `/projects/:projectId` | Get single project |
| PUT/DELETE | `/projects/:projectId` | Update/delete project |
| GET | `/projects/:projectId/users` | List project users |
| PUT | `/projects/:projectId/users/:userId` | Manage project user |

#### Other

| Method | Path | Description |
|--------|------|-------------|
| POST | `/audit` | Run security audit |
| POST | `/source-control/pull` | Pull from source control |
| CRUD | `/data-tables/*` | Data table operations (rows, update, upsert, delete) |

### 1.5 API Configuration Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_PUBLIC_API_DISABLED` | `false` | Disable public API entirely |
| `N8N_PUBLIC_API_ENDPOINT` | `api` | Path for public API endpoints |
| `N8N_PUBLIC_API_SWAGGERUI_DISABLED` | `false` | Disable Swagger UI (API playground) |

The API playground (Scalar UI) is available at `<base-url>/api/v1/docs` on self-hosted instances.

---

## 2. Docker Deployment

### 2.1 Basic Docker Run

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Europe/Amsterdam" \
  -e TZ="Europe/Amsterdam" \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_RUNNERS_ENABLED=true \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

**Key points:**
- Docker image: `docker.n8n.io/n8nio/n8n`
- Default port: 5678
- Data volume: `/home/node/.n8n` (contains SQLite DB, encryption key, settings)
- Access: `http://localhost:5678`

### 2.2 Docker Run with PostgreSQL

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Europe/Amsterdam" \
  -e TZ="Europe/Amsterdam" \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_RUNNERS_ENABLED=true \
  -e DB_TYPE=postgresdb \
  -e DB_POSTGRESDB_DATABASE=n8n \
  -e DB_POSTGRESDB_HOST=postgres-host \
  -e DB_POSTGRESDB_PORT=5432 \
  -e DB_POSTGRESDB_USER=postgres \
  -e DB_POSTGRESDB_SCHEMA=public \
  -e DB_POSTGRESDB_PASSWORD=secret \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

### 2.3 Docker Compose with Traefik (Production)

**`.env` file:**
```env
DOMAIN_NAME=example.com
SUBDOMAIN=n8n
GENERIC_TIMEZONE=Europe/Berlin
SSL_EMAIL=user@example.com
```

**`docker-compose.yml`:**
```yaml
services:
  traefik:
    image: traefik:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    # Traefik labels for HTTPS redirect, Let's Encrypt ACME

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files

volumes:
  traefik_data:
  n8n_data:
```

**Launch:**
```bash
sudo docker compose up -d
```

### 2.4 Volume Mounts

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `n8n_data` | `/home/node/.n8n` | SQLite database, encryption key, settings |
| `traefik_data` | `/letsencrypt` | TLS/SSL certificates (Let's Encrypt) |
| `./local-files` | `/files` | Host directory for file sharing with workflows |

### 2.5 Updating Docker

```bash
# Pull latest version
docker pull docker.n8n.io/n8nio/n8n

# Pull specific version
docker pull docker.n8n.io/n8nio/n8n:1.81.0

# Pull next (pre-release) version
docker pull docker.n8n.io/n8nio/n8n:next

# Replace running container
docker ps -a
docker stop <container_id>
docker rm <container_id>
docker run --name=<container_name> [options] -d docker.n8n.io/n8nio/n8n
```

### 2.6 Configuration Methods

n8n supports three configuration methods:

1. **Environment variables** (primary method)
   - Bash: `export N8N_TEMPLATES_ENABLED=false`
   - Docker CLI: `-e N8N_TEMPLATES_ENABLED=false`
   - Docker Compose: under `environment:` section

2. **Docker Compose file** — variables in `docker-compose.yml`

3. **`_FILE` suffix** — load sensitive values from files (Docker Secrets, Kubernetes Secrets)
   - Example: `DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password`
   - Supported on most credential/connection variables

---

## 3. Queue Mode & Scaling

### 3.1 Architecture

Queue mode separates n8n into distinct components:
- **Main instance**: Handles triggers, webhooks, generates execution IDs (does NOT run workflows)
- **Worker instances**: Execute workflows retrieved from the queue
- **Redis**: Message broker for the execution queue (BullMQ)
- **Database**: Stores persistent data (PostgreSQL required)

**Flow:**
1. Main instance generates execution ID (without running the workflow)
2. Execution ID is passed to Redis queue
3. Available worker retrieves message from Redis
4. Worker fetches workflow details from database
5. Worker executes workflow and writes results back to database
6. Worker notifies Redis of completion

### 3.2 Requirements

- **PostgreSQL 13+** required (SQLite NOT supported with queue mode)
- **Redis** for message queue
- **S3-compatible storage** required for binary data in queue mode
- All instances MUST share the same n8n version
- All instances MUST share the same `N8N_ENCRYPTION_KEY`
- Load balancer with **session persistence** required for multi-main setups

### 3.3 Environment Variables

**Core queue configuration:**

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_MODE` | `regular` | Set to `queue` to enable queue mode |
| `N8N_ENCRYPTION_KEY` | random | Shared encryption key across ALL instances |

**Redis connection:**

| Variable | Default | Description |
|----------|---------|-------------|
| `QUEUE_BULL_REDIS_HOST` | `localhost` | Redis host |
| `QUEUE_BULL_REDIS_PORT` | `6379` | Redis port |
| `QUEUE_BULL_REDIS_USERNAME` | — | Redis username (optional) |
| `QUEUE_BULL_REDIS_PASSWORD` | — | Redis password (optional) |
| `QUEUE_BULL_REDIS_DB` | `0` | Redis database number |
| `QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD` | `10000` | Redis unavailability tolerance (ms) |

**Webhook and process configuration:**

| Variable | Default | Description |
|----------|---------|-------------|
| `WEBHOOK_URL` | — | Webhook routing URL |
| `N8N_DISABLE_PRODUCTION_MAIN_PROCESS` | `false` | Disable webhook processing on main |
| `N8N_MULTI_MAIN_SETUP_ENABLED` | `false` | Enable multi-main deployment |
| `QUEUE_HEALTH_CHECK_ACTIVE` | — | Enable health check endpoints |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Worker shutdown grace period (seconds) |

### 3.4 Redis Setup

```bash
docker run --name some-redis -p 6379:6379 -d redis
```

### 3.5 Worker Commands

**Native:**
```bash
n8n worker
n8n worker --concurrency=5
```

**Docker:**
```bash
docker run --name n8n-worker \
  -e EXECUTIONS_MODE=queue \
  -e QUEUE_BULL_REDIS_HOST=redis-host \
  -e N8N_ENCRYPTION_KEY=<shared-key> \
  docker.n8n.io/n8nio/n8n worker
```

### 3.6 Webhook Processor

Separate webhook processor for high-traffic setups:

**Native:**
```bash
n8n webhook
```

**Docker:**
```bash
docker run --name n8n-webhook -p 5679:5678 \
  -e EXECUTIONS_MODE=queue \
  docker.n8n.io/n8nio/n8n webhook
```

---

## 4. Webhooks

### 4.1 Webhook Node

The Webhook node is a trigger node that listens for incoming HTTP requests.

**Supported HTTP methods:** DELETE, GET, HEAD, PATCH, POST, PUT

### 4.2 Test vs Production URLs

| Type | When Active | Data Display | URL Path |
|------|-------------|--------------|----------|
| **Test URL** | During "Listen for Test Event" or manual execution | Shown in workflow editor | `<base>/webhook-test/<path>` |
| **Production URL** | When workflow is published/active | Only in Executions tab | `<base>/webhook/<path>` |

**URL endpoint configuration variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ENDPOINT_WEBHOOK` | `webhook` | Production webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_TEST` | `webhook-test` | Test webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_WAIT` | `webhook-waiting` | Waiting webhook path prefix |

### 4.3 Path Configuration

The Path field supports dynamic route parameters:
- `/:variable`
- `/path/:variable`
- `/:variable/path`
- `/:variable1/path/:variable2`
- `/:variable1/:variable2`

### 4.4 Response Modes

| Mode | Behavior |
|------|----------|
| **Immediately** | Returns response code and "Workflow got started" message |
| **When Last Node Finishes** | Returns response code and final node's data |
| **Using 'Respond to Webhook' Node** | Responds per Respond to Webhook node configuration |
| **Streaming** | Real-time data streaming for streaming-capable nodes |

**Response Data options (for "When Last Node Finishes"):**
- **All Entries**: Array of all last node entries
- **First Entry JSON**: JSON object of first entry
- **First Entry Binary**: Binary file of first entry
- **No Response Body**: Empty response

### 4.5 Authentication Options

| Method | Description |
|--------|-------------|
| **None** | No authentication |
| **Basic Auth** | Username/password authentication |
| **Header Auth** | Custom header-based authentication |
| **JWT Auth** | JSON Web Token authentication |

### 4.6 Additional Options

| Option | Description |
|--------|-------------|
| **Binary Data** | Enable binary property reception (images, audio, files) |
| **IP Whitelist** | Comma-separated allowed IPs; unlisted IPs receive 403 |
| **CORS** | Cross-origin domains; `*` allows all (default) |
| **Ignore Bots** | Filter out bot requests |
| **Raw Body** | Receive unformatted data (JSON, XML) |
| **Response Headers** | Custom headers in webhook responses |
| **Response Code** | Custom HTTP status codes |
| **Property Name** | Return specific JSON keys instead of all data |

**Payload limit**: 16MB maximum (configurable via `N8N_PAYLOAD_SIZE_MAX`)

### 4.7 Respond to Webhook Node

The Respond to Webhook node controls what data is sent back to the webhook caller. It pairs with the Webhook trigger node.

**Setup:**
1. Add a Webhook node as workflow trigger
2. Set Webhook node's **Respond** parameter to "Using 'Respond to Webhook' node"
3. Place the Respond to Webhook node after processing nodes

**Respond With options:**

| Option | Description |
|--------|-------------|
| **All Incoming Items** | Return all JSON items from input |
| **Binary File** | Return a binary file |
| **First Incoming Item** | Return first item's JSON |
| **JSON** | Return custom JSON from Response Body |
| **JWT Token** | Return a JSON Web Token |
| **No Data** | Empty response payload |
| **Redirect** | Redirect to specified URL |
| **Text** | Return text (sent as HTML by default) |

**Additional options:**
- **Response Code**: Custom HTTP status code
- **Response Headers**: Custom response headers
- **Put Response in Field**: Rename response data field
- **Enable Streaming**: For streaming-configured triggers

**Behavior notes:**
- The node runs ONCE, using the first incoming data item
- Skipping the node returns 200 with standard message
- Pre-execution errors return 500
- Multiple Respond nodes: only the first executes
- Non-webhook contexts: node is ignored
- Since v1.103.0: HTML responses are wrapped in `<iframe>` for security (sandboxing prevents JS access to top-level window and local storage)

---

## 5. Error Handling

### 5.1 Error Workflows

Each workflow can designate a dedicated error workflow in **Workflow Settings**. When the primary workflow fails, the error workflow automatically executes. One error workflow can serve multiple workflows.

**Setup:**
1. Create a new workflow starting with the **Error Trigger** node
2. Add notification/recovery logic (email, Slack, database logging, etc.)
3. In the main workflow, open **Workflow Settings**
4. Select the error workflow from the dropdown

The Error Trigger node receives error data containing execution context information.

### 5.2 Stop And Error Node

The **Stop And Error** node deliberately causes a workflow execution failure. This triggers the configured error workflow.

**Use cases:**
- Custom validation failures
- Business logic violations
- Forcing error workflow activation based on data conditions

### 5.3 Error Investigation

- Review executions for individual or multiple workflows
- Load data from previous executions for debugging
- Enable Log Streaming for real-time monitoring

### 5.4 Retry on Fail

Individual nodes support retry-on-fail configuration in their **Settings** tab:

- **Retry On Fail**: Enable/disable (boolean)
- **Max Tries**: Number of retry attempts (default: 3)
- **Wait Between Tries (ms)**: Delay between retries (default: 1000ms)

### 5.5 Continue On Fail

Each node has a **Continue On Fail** setting (in Settings tab) that, when enabled, allows the workflow to continue even if that node fails. The error data is passed to the next node in `$json.error` instead of stopping execution.

### 5.6 Execution Timeouts

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_TIMEOUT` | `-1` (disabled) | Default timeout for all workflows (seconds) |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Maximum timeout users can set per workflow (seconds) |

### 5.7 Error Handling Patterns

**Pattern 1: Error Workflow with Notifications**
```
Main Workflow → (fails) → Error Trigger → Slack/Email notification
```

**Pattern 2: Graceful Degradation**
```
HTTP Request (continueOnFail=true) → IF ($json.error) → Fallback path
                                                       → Success path
```

**Pattern 3: Retry with Fallback**
```
API Call (retryOnFail=true, maxTries=3) → (still fails) → Error Workflow
```

**Pattern 4: Deliberate Failure**
```
IF (invalid data) → Stop And Error → triggers Error Workflow
```

---

## 6. Environment Variables & Security

### 6.1 Deployment & Core

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_HOST` | `localhost` | Hostname |
| `N8N_PORT` | `5678` | HTTP port |
| `N8N_PROTOCOL` | `http` | Protocol (`http` or `https`) |
| `N8N_PATH` | `/` | Deployment path |
| `N8N_LISTEN_ADDRESS` | `::` | IP address to listen on |
| `N8N_ENCRYPTION_KEY` | random | Encryption key for credentials in database |
| `N8N_USER_FOLDER` | `user-folder` | Path for `.n8n` data folder |
| `N8N_EDITOR_BASE_URL` | — | Public URL for editor access |
| `N8N_DISABLE_UI` | `false` | Disable the web UI |
| `N8N_PUSH_BACKEND` | `websocket` | SSE or WebSocket for UI updates |
| `NODE_ENV` | — | Set to `production` for production deployments |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Shutdown grace period (seconds) |
| `N8N_PROXY_HOPS` | `0` | Number of reverse proxies in front of n8n |

### 6.2 Database

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_TYPE` | `sqlite` | Database type: `sqlite` or `postgresdb` |
| `DB_TABLE_PREFIX` | — | Table name prefix |
| `DB_PING_INTERVAL_SECONDS` | `2` | Database ping interval |
| `DB_POSTGRESDB_DATABASE` | `n8n` | PostgreSQL database name |
| `DB_POSTGRESDB_HOST` | `localhost` | PostgreSQL host |
| `DB_POSTGRESDB_PORT` | `5432` | PostgreSQL port |
| `DB_POSTGRESDB_USER` | `postgres` | PostgreSQL user |
| `DB_POSTGRESDB_PASSWORD` | — | PostgreSQL password |
| `DB_POSTGRESDB_SCHEMA` | `public` | PostgreSQL schema |
| `DB_POSTGRESDB_POOL_SIZE` | `2` | Connection pool size |
| `DB_POSTGRESDB_CONNECTION_TIMEOUT` | `20000` | Connection timeout (ms) |
| `DB_POSTGRESDB_IDLE_CONNECTION_TIMEOUT` | `30000` | Idle connection timeout (ms) |
| `DB_POSTGRESDB_SSL_ENABLED` | `false` | Enable SSL |
| `DB_POSTGRESDB_SSL_CA` | — | SSL CA certificate |
| `DB_POSTGRESDB_SSL_CERT` | — | SSL certificate |
| `DB_POSTGRESDB_SSL_KEY` | — | SSL key |
| `DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED` | `true` | Reject unauthorized SSL |
| `DB_SQLITE_POOL_SIZE` | `0` | SQLite pool (0=journal, >0=WAL mode) |
| `DB_SQLITE_VACUUM_ON_STARTUP` | `false` | Optimize SQLite on startup |

### 6.3 Execution

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_MODE` | `regular` | `regular` or `queue` |
| `EXECUTIONS_TIMEOUT` | `-1` | Default workflow timeout (seconds) |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Max timeout per workflow (seconds) |
| `EXECUTIONS_DATA_SAVE_ON_ERROR` | `all` | Save execution data on error |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `all` | Save execution data on success |
| `EXECUTIONS_DATA_SAVE_ON_PROGRESS` | `false` | Save progress per node |
| `EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS` | `true` | Save manual execution data |
| `EXECUTIONS_DATA_PRUNE` | `true` | Auto-delete old executions |
| `EXECUTIONS_DATA_MAX_AGE` | `336` | Max execution age (hours) = 14 days |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` | Max execution count before pruning |
| `EXECUTIONS_DATA_HARD_DELETE_BUFFER` | `1` | Hours before hard deletion |
| `EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL` | `15` | Hard delete interval (minutes) |
| `EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL` | `60` | Soft delete interval (minutes) |
| `N8N_CONCURRENCY_PRODUCTION_LIMIT` | `-1` | Max concurrent production executions |
| `N8N_AI_TIMEOUT_MAX` | `3600000` | AI/LLM node HTTP timeout (ms) |

### 6.4 Credentials

| Variable | Default | Description |
|----------|---------|-------------|
| `CREDENTIALS_OVERWRITE_DATA` | — | Overwrite data for credentials |
| `CREDENTIALS_OVERWRITE_ENDPOINT` | — | API endpoint to fetch credentials |
| `CREDENTIALS_OVERWRITE_PERSISTENCE` | `false` | Enable DB persistence for overwrites (required for queue mode) |
| `CREDENTIALS_DEFAULT_NAME` | `My credentials` | Default credential name |

### 6.5 Endpoints & Webhooks

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ENDPOINT_REST` | `rest` | REST API path |
| `N8N_ENDPOINT_WEBHOOK` | `webhook` | Production webhook path |
| `N8N_ENDPOINT_WEBHOOK_TEST` | `webhook-test` | Test webhook path |
| `N8N_ENDPOINT_WEBHOOK_WAIT` | `webhook-waiting` | Waiting webhook path |
| `N8N_ENDPOINT_HEALTH` | `healthz` | Health check endpoint path |
| `WEBHOOK_URL` | — | Manual webhook URL (for reverse proxies) |
| `N8N_DISABLE_PRODUCTION_MAIN_PROCESS` | `false` | Disable production webhooks on main |
| `N8N_PAYLOAD_SIZE_MAX` | — | Maximum request payload size |
| `N8N_FORMDATA_FILE_SIZE_MAX` | — | Maximum form data file size |

### 6.6 Security

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Block environment variable access in expressions/Code node |
| `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES` | `true` | Block access to `.n8n` directory |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` | `false` | Set 0600 permissions on settings file |
| `N8N_RESTRICT_FILE_ACCESS_TO` | — | Limit file access to specified directories (semicolon-separated) |
| `N8N_SECURITY_AUDIT_DAYS_ABANDONED_WORKFLOW` | `90` | Days before workflow considered abandoned |
| `N8N_CONTENT_SECURITY_POLICY` | `{}` | CSP headers (helmet.js format) |
| `N8N_SECURE_COOKIE` | `true` | HTTPS-only cookies |
| `N8N_SAMESITE_COOKIE` | `lax` | Cross-site cookie policy (`strict`, `lax`, `none`) |
| `N8N_GIT_NODE_DISABLE_BARE_REPOS` | `false` | Prevent Git node bare repos |
| `N8N_GIT_NODE_ENABLE_HOOKS` | `false` | Allow Git node hooks |

### 6.7 SSL/TLS

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_SSL_KEY` | — | SSL key file path |
| `N8N_SSL_CERT` | — | SSL certificate file path |

### 6.8 Node Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_COMMUNITY_PACKAGES_ENABLED` | `true` | Enable community node installation |
| `N8N_COMMUNITY_PACKAGES_PREVENT_LOADING` | `false` | Prevent loading installed community nodes |
| `N8N_COMMUNITY_PACKAGES_REGISTRY` | `https://registry.npmjs.org` | NPM registry for community packages |
| `N8N_CUSTOM_EXTENSIONS` | — | Path to custom node directories |
| `N8N_PYTHON_ENABLED` | `true` | Enable Python in Code node |
| `NODE_FUNCTION_ALLOW_BUILTIN` | — | Allowed built-in modules in Code node (`*` for all) |
| `NODE_FUNCTION_ALLOW_EXTERNAL` | — | Allowed external modules in Code node |
| `NODES_ERROR_TRIGGER_TYPE` | `n8n-nodes-base.errorTrigger` | Error trigger node type |
| `NODES_EXCLUDE` | `["n8n-nodes-base.executeCommand", "n8n-nodes-base.localFileTrigger"]` | Nodes to exclude from loading |
| `NODES_INCLUDE` | — | Nodes to include (whitelist) |

### 6.9 Workflow Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ONBOARDING_FLOW_DISABLED` | `false` | Disable onboarding tips |
| `N8N_WORKFLOW_ACTIVATION_BATCH_SIZE` | `1` | Workflows activated concurrently at startup |
| `N8N_WORKFLOW_CALLER_POLICY_DEFAULT_OPTION` | `workflowsFromSameOwner` | Default sub-workflow caller policy |
| `N8N_WORKFLOW_TAGS_DISABLED` | `false` | Disable workflow tags |
| `WORKFLOWS_DEFAULT_NAME` | `My workflow` | Default workflow name |
| `N8N_WORKFLOW_AUTODEACTIVATION_ENABLED` | `false` | Auto-unpublish after repeated crashes |
| `N8N_WORKFLOW_AUTODEACTIVATION_MAX_LAST_EXECUTIONS` | `3` | Crash count before auto-deactivation |

### 6.10 Proxy

| Variable | Default | Description |
|----------|---------|-------------|
| `HTTP_PROXY` | — | Proxy for HTTP requests |
| `HTTPS_PROXY` | — | Proxy for HTTPS requests |
| `ALL_PROXY` | — | Proxy for all requests |
| `NO_PROXY` | — | Comma-separated hostnames to bypass proxy |

### 6.11 Binary Data & External Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_DEFAULT_BINARY_DATA_MODE` | `default` | Storage mode: `default` (memory), `filesystem`, `s3` |
| `N8N_AVAILABLE_BINARY_DATA_MODES` | `filesystem` | Comma-separated available modes |
| `N8N_BINARY_DATA_STORAGE_PATH` | `<N8N_USER_FOLDER>/binaryData` | Filesystem storage path |
| `N8N_EXTERNAL_STORAGE_S3_HOST` | — | S3 host |
| `N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME` | — | S3 bucket name |
| `N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION` | — | S3 bucket region |
| `N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY` | — | S3 access key |
| `N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET` | — | S3 access secret |
| `N8N_EXTERNAL_STORAGE_S3_AUTH_AUTO_DETECT` | — | Use AWS credential provider chain |

### 6.12 Task Runners

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_RUNNERS_ENABLED` | `false` | Enable task runners |
| `N8N_RUNNERS_MODE` | `internal` | `internal` or `external` |
| `N8N_RUNNERS_AUTH_TOKEN` | — | Shared auth secret for runners |
| `N8N_RUNNERS_BROKER_PORT` | `5679` | Broker connection port |
| `N8N_RUNNERS_BROKER_LISTEN_ADDRESS` | `127.0.0.1` | Broker listen address |
| `N8N_RUNNERS_MAX_PAYLOAD` | `1073741824` | Max communication payload (bytes) |
| `N8N_RUNNERS_MAX_OLD_SPACE_SIZE` | — | Node.js memory allocation (MB) |
| `N8N_RUNNERS_MAX_CONCURRENCY` | `5` | Concurrent task capacity |
| `N8N_RUNNERS_TASK_TIMEOUT` | `300` | Task execution limit (seconds) |
| `N8N_RUNNERS_HEARTBEAT_INTERVAL` | `30` | Health check frequency (seconds) |

### 6.13 Telemetry & Misc

| Variable | Default | Description |
|----------|---------|-------------|
| `GENERIC_TIMEZONE` | — | Timezone for schedule nodes |
| `TZ` | — | System timezone |
| `N8N_TEMPLATES_ENABLED` | `false` | Enable workflow templates |
| `N8N_TEMPLATES_HOST` | `https://api.n8n.io` | Custom template library endpoint |
| `N8N_DIAGNOSTICS_ENABLED` | `true` | Enable anonymous telemetry |
| `N8N_VERSION_NOTIFICATIONS_ENABLED` | `true` | Version update notifications |
| `N8N_PERSONALIZATION_ENABLED` | `true` | Personalization questions |
| `N8N_HIRING_BANNER_ENABLED` | `true` | Hiring banner visibility |
| `N8N_REINSTALL_MISSING_PACKAGES` | `false` | Auto-reinstall missing packages |
| `N8N_TUNNEL_SUBDOMAIN` | — | Custom tunnel subdomain |
| `N8N_DEV_RELOAD` | `false` | Auto-reload during development |
| `N8N_PREVIEW_MODE` | `false` | Preview mode |
| `VUE_APP_URL_BASE_API` | `http://localhost:5678/` | Frontend API endpoint |

### 6.14 Metrics

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_METRICS` | — | Enable Prometheus metrics |
| `N8N_METRICS_PREFIX` | — | Metrics name prefix |
| `N8N_METRICS_INCLUDE_*` | — | Various include flags for metric types |

---

## 7. CLI & Backup

### 7.1 Workflow Execution

```bash
# Execute a saved workflow by ID
n8n execute --id <ID>
```

### 7.2 Workflow Status Management

```bash
# Activate/deactivate single workflow
n8n update:workflow --id=<ID> --active=false
n8n update:workflow --id=<ID> --active=true

# Activate/deactivate all workflows
n8n update:workflow --all --active=false
n8n update:workflow --all --active=true
```

### 7.3 Workflow Export

```bash
# Export all workflows to terminal
n8n export:workflow --all

# Export by ID to file
n8n export:workflow --id=<ID> --output=file.json

# Export all to directory
n8n export:workflow --all --output=backups/latest/file.json

# Backup (convenience flag)
n8n export:workflow --backup --output=backups/latest/

# Flags: --help, --all, --backup, --id, --output, --pretty, --separate
```

### 7.4 Credential Export

```bash
# Export all credentials (encrypted)
n8n export:credentials --all

# Export by ID
n8n export:credentials --id=<ID> --output=file.json

# Export all to directory
n8n export:credentials --all --output=backups/latest/file.json

# Backup credentials
n8n export:credentials --backup --output=backups/latest/

# Export DECRYPTED credentials (security risk!)
n8n export:credentials --all --decrypted --output=backups/decrypted.json

# Flags: --help, --all, --backup, --id, --output, --pretty, --separate, --decrypted
```

### 7.5 Entity Export (Combined)

```bash
# Export all entities with execution history
n8n export:entities --outputDir=./outputs --includeExecutionHistoryDataTables=true

# Flags: --help, --outputDir, --includeExecutionHistoryDataTables
```

### 7.6 Workflow Import

```bash
# Import from file
n8n import:workflow --input=file.json

# Import from directory
n8n import:workflow --separate --input=backups/latest/

# Flags: --help, --input, --projectId, --separate, --userId, --skipMigrationChecks
```

### 7.7 Credential Import

```bash
# Import from file
n8n import:credentials --input=file.json

# Import from directory
n8n import:credentials --separate --input=backups/latest/
```

### 7.8 Entity Import (Combined)

```bash
# Import all entities
n8n import:entities --inputDir ./outputs --truncateTables true

# Flags: --help, --inputDir, --truncateTables
```

### 7.9 License Management

```bash
# Clear license
n8n license:clear

# Display license info
n8n license:info
```

### 7.10 User Management

```bash
# Reset user management
n8n user-management:reset

# Disable MFA for a user
n8n mfa:disable --email=johndoe@example.com

# Reset LDAP settings
n8n ldap:reset
```

### 7.11 Community Node Management

```bash
# Uninstall community node
n8n community-node --uninstall --package <COMMUNITY_NODE_NAME>

# Uninstall community node credential
n8n community-node --uninstall --credential <CREDENTIAL_TYPE> --userId <ID>
```

### 7.12 Security Audit

```bash
# Run security audit
n8n audit
```

### 7.13 Backup Strategies

**Strategy 1: CLI Export (Recommended for small instances)**
```bash
# Daily backup script
n8n export:workflow --backup --output=/backups/workflows/
n8n export:credentials --backup --output=/backups/credentials/
```

**Strategy 2: Database Backup (Recommended for production)**
```bash
# PostgreSQL dump
pg_dump -U postgres n8n > /backups/n8n-$(date +%Y%m%d).sql

# With Docker
docker exec postgres pg_dump -U postgres n8n > /backups/n8n-$(date +%Y%m%d).sql
```

**Strategy 3: Volume Backup (SQLite)**
```bash
# Stop n8n, backup volume
docker stop n8n
docker run --rm -v n8n_data:/data -v $(pwd)/backup:/backup alpine tar czf /backup/n8n-data.tar.gz /data
docker start n8n
```

**Critical backup items:**
- Workflow definitions (JSON)
- Credentials (encrypted — need same `N8N_ENCRYPTION_KEY` to restore)
- `N8N_ENCRYPTION_KEY` itself (stored in `.n8n/config` or as env var)
- Database (PostgreSQL dump or SQLite file)
- Binary data (filesystem or S3)

---

## Appendix: Quick Reference

### Default Ports

| Service | Port | Description |
|---------|------|-------------|
| n8n | 5678 | Main HTTP port |
| Redis | 6379 | Queue message broker |
| PostgreSQL | 5432 | Database |
| Task Runner Broker | 5679 | Task runner communication |
| Launcher Health Check | 5680 | Task runner health |

### Minimum Production Setup Checklist

1. PostgreSQL database (not SQLite)
2. `N8N_ENCRYPTION_KEY` explicitly set and backed up
3. `NODE_ENV=production`
4. `N8N_PROTOCOL=https` with TLS termination
5. `WEBHOOK_URL` set to public URL
6. `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`
7. `N8N_RUNNERS_ENABLED=true`
8. Execution data pruning configured
9. Regular backups (database + encryption key)
10. Reverse proxy (Traefik, nginx, Caddy) for TLS

### SQLite vs PostgreSQL Decision

| Factor | SQLite | PostgreSQL |
|--------|--------|------------|
| Setup complexity | Minimal | Moderate |
| Queue mode support | NO | YES |
| Concurrent access | Limited | Full |
| Production recommended | No (single user/dev) | Yes |
| Backup | File copy | pg_dump |
| Scaling | Single instance only | Multi-instance |
