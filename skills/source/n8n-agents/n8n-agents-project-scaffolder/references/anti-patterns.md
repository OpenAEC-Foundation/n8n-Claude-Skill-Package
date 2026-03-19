# Scaffolding Anti-Patterns & Common Mistakes

## 1. Deployment Anti-Patterns

### AP-1: Missing Encryption Key

**Wrong:**
```yaml
# docker-compose.yml — no encryption key specified
n8n:
  image: docker.n8n.io/n8nio/n8n
  environment:
    - DB_TYPE=postgresdb
    # N8N_ENCRYPTION_KEY not set — n8n auto-generates one
```

**Why it breaks:** n8n auto-generates a random encryption key on first start and stores it in `/home/node/.n8n/config`. If the container is recreated or the volume is lost, ALL encrypted credentials become permanently inaccessible.

**Correct:**
```yaml
n8n:
  image: docker.n8n.io/n8nio/n8n
  environment:
    - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
```
ALWAYS generate explicitly with `openssl rand -hex 32` and store in `.env`. ALWAYS back up this key separately.

---

### AP-2: Using SQLite in Production

**Wrong:**
```yaml
n8n:
  image: docker.n8n.io/n8nio/n8n
  # No DB_TYPE set — defaults to SQLite
  volumes:
    - n8n_data:/home/node/.n8n
```

**Why it breaks:**
- SQLite does NOT support queue mode (no horizontal scaling)
- SQLite has limited concurrent access (single writer)
- SQLite cannot handle multiple instances
- SQLite backup requires stopping n8n (or risking corruption)

**Correct:** ALWAYS use PostgreSQL for production. Set `DB_TYPE=postgresdb` with a dedicated PostgreSQL service.

---

### AP-3: Exposing Database Ports

**Wrong:**
```yaml
postgres:
  image: postgres:16-alpine
  ports:
    - "5432:5432"  # Exposed to host network!
```

**Why it breaks:** Exposes the database to the internet (or at minimum the host network). Attackers can brute-force the password or exploit PostgreSQL vulnerabilities.

**Correct:** NEVER add `ports` to the PostgreSQL service. n8n connects via Docker's internal network using the service name `postgres`. Only expose ports that MUST be public (Traefik 80/443, n8n 5678 if no reverse proxy).

---

### AP-4: No Health Checks

**Wrong:**
```yaml
n8n:
  depends_on:
    - postgres  # Only waits for container start, NOT readiness
```

**Why it breaks:** Docker `depends_on` without a condition only waits for the container to start, not for the service inside to be ready. n8n may crash because PostgreSQL is not yet accepting connections.

**Correct:**
```yaml
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 10s
    timeout: 5s
    retries: 5

n8n:
  depends_on:
    postgres:
      condition: service_healthy
```

---

### AP-5: Missing Volume Mounts

**Wrong:**
```yaml
n8n:
  image: docker.n8n.io/n8nio/n8n
  # No volumes — data stored in ephemeral container filesystem
```

**Why it breaks:** All n8n data (settings, auto-generated encryption key, SQLite database) is lost when the container is removed or recreated.

**Correct:** ALWAYS mount `n8n_data:/home/node/.n8n` as a named volume.

---

### AP-6: No restart: always

**Wrong:**
```yaml
n8n:
  image: docker.n8n.io/n8nio/n8n
  # No restart policy — container stays down after crash or reboot
```

**Why it breaks:** If n8n crashes or the server reboots, the container does not automatically restart. Active workflows stop triggering, webhooks return errors.

**Correct:** ALWAYS set `restart: always` on all services in production.

---

## 2. Queue Mode Anti-Patterns

### AP-7: Mismatched Encryption Keys

**Wrong:**
```yaml
n8n:
  environment:
    - N8N_ENCRYPTION_KEY=key-for-main-instance

n8n-worker:
  environment:
    - N8N_ENCRYPTION_KEY=different-key-for-worker  # DIFFERENT!
```

**Why it breaks:** Workers cannot decrypt credentials stored by the main instance. All workflow executions that require credentials fail silently or with cryptic errors.

**Correct:** ALWAYS use the exact same `N8N_ENCRYPTION_KEY` on ALL instances (main, workers, webhook processors). Reference the same `.env` variable.

---

### AP-8: Mismatched n8n Versions

**Wrong:**
```yaml
n8n:
  image: docker.n8n.io/n8nio/n8n:1.80.0

n8n-worker:
  image: docker.n8n.io/n8nio/n8n:1.75.0  # DIFFERENT VERSION!
```

**Why it breaks:** Different n8n versions may have incompatible database schemas, workflow execution logic, or queue message formats. This causes unpredictable execution failures.

**Correct:** ALWAYS use the exact same image tag for all n8n services. Pin a specific version or use a shared variable.

---

### AP-9: Filesystem Binary Data in Queue Mode

**Wrong:**
```yaml
n8n:
  environment:
    - N8N_DEFAULT_BINARY_DATA_MODE=filesystem  # Local filesystem
```

**Why it breaks:** In queue mode, the main instance and workers run in separate containers (potentially on separate hosts). Filesystem binary data stored by the main instance is NOT accessible to workers.

**Correct:** ALWAYS use `N8N_DEFAULT_BINARY_DATA_MODE=s3` with S3-compatible storage when running in queue mode. All instances MUST share the same S3 configuration.

---

### AP-10: Missing EXECUTIONS_MODE on Workers

**Wrong:**
```yaml
n8n-worker:
  image: docker.n8n.io/n8nio/n8n
  command: worker
  environment:
    # EXECUTIONS_MODE not set — defaults to 'regular'
    - QUEUE_BULL_REDIS_HOST=redis
```

