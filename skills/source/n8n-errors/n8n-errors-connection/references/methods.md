# Connection Error Types, Retry Configuration & Timeout Settings

> Reference for n8n-errors-connection skill. Detailed configuration for all connection-related settings.

## 1. Connection Error Types

### 1.1 Node.js Network Errors

| Error Code | Full Name | Description | Typical Cause |
|------------|-----------|-------------|---------------|
| `ECONNREFUSED` | Connection Refused | Target actively refused the connection | Service not running, wrong port, firewall |
| `ENOTFOUND` | DNS Not Found | Hostname could not be resolved | Typo in URL, DNS misconfiguration |
| `ETIMEDOUT` | Connection Timed Out | No response within TCP timeout | Network unreachable, firewall drop rule |
| `ESOCKETTIMEDOUT` | Socket Timed Out | Socket-level timeout after connection | Slow response from target service |
| `ECONNRESET` | Connection Reset | Remote peer forcefully closed connection | Network instability, target crash mid-request |
| `EPIPE` | Broken Pipe | Write to closed connection | Target closed before n8n finished sending |
| `EHOSTUNREACH` | Host Unreachable | No route to host | Network configuration, VPN not connected |
| `EAI_AGAIN` | DNS Temporary Failure | DNS lookup temporarily failed | DNS server overloaded, intermittent issue |

### 1.2 HTTP Status Code Errors

| Code | Meaning | n8n Behavior | Resolution Strategy |
|------|---------|--------------|---------------------|
| 400 | Bad Request | Node fails with error details | Fix request payload or parameters |
| 401 | Unauthorized | Node fails, credential error | Re-verify credentials, regenerate API key |
| 403 | Forbidden | Node fails, permission error | Check API key scopes, IP allowlists |
| 404 | Not Found | Node fails | Check URL path, resource existence |
| 408 | Request Timeout | Node fails | Increase timeout, check target service |
| 429 | Too Many Requests | Node fails unless retry configured | Enable Retry On Fail, add delays |
| 500 | Internal Server Error | Node fails | Target service issue, retry may help |
| 502 | Bad Gateway | Node fails | Upstream service unavailable |
| 503 | Service Unavailable | Node fails | Target overloaded or in maintenance |
| 504 | Gateway Timeout | Node fails | Target too slow, increase upstream timeout |

### 1.3 SSL/TLS Errors

| Error | Description | Environment Variable Fix |
|-------|-------------|--------------------------|
| `UNABLE_TO_VERIFY_LEAF_SIGNATURE` | Self-signed cert or missing intermediate | `NODE_EXTRA_CA_CERTS=/path/to/ca.pem` |
| `DEPTH_ZERO_SELF_SIGNED_CERT` | Self-signed without CA | `NODE_EXTRA_CA_CERTS=/path/to/cert.pem` |
| `CERT_HAS_EXPIRED` | Target certificate expired | Contact target service owner |
| `ERR_TLS_CERT_ALTNAME_INVALID` | Hostname mismatch | Fix URL to match certificate SAN |
| `CERT_UNTRUSTED` | Unknown certificate authority | Add CA to `NODE_EXTRA_CA_CERTS` |

### 1.4 Database Connection Errors

| Error | Database | Description |
|-------|----------|-------------|
| `ECONNREFUSED` on 5432 | PostgreSQL | Database server not running |
| `password authentication failed` | PostgreSQL | Wrong credentials |
| `too many connections` | PostgreSQL | Pool exhaustion |
| `SSL connection required` | PostgreSQL | SSL not configured |
| `SQLITE_BUSY` | SQLite | Write lock contention |
| `SQLITE_CORRUPT` | SQLite | File corruption |
| `SQLITE_READONLY` | SQLite | Permission issue on file |

### 1.5 Redis Connection Errors (Queue Mode)

| Error | Description | Fix |
|-------|-------------|-----|
| `ECONNREFUSED` on 6379 | Redis not running | Start Redis, check `QUEUE_BULL_REDIS_HOST` |
| `NOAUTH Authentication required` | Redis requires password | Set `QUEUE_BULL_REDIS_PASSWORD` |
| `ERR AUTH invalid password` | Wrong Redis password | Correct `QUEUE_BULL_REDIS_PASSWORD` |
| `ETIMEDOUT` | Redis unreachable | Check network, increase `QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD` |
| `MOVED` | Redis cluster redirect | ALWAYS use Redis Cluster-aware connection settings |

