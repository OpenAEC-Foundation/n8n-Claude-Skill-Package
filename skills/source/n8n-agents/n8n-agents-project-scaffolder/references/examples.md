# Complete Project Scaffold Examples

## Example 1: Production Deployment Scaffold

### Scenario
A team needs a production n8n instance with PostgreSQL, HTTPS via Traefik, automated backups, and proper security defaults.

### Generated Files

**Directory structure:**
```
n8n-production/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── backup.sh
├── restore.sh
├── local-files/
│   └── .gitkeep
└── backups/
    └── .gitkeep
```

**.gitignore:**
```gitignore
.env
backups/
local-files/*
!local-files/.gitkeep
```

**Initialization commands:**
```bash
# 1. Clone or create project directory
mkdir n8n-production && cd n8n-production

# 2. Copy .env.example to .env and configure
cp .env.example .env

# 3. Generate encryption key
echo "N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)" >> .env

# 4. Generate database password
echo "POSTGRES_PASSWORD=$(openssl rand -base64 32)" >> .env

# 5. Create required directories
mkdir -p local-files backups

# 6. Make scripts executable
chmod +x backup.sh restore.sh

# 7. Start services
docker compose up -d

# 8. Verify health
docker compose ps
docker compose logs n8n --tail=50

# 9. Schedule daily backups (2 AM)
(crontab -l 2>/dev/null; echo "0 2 * * * cd /path/to/n8n-production && ./backup.sh >> backups/cron.log 2>&1") | crontab -
```

---

## Example 2: Custom Node Package Scaffold

### Scenario
A developer needs to create a community node package for a custom API service called "Acme API".

### Generated Files

**Directory structure:**
```
n8n-nodes-acme/
├── package.json
├── tsconfig.json
├── LICENSE
├── README.md
├── .gitignore
├── credentials/
│   └── AcmeApi.credentials.ts
├── nodes/
│   └── Acme/
│       ├── Acme.node.ts
│       └── Acme.node.json
└── icons/
    └── acme.svg
```

**.gitignore:**
```gitignore
node_modules/
dist/
*.js
*.d.ts
*.js.map
!jest.config.js
```

**README.md:**
```markdown
# n8n-nodes-acme

This is an n8n community node for interacting with the Acme API.

## Installation

### Community Nodes (Recommended)

1. Go to **Settings > Community Nodes**
2. Select **Install**
3. Enter `n8n-nodes-acme`
4. Agree to the risks and select **Install**

### Manual Installation

```bash
cd ~/.n8n/custom
npm install n8n-nodes-acme
```

## Credentials

You need an Acme API key. Get one at [https://acme.example.com/settings/api](https://acme.example.com/settings/api).

## Resources

- [n8n Community Nodes Documentation](https://docs.n8n.io/integrations/community-nodes/)
- [Acme API Documentation](https://docs.acme.example.com/)
```

**Development commands:**
```bash
# 1. Install dependencies
npm install

# 2. Start development mode (hot-reload at http://localhost:5678)
npm run dev

# 3. Build for production
npm run build

# 4. Lint code
npm run lint

# 5. Publish to npm
npm publish
```

---

## Example 3: Workflow Template — Webhook with Response

### Scenario
A webhook that receives data, validates it, processes it, and returns a structured response.

### webhook-workflow.json

```json
{
  "name": "Webhook Data Processor",
  "nodes": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [250, 300],
      "webhookId": "unique-webhook-id",
      "parameters": {
        "path": "process-data",
        "httpMethod": "POST",
        "responseMode": "responseNode",
        "options": {}
      }
    },
    {
      "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "name": "Validate Input",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [450, 300],
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": true },
          "conditions": [
            {
              "id": "condition-1",
              "leftValue": "={{ $json.email }}",
              "rightValue": "",
              "operator": { "type": "string", "operation": "notEmpty" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "name": "Process Data",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [650, 200],
      "parameters": {
        "mode": "manual",
        "fields": {
          "values": [
            {
              "name": "status",
              "stringValue": "processed"
            },
            {
              "name": "email",
              "stringValue": "={{ $json.email }}"
            },
            {
              "name": "processedAt",
              "stringValue": "={{ $now.toISO() }}"
            }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "d4e5f6a7-b8c9-0123-defa-234567890123",
      "name": "Success Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1.1,
      "position": [850, 200],
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ { success: true, data: $json } }}",
        "options": {
          "responseCode": 200
        }
      }
    },
    {
      "id": "e5f6a7b8-c9d0-1234-efab-345678901234",
      "name": "Error Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1.1,
      "position": [650, 400],
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ { success: false, error: 'Missing required field: email' } }}",
        "options": {
          "responseCode": 400
        }
      }
    }
  ],
  "connections": {
    "Webhook": {
      "main": [[{ "node": "Validate Input", "type": "main", "index": 0 }]]
    },
    "Validate Input": {
      "main": [
        [{ "node": "Process Data", "type": "main", "index": 0 }],
        [{ "node": "Error Response", "type": "main", "index": 0 }]
      ]
    },
    "Process Data": {
      "main": [[{ "node": "Success Response", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveDataErrorExecution": "all",
    "saveDataSuccessExecution": "all"
  },
  "pinData": {},
  "staticData": null
}
```

---

## Example 4: Workflow Template — Error Handler

### error-handler-workflow.json