**Why it breaks:** Without `EXECUTIONS_MODE=queue`, the worker does not connect to the Redis queue and cannot pick up jobs.

**Correct:** ALWAYS set `EXECUTIONS_MODE=queue` on ALL n8n services (main and workers) in a queue mode deployment.

---

## 3. Custom Node Anti-Patterns

### AP-11: Wrong paths in n8n Block

**Wrong:**
```json
{
  "n8n": {
    "nodes": ["nodes/MyNode/MyNode.node.ts"],
    "credentials": ["credentials/MyApi.credentials.ts"]
  }
}
```

**Why it breaks:** n8n loads compiled JavaScript, not TypeScript source files. The paths MUST point to the `dist/` directory and use `.js` extensions.

**Correct:**
```json
{
  "n8n": {
    "nodes": ["dist/nodes/MyNode/MyNode.node.js"],
    "credentials": ["dist/credentials/MyApi.credentials.js"]
  }
}
```

---

### AP-12: n8n-workflow as Direct Dependency

**Wrong:**
```json
{
  "dependencies": {
    "n8n-workflow": "^1.0.0"
  }
}
```

**Why it breaks:** Bundling `n8n-workflow` as a direct dependency causes version conflicts with n8n's own copy. Two different versions of the same module can lead to `instanceof` checks failing and type mismatches.

**Correct:**
```json
{
  "peerDependencies": {
    "n8n-workflow": "*"
  }
}
```
ALWAYS use `n8n-workflow` as a peer dependency.

---

### AP-13: Missing Community Node Keyword

**Wrong:**
```json
{
  "name": "n8n-nodes-acme",
  "keywords": ["n8n", "automation"]
}
```

**Why it breaks:** n8n's community node installer searches npm for packages with the keyword `n8n-community-node-package`. Without it, the node is not discoverable in n8n's community node browser.

**Correct:**
```json
{
  "keywords": ["n8n-community-node-package"]
}
```

---

### AP-14: Mutating Input Data

**Wrong:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  items[0].json.newField = 'value';  // MUTATING input data directly!
  return [items];
}
```

**Why it breaks:** Mutating the original input data can cause unpredictable behavior in other nodes that reference the same data, especially when the workflow has branching paths.

**Correct:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];
  for (let i = 0; i < items.length; i++) {
    returnData.push({
      json: { ...items[i].json, newField: 'value' },
      pairedItem: { item: i },
    });
  }
  return [returnData];
}
```
ALWAYS create new objects. NEVER modify `getInputData()` results directly.

---

### AP-15: Missing continueOnFail Handling

**Wrong:**
```typescript
for (let i = 0; i < items.length; i++) {
  const response = await this.helpers.httpRequest({ /* ... */ });
  returnData.push({ json: response as IDataObject });
  // No error handling — one failing item kills the entire batch
}
```

**Why it breaks:** If any single item fails (e.g., API returns 404 for one ID), the entire node execution fails and all items are lost.

**Correct:**
```typescript
for (let i = 0; i < items.length; i++) {
  try {
    const response = await this.helpers.httpRequest({ /* ... */ });
    returnData.push({ json: response as IDataObject });
  } catch (error) {
    if (this.continueOnFail()) {
      returnData.push({
        json: { error: (error as Error).message },
        pairedItem: { item: i },
      });
      continue;
    }
    throw error;
  }
}
```

---

## 4. Workflow Template Anti-Patterns

### AP-16: Missing executionOrder Setting

**Wrong:**
```json
{
  "settings": {}
}
```

**Why it breaks:** Without `"executionOrder": "v1"`, n8n may use the legacy v0 execution order algorithm, which has different node execution semantics and can produce unexpected results.

**Correct:**
```json
{
  "settings": {
    "executionOrder": "v1"
  }
}
```
ALWAYS set `"executionOrder": "v1"` in workflow template settings.

---

### AP-17: Duplicate Node Names

**Wrong:**
```json
{
  "nodes": [
    { "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest" },
    { "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest" }
  ]
}
```

**Why it breaks:** Node names MUST be unique within a workflow. Connections reference nodes by name. Duplicate names cause ambiguous references and broken connections.

**Correct:** ALWAYS use unique, descriptive node names: "Fetch Users", "Update Record", "Send Notification".

---

### AP-18: No Error Workflow Reference

**Wrong:** Workflow templates that omit error handling entirely.

**Why it breaks:** When a production workflow fails, nobody is notified. Failed executions accumulate silently. Issues are discovered hours or days later.

**Correct:** ALWAYS include either:
1. An `errorWorkflow` reference in workflow settings, OR
2. `continueOnFail` with explicit error-path handling on critical nodes

---

## 5. Backup Anti-Patterns

### AP-19: Decrypted Credential Export in Scripts

**Wrong:**
```bash
n8n export:credentials --all --decrypted --output=/backups/credentials.json
```

**Why it breaks:** Exports credentials in plain text. If the backup is compromised, all API keys, passwords, and secrets are exposed.

**Correct:** NEVER use `--decrypted` in automated backup scripts. Export credentials in encrypted form and ensure the `N8N_ENCRYPTION_KEY` is backed up separately through a secure channel.

---

### AP-20: Backup Without Encryption Key

**Wrong:** Backing up the database and workflow exports but not recording the `N8N_ENCRYPTION_KEY`.

**Why it breaks:** Encrypted credentials in the backup CANNOT be restored without the exact encryption key that was used when they were created. The backup is partially useless.

**Correct:** ALWAYS verify and separately store the `N8N_ENCRYPTION_KEY` as part of the backup procedure. Store it in a secrets manager or secure vault, NOT alongside the backup files.
