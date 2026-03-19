# Connection Handling Anti-Patterns

> Mistakes to NEVER make when handling connections in n8n v1.x.

---

## Anti-Pattern 1: Disabling TLS Verification in Production

**What people do**: Set `NODE_TLS_REJECT_UNAUTHORIZED=0` to "fix" SSL errors.

**Why it is wrong**: This disables ALL certificate validation for ALL connections, including to databases and external APIs. Any man-in-the-middle attack succeeds silently.

**What to do instead**: ALWAYS add the specific CA certificate using `NODE_EXTRA_CA_CERTS`:
```env
NODE_EXTRA_CA_CERTS=/certs/your-ca.pem
```

---

## Anti-Pattern 2: Using Test Webhook URLs in Production

**What people do**: Copy the test URL (`/webhook-test/...`) from the editor and paste it into external services.

**Why it is wrong**: Test URLs ONLY work while the editor has "Listen for Test Event" active. They stop working as soon as you close the editor or stop listening.

**What to do instead**: ALWAYS use the production URL (`/webhook/...`) for any external integration. ALWAYS activate the workflow.

---

## Anti-Pattern 3: No Retry Configuration on External API Calls

**What people do**: Leave HTTP Request nodes with default settings (no retry) and wonder why workflows fail intermittently.

**Why it is wrong**: External APIs experience transient failures (network blips, temporary overload, rate limits). A single attempt means any transient failure stops the workflow.

**What to do instead**: ALWAYS enable Retry On Fail on HTTP Request nodes calling external APIs:
- Retry On Fail: `true`
- Max Tries: `3`
- Wait Between Tries: `1000` ms (increase for rate-limited APIs)

---

## Anti-Pattern 4: SQLite in Production with Concurrent Workflows

**What people do**: Deploy n8n with default SQLite database and run dozens of concurrent workflows.

**Why it is wrong**: SQLite does not handle concurrent writes. This causes `SQLITE_BUSY` errors and workflow failures.

**What to do instead**: ALWAYS use PostgreSQL in production:
```env
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_DATABASE=n8n
```
NEVER use SQLite with queue mode or when expecting concurrent workflow executions.

---

## Anti-Pattern 5: Forgetting WEBHOOK_URL Behind Reverse Proxy

**What people do**: Deploy n8n behind nginx/Traefik without setting `WEBHOOK_URL`.

**Why it is wrong**: n8n generates webhook URLs using its internal address (e.g., `http://localhost:5678/webhook/...`). External services cannot reach localhost.

**What to do instead**: ALWAYS set `WEBHOOK_URL` to the public-facing URL:
```env
WEBHOOK_URL=https://n8n.yourdomain.com/
```

---

## Anti-Pattern 6: Same Encryption Key Not Shared Across Instances

**What people do**: Deploy queue mode workers without setting `N8N_ENCRYPTION_KEY`, letting each instance generate its own.

**Why it is wrong**: Workers cannot decrypt credentials stored by the main instance. All credential-using nodes fail on workers.

**What to do instead**: ALWAYS set the SAME `N8N_ENCRYPTION_KEY` on ALL instances:
```env
# MUST be identical on main instance AND every worker
N8N_ENCRYPTION_KEY=your-shared-encryption-key
```

---

## Anti-Pattern 7: No Execution Timeout in Production

**What people do**: Leave `EXECUTIONS_TIMEOUT=-1` (disabled) in production.

**Why it is wrong**: A single stuck workflow (waiting for an unresponsive API, infinite loop) consumes resources indefinitely. In queue mode, it blocks a worker slot permanently.

**What to do instead**: ALWAYS set an execution timeout in production:
```env
EXECUTIONS_TIMEOUT=3600       # 1 hour default timeout
EXECUTIONS_TIMEOUT_MAX=7200   # 2 hour maximum
```

---

## Anti-Pattern 8: Ignoring Connection Pool Sizing

**What people do**: Keep `DB_POSTGRESDB_POOL_SIZE=2` (default) in production with many concurrent workflows.

**Why it is wrong**: With only 2 connections, workflows queue up waiting for a database connection, causing slowdowns and timeouts.

**What to do instead**: Size the pool based on concurrency:
```env
# Production single instance: 10-20
DB_POSTGRESDB_POOL_SIZE=10

# Queue mode: 5 per instance
DB_POSTGRESDB_POOL_SIZE=5
```
NEVER exceed PostgreSQL `max_connections` across all instances combined.

---

## Anti-Pattern 9: Processing Large Batches Without Rate Limit Awareness

**What people do**: Send 1000 items through an HTTP Request node in a loop without any delay.

**Why it is wrong**: Most APIs have rate limits. Exceeding them results in 429 errors and potentially temporary IP bans.

**What to do instead**: ALWAYS use Split In Batches + Wait:
1. **Split In Batches** node: set batch size within API rate limits
2. **Wait** node: add delay between batches (e.g., 1-2 seconds)
3. **Retry On Fail**: enable on the API call node as a safety net

---

## Anti-Pattern 10: Not Setting NO_PROXY for Internal Services

**What people do**: Set `HTTP_PROXY`/`HTTPS_PROXY` for corporate network access but forget `NO_PROXY`.

**Why it is wrong**: ALL traffic goes through the proxy, including connections to Redis, PostgreSQL, and internal services. This causes connection failures or massive latency.

**What to do instead**: ALWAYS set `NO_PROXY` to include all internal service hostnames:
```env
HTTP_PROXY=http://proxy.corp.com:8080
HTTPS_PROXY=http://proxy.corp.com:8080
NO_PROXY=localhost,127.0.0.1,redis,postgres,internal-api.local
```

---

## Anti-Pattern 11: Using Continue On Fail Without Error Checking

**What people do**: Enable "Continue On Fail" on a node but do not check for errors downstream.

**Why it is wrong**: The workflow silently continues with error data in `$json.error`. Downstream nodes process invalid data, causing incorrect results or data corruption.

**What to do instead**: ALWAYS add an IF node immediately after any node with Continue On Fail:
```
HTTP Request (continueOnFail=true) → IF ($json.error exists)
                                      ├── TRUE → Error handling path
                                      └── FALSE → Success path
```

---

## Anti-Pattern 12: Hardcoding Credentials in Workflow JSON

**What people do**: Put API keys directly in HTTP Request node URL or headers instead of using n8n credentials.

**Why it is wrong**: Credentials in workflow JSON are visible to anyone with workflow access. They are exported in backups and version control. They cannot be rotated without editing every workflow.

**What to do instead**: ALWAYS use n8n's credential system:
1. Create credentials in Settings > Credentials
2. Select the credential in the node's authentication settings
3. Credentials are encrypted in the database with `N8N_ENCRYPTION_KEY`
