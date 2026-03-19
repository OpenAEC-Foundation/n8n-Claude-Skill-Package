# Connection Error Scenarios with Fixes

> Step-by-step resolution examples for common n8n v1.x connection errors.

---

## Scenario 1: Webhook Works in Editor but Fails in Production

**Symptom**: External service sends POST to webhook URL, gets 404. Webhook works when testing in the n8n editor.

**Root Cause**: Using the test URL (`/webhook-test/`) instead of the production URL (`/webhook/`).

**Fix**:
1. Open the Webhook node in the workflow editor
2. Copy the **Production URL** (not the Test URL)
3. Replace the URL in the external service configuration
4. ALWAYS activate the workflow (toggle to "Active")
5. Verify the URL follows the pattern: `https://your-n8n-domain.com/webhook/<path>`

**Prevention**: NEVER share test URLs with external services. Test URLs ONLY work while the editor has "Listen for Test Event" active.

---

## Scenario 2: WEBHOOK_URL Misconfiguration Behind Reverse Proxy

**Symptom**: Webhook URLs in the n8n UI show `http://localhost:5678/webhook/...` instead of the public domain.

**Root Cause**: `WEBHOOK_URL` environment variable not set.

**Fix**:
```env
# Add to docker-compose.yml or environment
WEBHOOK_URL=https://n8n.yourdomain.com/
```

ALWAYS include the trailing slash. ALWAYS use the protocol that matches your public URL (https if behind TLS-terminating proxy).

---

## Scenario 3: API Returns 401 After Working Previously

**Symptom**: HTTP Request node that previously worked now returns 401 Unauthorized.

**Root Cause**: API key or OAuth token expired.

**Fix (API Key)**:
1. Go to the target service's dashboard
2. Regenerate the API key
3. In n8n: Settings > Credentials > select the credential
4. Update the API key value
5. Test the credential using the "Test" button

**Fix (OAuth2)**:
1. In n8n: Settings > Credentials > select the OAuth2 credential
2. Click "Reconnect" to re-authorize
3. Complete the OAuth flow in the popup window
4. Test the credential

---

## Scenario 4: Redis Connection Failure in Queue Mode

**Symptom**: n8n main instance starts but workers fail with `ECONNREFUSED` on port 6379.

**Root Cause**: Workers cannot reach Redis host.

**Fix**:
1. Verify Redis is running:
   ```bash
   redis-cli -h <redis-host> -p 6379 ping
   # Expected: PONG
   ```
2. Check Docker networking — workers MUST be on the same Docker network as Redis:
   ```yaml
   # docker-compose.yml
   services:
     redis:
       image: redis:7
       networks:
         - n8n-network
     n8n-worker:
       image: docker.n8n.io/n8nio/n8n
       command: worker
       environment:
         - QUEUE_BULL_REDIS_HOST=redis  # Use Docker service name
       networks:
         - n8n-network
   networks:
     n8n-network:
   ```
3. Verify environment variables on ALL worker instances:
   ```env
   EXECUTIONS_MODE=queue
   QUEUE_BULL_REDIS_HOST=redis
   QUEUE_BULL_REDIS_PORT=6379
   N8N_ENCRYPTION_KEY=<same-key-as-main>
   ```

---

## Scenario 5: PostgreSQL Connection Refused

**Symptom**: n8n fails to start with `ECONNREFUSED` on port 5432.

**Root Cause**: PostgreSQL not running, wrong host, or firewall blocking.

**Fix**:
1. Verify PostgreSQL is running:
   ```bash
   pg_isready -h <db-host> -p 5432
   # Expected: accepting connections
   ```
2. Check n8n environment variables:
   ```env
   DB_TYPE=postgresdb
   DB_POSTGRESDB_HOST=postgres  # Docker service name or hostname
   DB_POSTGRESDB_PORT=5432
   DB_POSTGRESDB_DATABASE=n8n
   DB_POSTGRESDB_USER=postgres
   DB_POSTGRESDB_PASSWORD=<password>
   ```
3. Verify the database exists:
   ```bash
   psql -h <db-host> -U postgres -c "SELECT datname FROM pg_database WHERE datname='n8n';"
   ```
4. If database does not exist:
   ```bash
   psql -h <db-host> -U postgres -c "CREATE DATABASE n8n;"
   ```