---

## 2. Retry Configuration

### 2.1 Node-Level Retry (Per Node)

Configure in the **Settings** tab of any node:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| Retry On Fail | Boolean | `false` | Enable automatic retry on node failure |
| Max Tries | Number | `3` | Total number of attempts (including first) |
| Wait Between Tries (ms) | Number | `1000` | Milliseconds to wait between retry attempts |

**When to enable:**
- ALWAYS enable on HTTP Request nodes calling external APIs
- ALWAYS enable on database query nodes in high-traffic workflows
- NEVER enable on nodes that modify data without idempotency (risk of duplicates)

### 2.2 Continue On Fail (Per Node)

Configure in the **Settings** tab of any node:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| Continue On Fail | Boolean | `false` | Continue workflow execution even if node fails |

When enabled, error data is available in `$json.error` on the next node. ALWAYS use an IF node after to check for errors.

### 2.3 Execution Retry (API)

Failed executions can be retried via the REST API:

```
POST /api/v1/executions/:id/retry
```

Or from the Executions UI: open the failed execution and click "Retry".

### 2.4 Error Workflow (Instance-Level)

Configure per workflow in **Workflow Settings > Error Workflow**:

1. Create a workflow with an **Error Trigger** node
2. Add notification/recovery logic
3. In the main workflow settings, select the error workflow

The Error Trigger receives execution context data including the error message, workflow ID, and execution ID.

---

## 3. Timeout Settings

### 3.1 Execution Timeouts

| Variable | Default | Scope | Description |
|----------|---------|-------|-------------|
| `EXECUTIONS_TIMEOUT` | `-1` | Instance | Default timeout for ALL workflows (seconds). `-1` = disabled. |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Instance | Maximum timeout any workflow can set (seconds) |
| Workflow timeout | Inherits | Workflow | Set in Workflow Settings, capped by `EXECUTIONS_TIMEOUT_MAX` |

### 3.2 Database Timeouts

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_POSTGRESDB_CONNECTION_TIMEOUT` | `20000` | Connection establishment timeout (ms) |
| `DB_POSTGRESDB_IDLE_CONNECTION_TIMEOUT` | `30000` | Idle connection timeout (ms) |
| `DB_PING_INTERVAL_SECONDS` | `2` | Keep-alive ping interval (seconds) |

### 3.3 Redis Timeouts (Queue Mode)

| Variable | Default | Description |
|----------|---------|-------------|
| `QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD` | `10000` | Redis unavailability tolerance (ms) |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Worker shutdown grace period (seconds) |

### 3.4 Task Runner Timeouts

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_RUNNERS_TASK_TIMEOUT` | `300` | Individual task execution limit (seconds) |
| `N8N_RUNNERS_HEARTBEAT_INTERVAL` | `30` | Health check frequency (seconds) |

### 3.5 AI/LLM Node Timeout

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_AI_TIMEOUT_MAX` | `3600000` | AI/LLM node HTTP timeout (ms) = 1 hour |

---

## 4. Proxy Configuration

| Variable | Description |
|----------|-------------|
| `HTTP_PROXY` | Proxy for HTTP requests |
| `HTTPS_PROXY` | Proxy for HTTPS requests |
| `ALL_PROXY` | Proxy for all requests |
| `NO_PROXY` | Comma-separated hostnames to bypass proxy |

ALWAYS set `NO_PROXY` to include internal service hostnames (database, Redis, internal APIs) to prevent proxy loops.

---

## 5. Connection Pool Settings

### 5.1 PostgreSQL Pool

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_POSTGRESDB_POOL_SIZE` | `2` | Maximum number of connections in pool |

**Sizing guidance:**
- Development: `2` (default)
- Production single instance: `10-20`
- Production with queue mode: `5` per instance (main + each worker)
- NEVER exceed PostgreSQL `max_connections` across all instances

### 5.2 SQLite Pool

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_SQLITE_POOL_SIZE` | `0` | `0` = journal mode, `>0` = WAL mode with pool |

ALWAYS use `DB_SQLITE_POOL_SIZE=4` or higher if using SQLite in development with concurrent access.