```json
{
  "name": "Centralized Error Handler",
  "nodes": [
    {
      "id": "f6a7b8c9-d0e1-2345-fabc-456789012345",
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [250, 300],
      "parameters": {}
    },
    {
      "id": "a7b8c9d0-e1f2-3456-abcd-567890123456",
      "name": "Format Error",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [450, 300],
      "parameters": {
        "mode": "manual",
        "fields": {
          "values": [
            {
              "name": "workflow_name",
              "stringValue": "={{ $json.workflow.name }}"
            },
            {
              "name": "workflow_id",
              "stringValue": "={{ $json.workflow.id }}"
            },
            {
              "name": "error_message",
              "stringValue": "={{ $json.execution.error.message }}"
            },
            {
              "name": "execution_id",
              "stringValue": "={{ $json.execution.id }}"
            },
            {
              "name": "execution_url",
              "stringValue": "={{ $json.execution.url }}"
            },
            {
              "name": "timestamp",
              "stringValue": "={{ $now.toISO() }}"
            }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "b8c9d0e1-f2a3-4567-bcde-678901234567",
      "name": "Send Notification",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [650, 300],
      "parameters": {
        "method": "POST",
        "url": "={{ $env.SLACK_WEBHOOK_URL }}",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "text",
              "value": "=:rotating_light: *Workflow Error*\n*Workflow:* {{ $json.workflow_name }}\n*Error:* {{ $json.error_message }}\n*Execution:* {{ $json.execution_url }}"
            }
          ]
        },
        "options": {}
      }
    }
  ],
  "connections": {
    "Error Trigger": {
      "main": [[{ "node": "Format Error", "type": "main", "index": 0 }]]
    },
    "Format Error": {
      "main": [[{ "node": "Send Notification", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveDataErrorExecution": "all",
    "saveDataSuccessExecution": "all"
  }
}
```

**Usage:** In other workflows, go to **Workflow Settings** and select this workflow as the error workflow.

---

## Example 5: Workflow Template — Scheduled Data Sync

### scheduled-sync-workflow.json

```json
{
  "name": "Scheduled Data Sync",
  "nodes": [
    {
      "id": "c9d0e1f2-a3b4-5678-cdef-789012345678",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [250, 300],
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "hours",
              "hoursInterval": 1
            }
          ]
        }
      }
    },
    {
      "id": "d0e1f2a3-b4c5-6789-defa-890123456789",
      "name": "Fetch Source Data",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [450, 300],
      "parameters": {
        "method": "GET",
        "url": "https://api.source.com/data",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {}
      },
      "credentials": {
        "httpHeaderAuth": {
          "id": "1",
          "name": "Source API Auth"
        }
      },
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 2000
    },
    {
      "id": "e1f2a3b4-c5d6-7890-efab-901234567890",
      "name": "Filter Changed",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [650, 300],
      "parameters": {
        "conditions": {
          "options": {},
          "conditions": [
            {
              "id": "condition-1",
              "leftValue": "={{ $json.updatedAt }}",
              "rightValue": "={{ $now.minus({hours: 1}).toISO() }}",
              "operator": { "type": "dateTime", "operation": "after" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "f2a3b4c5-d6e7-8901-fabc-012345678901",
      "name": "Upsert to Destination",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [850, 200],
      "parameters": {
        "method": "PUT",
        "url": "=https://api.destination.com/data/{{ $json.id }}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ JSON.stringify($json) }}",
        "options": {}
      },
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 2000
    },
    {
      "id": "a3b4c5d6-e7f8-9012-abcd-123456789012",
      "name": "No Changes",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [850, 400],
      "parameters": {}
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [[{ "node": "Fetch Source Data", "type": "main", "index": 0 }]]
    },
    "Fetch Source Data": {
      "main": [[{ "node": "Filter Changed", "type": "main", "index": 0 }]]
    },
    "Filter Changed": {
      "main": [
        [{ "node": "Upsert to Destination", "type": "main", "index": 0 }],
        [{ "node": "No Changes", "type": "main", "index": 0 }]
      ]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveDataErrorExecution": "all",
    "saveDataSuccessExecution": "none",
    "errorWorkflow": "REPLACE_WITH_ERROR_HANDLER_WORKFLOW_ID"
  }
}
```

---

## Example 6: Workflow Template — Sub-Workflow (Reusable Module)

### sub-workflow-template.json

```json
{
  "name": "Reusable: Enrich Contact Data",
  "nodes": [
    {
      "id": "b4c5d6e7-f8a9-0123-bcde-234567890123",
      "name": "Execute Workflow Trigger",
      "type": "n8n-nodes-base.executeWorkflowTrigger",
      "typeVersion": 1.1,
      "position": [250, 300],
      "parameters": {}
    },
    {
      "id": "c5d6e7f8-a9b0-1234-cdef-345678901234",
      "name": "Lookup Email Domain",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [450, 300],
      "parameters": {
        "mode": "manual",
        "fields": {
          "values": [
            {
              "name": "email",
              "stringValue": "={{ $json.email }}"
            },
            {
              "name": "domain",
              "stringValue": "={{ $json.email.extractDomain() }}"
            },
            {
              "name": "enrichedAt",
              "stringValue": "={{ $now.toISO() }}"
            }
          ]
        },
        "options": { "includeBinary": false }
      }
    }
  ],
  "connections": {
    "Execute Workflow Trigger": {
      "main": [[{ "node": "Lookup Email Domain", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "callerPolicy": "workflowsFromSameOwner"
  }
}
```

**Usage:** Call from another workflow using the "Execute Workflow" node, passing data items as input.
