# Scaffolding Templates & Methods

## Deployment Docker Compose Template

### docker-compose.yml (PostgreSQL + Traefik)

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
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
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
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

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
      - TZ=${TZ}
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Security
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_BLOCK_ENV_ACCESS_IN_NODE=true
      # Execution
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=${EXECUTIONS_DATA_MAX_AGE:-336}
      - EXECUTIONS_DATA_PRUNE_MAX_COUNT=${EXECUTIONS_DATA_PRUNE_MAX_COUNT:-10000}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)"
      - "traefik.http.routers.n8n.tls=true"
      - "traefik.http.routers.n8n.tls.certresolver=mytlschallenge"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"

volumes:
  traefik_data:
  postgres_data:
  n8n_data:
```

### .env Template

```env
# =============================================================================
# n8n Deployment Configuration
# =============================================================================
# INSTRUCTIONS:
# 1. Copy this file to .env
# 2. Replace ALL placeholder values
# 3. Generate encryption key: openssl rand -hex 32
# 4. NEVER commit .env to version control
# =============================================================================

# === DOMAIN ===
DOMAIN_NAME=example.com
SUBDOMAIN=n8n
SSL_EMAIL=admin@example.com

# === TIMEZONE ===
GENERIC_TIMEZONE=Europe/Amsterdam
TZ=Europe/Amsterdam

# === DATABASE ===
POSTGRES_USER=n8n
POSTGRES_PASSWORD=CHANGE_ME_USE_openssl_rand_base64_32
POSTGRES_DB=n8n

# === SECURITY (CRITICAL - BACK THIS UP!) ===
# Generate with: openssl rand -hex 32
# Losing this key = losing access to ALL encrypted credentials
N8N_ENCRYPTION_KEY=CHANGE_ME_GENERATE_WITH_openssl_rand_hex_32

# === EXECUTION DATA ===
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336
EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
```

---

## Queue Mode Docker Compose Template

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
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
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
      # Database
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Queue Mode
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      # Security
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      # Binary Data (S3 required for queue mode)
      - N8N_DEFAULT_BINARY_DATA_MODE=s3
      - N8N_EXTERNAL_STORAGE_S3_HOST=${S3_HOST}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=${S3_BUCKET_NAME}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION=${S3_BUCKET_REGION}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY=${S3_ACCESS_KEY}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET=${S3_ACCESS_SECRET}
      # Execution
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=${EXECUTIONS_DATA_MAX_AGE:-336}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files

  n8n-worker:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    command: worker --concurrency=5
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - NODE_ENV=production
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}
      # Database (MUST match main instance)
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      # Queue Mode (MUST match main instance)
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      # Security (MUST match main instance)
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_RUNNERS_ENABLED=true
      # Binary Data (MUST match main instance)
      - N8N_DEFAULT_BINARY_DATA_MODE=s3
      - N8N_EXTERNAL_STORAGE_S3_HOST=${S3_HOST}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=${S3_BUCKET_NAME}
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION=${S3_BUCKET_REGION}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY=${S3_ACCESS_KEY}
      - N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET=${S3_ACCESS_SECRET}
    volumes:
      - n8n_worker_data:/home/node/.n8n

volumes:
  postgres_data:
  redis_data:
  n8n_data:
  n8n_worker_data:
```

### Queue Mode .env Additions

```env
# === QUEUE MODE (add to base .env) ===

# S3 Binary Storage (REQUIRED for queue mode)
S3_HOST=https://s3.amazonaws.com
S3_BUCKET_NAME=n8n-binary-data
S3_BUCKET_REGION=eu-west-1
S3_ACCESS_KEY=CHANGE_ME
S3_ACCESS_SECRET=CHANGE_ME

# Worker Scaling
# To scale workers: docker compose up -d --scale n8n-worker=3
```

---

## Custom Node Templates

### package.json

```json
{
  "name": "n8n-nodes-example",
  "version": "0.1.0",
  "description": "n8n community node for Example Service",
  "license": "MIT",
  "homepage": "https://github.com/your-org/n8n-nodes-example",
  "repository": {
    "type": "git",
    "url": "https://github.com/your-org/n8n-nodes-example.git"
  },
  "keywords": ["n8n-community-node-package"],
  "n8n": {
    "n8nNodesApiVersion": 1,
    "strict": true,
    "credentials": [
      "dist/credentials/ExampleApi.credentials.js"
    ],
    "nodes": [
      "dist/nodes/ExampleNode/ExampleNode.node.js"
    ]
  },
  "main": "index.js",
  "scripts": {
    "build": "n8n-node build",
    "dev": "n8n-node dev",
    "lint": "n8n-node lint",
    "lint:fix": "n8n-node lint --fix",
    "release": "n8n-node release",
    "prepublishOnly": "n8n-node prerelease"
  },
  "files": ["dist"],
  "devDependencies": {
    "@n8n/node-cli": "*",
    "typescript": "5.9.2"
  },
  "peerDependencies": {
    "n8n-workflow": "*"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "strict": true,
    "module": "commonjs",
    "target": "es2022",
    "lib": ["es2022"],
    "moduleResolution": "node",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "."
  },
  "include": [
    "nodes/**/*.ts",
    "credentials/**/*.ts"
  ],
  "exclude": [
    "node_modules",
    "dist"
  ]
}
```