---

## Scenario 6: SSL Certificate Error Connecting to Internal API

**Symptom**: HTTP Request node fails with `UNABLE_TO_VERIFY_LEAF_SIGNATURE` when calling an internal API with a self-signed certificate.

**Root Cause**: Node.js does not trust the self-signed certificate.

**Fix (Production — add CA certificate)**:
1. Export the CA certificate as PEM file
2. Mount the certificate into the Docker container
3. Set the environment variable:
   ```env
   NODE_EXTRA_CA_CERTS=/certs/internal-ca.pem
   ```
4. In docker-compose:
   ```yaml
   volumes:
     - ./certs:/certs:ro
   environment:
     - NODE_EXTRA_CA_CERTS=/certs/internal-ca.pem
   ```

**Fix (Development only)**:
```env
NODE_TLS_REJECT_UNAUTHORIZED=0
```
NEVER use `NODE_TLS_REJECT_UNAUTHORIZED=0` in production. It disables ALL TLS verification.

---

## Scenario 7: Rate Limited by External API

**Symptom**: Workflow processes 500 items, starts getting 429 errors after ~100 items.

**Root Cause**: API rate limit exceeded by processing items too quickly.

**Fix**:
1. Add a **Split In Batches** node before the API call node
2. Set batch size to stay within rate limits (e.g., 10 items per batch)
3. Add a **Wait** node between batches (e.g., 1 second)
4. Enable **Retry On Fail** on the API call node:
   - Max Tries: `3`
   - Wait Between Tries: `5000` (ms)

**Workflow structure**:
```
Trigger → Split In Batches → HTTP Request (retryOnFail=true) → Wait (1s) → [loop back to Split]
```

---

## Scenario 8: SQLite Database Locking

**Symptom**: Workflow executions randomly fail with `SQLITE_BUSY` error.

**Root Cause**: Multiple concurrent processes trying to write to SQLite.

**Fix**:
1. ALWAYS switch to PostgreSQL for production:
   ```env
   DB_TYPE=postgresdb
   DB_POSTGRESDB_HOST=postgres
   DB_POSTGRESDB_DATABASE=n8n
   DB_POSTGRESDB_USER=postgres
   DB_POSTGRESDB_PASSWORD=<password>
   ```
2. If staying on SQLite for development, enable WAL mode:
   ```env
   DB_SQLITE_POOL_SIZE=4
   ```

NEVER use SQLite with queue mode. NEVER use SQLite when running concurrent workflows in production.

---

## Scenario 9: Credentials Broken After Migration

**Symptom**: After migrating n8n to a new server, all credentials show as invalid or fail with decryption errors.

**Root Cause**: `N8N_ENCRYPTION_KEY` was not migrated.

**Fix**:
1. Find the encryption key on the old server:
   - Check environment variable: `N8N_ENCRYPTION_KEY`
   - Or check file: `/home/node/.n8n/config` (inside Docker volume)
2. Set the SAME key on the new server:
   ```env
   N8N_ENCRYPTION_KEY=<key-from-old-server>
   ```
3. Restart n8n

**Prevention**: ALWAYS back up the `N8N_ENCRYPTION_KEY`. Without it, ALL credentials are permanently lost.

---

## Scenario 10: Webhook Behind Load Balancer Returns Inconsistent Results

**Symptom**: Webhook requests sometimes succeed and sometimes fail with 404, or execution results are inconsistent.

**Root Cause**: Load balancer distributing webhook requests across multiple n8n instances without session persistence.

**Fix (Queue Mode)**:
1. Ensure `WEBHOOK_URL` is set to the load balancer's public URL
2. Enable session persistence (sticky sessions) on the load balancer
3. Or: run a dedicated webhook processor:
   ```bash
   docker run --name n8n-webhook \
     -e EXECUTIONS_MODE=queue \
     -e QUEUE_BULL_REDIS_HOST=redis \
     docker.n8n.io/n8nio/n8n webhook
   ```
4. Route ALL webhook traffic to the webhook processor instance

**Fix (Multi-Main)**:
- Set `N8N_MULTI_MAIN_SETUP_ENABLED=true` on all main instances
- ALWAYS use a load balancer with session persistence