### Credential Template (credentials/ExampleApi.credentials.ts)

```typescript
import type {
  IAuthenticate,
  ICredentialTestRequest,
  ICredentialType,
  INodeProperties,
} from 'n8n-workflow';

export class ExampleApi implements ICredentialType {
  name = 'exampleApi';
  displayName = 'Example API';
  documentationUrl = 'https://docs.example.com/api';

  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
      required: true,
      description: 'API key from your Example account settings',
    },
    {
      displayName: 'Base URL',
      name: 'baseUrl',
      type: 'string',
      default: 'https://api.example.com',
      description: 'Base URL for the Example API',
    },
  ];

  authenticate: IAuthenticate = {
    type: 'generic',
    properties: {
      headers: {
        Authorization: '=Bearer {{$credentials.apiKey}}',
      },
    },
  };

  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials.baseUrl}}',
      url: '/v1/me',
    },
  };
}
```

### Node Template (nodes/ExampleNode/ExampleNode.node.ts)

```typescript
import type {
  IDataObject,
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class ExampleNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Example Node',
    name: 'exampleNode',
    icon: 'file:../../icons/example.svg',
    group: ['transform'],
    version: 1,
    subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
    description: 'Interact with the Example API',
    defaults: {
      name: 'Example Node',
    },
    inputs: [NodeConnectionTypes.Main],
    outputs: [NodeConnectionTypes.Main],
    credentials: [
      {
        name: 'exampleApi',
        required: true,
      },
    ],
    properties: [
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        noDataExpression: true,
        options: [
          { name: 'Item', value: 'item' },
        ],
        default: 'item',
      },
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: {
          show: { resource: ['item'] },
        },
        options: [
          {
            name: 'Get',
            value: 'get',
            action: 'Get an item',
            description: 'Retrieve a single item by ID',
          },
          {
            name: 'Get Many',
            value: 'getAll',
            action: 'Get many items',
            description: 'Retrieve multiple items',
          },
        ],
        default: 'get',
      },
      {
        displayName: 'Item ID',
        name: 'itemId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: {
          show: { resource: ['item'], operation: ['get'] },
        },
        description: 'The unique identifier of the item',
      },
      {
        displayName: 'Limit',
        name: 'limit',
        type: 'number',
        default: 50,
        typeOptions: { minValue: 1, maxValue: 100 },
        displayOptions: {
          show: { resource: ['item'], operation: ['getAll'] },
        },
        description: 'Max number of results to return',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    const resource = this.getNodeParameter('resource', 0) as string;
    const operation = this.getNodeParameter('operation', 0) as string;
    const credentials = await this.getCredentials('exampleApi');

    for (let i = 0; i < items.length; i++) {
      try {
        if (resource === 'item') {
          if (operation === 'get') {
            const itemId = this.getNodeParameter('itemId', i) as string;
            const response = await this.helpers.httpRequest({
              method: 'GET',
              url: `${credentials.baseUrl}/v1/items/${itemId}`,
              headers: {
                Authorization: `Bearer ${credentials.apiKey}`,
              },
            });
            returnData.push({ json: response as IDataObject });
          }
          if (operation === 'getAll') {
            const limit = this.getNodeParameter('limit', i) as number;
            const response = await this.helpers.httpRequest({
              method: 'GET',
              url: `${credentials.baseUrl}/v1/items`,
              qs: { limit },
              headers: {
                Authorization: `Bearer ${credentials.apiKey}`,
              },
            });
            const responseItems = (response as IDataObject).items as IDataObject[];
            for (const item of responseItems) {
              returnData.push({ json: item });
            }
          }
        }
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

    return [returnData];
  }
}
```

### Codex Metadata (nodes/ExampleNode/ExampleNode.node.json)

```json
{
  "node": "n8n-nodes-example.exampleNode",
  "nodeVersion": "1.0",
  "codexVersion": "1.0",
  "categories": ["Miscellaneous"],
  "resources": {
    "primaryDocumentation": [
      {
        "url": "https://docs.example.com/"
      }
    ]
  }
}
```

---

## Backup Script Template

### backup.sh

```bash
#!/bin/bash
# =============================================================================
# n8n Backup Script
# Usage: ./backup.sh
# Schedule: crontab -e -> 0 2 * * * /path/to/backup.sh
# =============================================================================

set -euo pipefail

# Configuration
BACKUP_DIR="./backups"
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="${BACKUP_DIR}/${TIMESTAMP}"
COMPOSE_FILE="./docker-compose.yml"

# Create backup directory
mkdir -p "${BACKUP_PATH}"

echo "=== n8n Backup Started: ${TIMESTAMP} ==="

# 1. PostgreSQL database dump
echo "[1/4] Dumping PostgreSQL database..."
docker compose -f "${COMPOSE_FILE}" exec -T postgres \
  pg_dump -U "${POSTGRES_USER:-n8n}" "${POSTGRES_DB:-n8n}" \
  > "${BACKUP_PATH}/database.sql"

# 2. Export workflows
echo "[2/4] Exporting workflows..."
docker compose -f "${COMPOSE_FILE}" exec -T n8n \
  n8n export:workflow --backup --output="/home/node/.n8n/backups/" 2>/dev/null || true
docker compose -f "${COMPOSE_FILE}" cp \
  n8n:/home/node/.n8n/backups/ "${BACKUP_PATH}/workflows/" 2>/dev/null || true

# 3. Export credentials (encrypted)
echo "[3/4] Exporting credentials (encrypted)..."
docker compose -f "${COMPOSE_FILE}" exec -T n8n \
  n8n export:credentials --backup --output="/home/node/.n8n/backups-creds/" 2>/dev/null || true
docker compose -f "${COMPOSE_FILE}" cp \
  n8n:/home/node/.n8n/backups-creds/ "${BACKUP_PATH}/credentials/" 2>/dev/null || true

# 4. Verify encryption key is documented
echo "[4/4] Verifying encryption key documentation..."
if [ -z "${N8N_ENCRYPTION_KEY:-}" ]; then
  echo "WARNING: N8N_ENCRYPTION_KEY is not set in environment."
  echo "WARNING: Without this key, credential backups CANNOT be restored."
  echo "WARNING: Check your .env file and back up the encryption key separately."
fi

# Cleanup old backups
echo "Cleaning up backups older than ${RETENTION_DAYS} days..."
find "${BACKUP_DIR}" -maxdepth 1 -type d -mtime +${RETENTION_DAYS} -exec rm -rf {} \; 2>/dev/null || true

# Summary
BACKUP_SIZE=$(du -sh "${BACKUP_PATH}" | cut -f1)
echo "=== Backup Complete ==="
echo "Location: ${BACKUP_PATH}"
echo "Size: ${BACKUP_SIZE}"
echo "Retention: ${RETENTION_DAYS} days"
echo ""
echo "REMINDER: Ensure N8N_ENCRYPTION_KEY is backed up separately!"
echo "Without it, encrypted credentials CANNOT be restored."
```

### restore.sh

```bash
#!/bin/bash
# =============================================================================
# n8n Restore Script
# Usage: ./restore.sh <backup_directory>
# Example: ./restore.sh ./backups/20260319_020000
# =============================================================================

set -euo pipefail

BACKUP_PATH="${1:?Usage: ./restore.sh <backup_directory>}"
COMPOSE_FILE="./docker-compose.yml"

if [ ! -d "${BACKUP_PATH}" ]; then
  echo "ERROR: Backup directory not found: ${BACKUP_PATH}"
  exit 1
fi

echo "=== n8n Restore from ${BACKUP_PATH} ==="
echo "WARNING: This will overwrite the current database."
read -p "Continue? (y/N): " confirm
if [ "${confirm}" != "y" ]; then
  echo "Restore cancelled."
  exit 0
fi

# 1. Restore PostgreSQL database
if [ -f "${BACKUP_PATH}/database.sql" ]; then
  echo "[1/3] Restoring PostgreSQL database..."
  docker compose -f "${COMPOSE_FILE}" exec -T postgres \
    psql -U "${POSTGRES_USER:-n8n}" "${POSTGRES_DB:-n8n}" \
    < "${BACKUP_PATH}/database.sql"
  echo "Database restored."
else
  echo "[1/3] SKIP: No database.sql found."
fi

# 2. Import workflows
if [ -d "${BACKUP_PATH}/workflows" ]; then
  echo "[2/3] Importing workflows..."
  docker compose -f "${COMPOSE_FILE}" cp \
    "${BACKUP_PATH}/workflows/" n8n:/home/node/.n8n/restore-workflows/
  docker compose -f "${COMPOSE_FILE}" exec -T n8n \
    n8n import:workflow --separate --input="/home/node/.n8n/restore-workflows/"
  echo "Workflows imported."
else
  echo "[2/3] SKIP: No workflows directory found."
fi

# 3. Import credentials
if [ -d "${BACKUP_PATH}/credentials" ]; then
  echo "[3/3] Importing credentials..."
  echo "NOTE: N8N_ENCRYPTION_KEY must match the key used during backup."
  docker compose -f "${COMPOSE_FILE}" cp \
    "${BACKUP_PATH}/credentials/" n8n:/home/node/.n8n/restore-credentials/
  docker compose -f "${COMPOSE_FILE}" exec -T n8n \
    n8n import:credentials --separate --input="/home/node/.n8n/restore-credentials/"
  echo "Credentials imported."
else
  echo "[3/3] SKIP: No credentials directory found."
fi

echo "=== Restore Complete ==="
echo "Restart n8n to apply changes: docker compose restart n8n"
```
