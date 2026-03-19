# n8n REST API -- Examples

All examples use curl. Replace `<BASE_URL>` with your n8n instance URL (e.g., `https://n8n.example.com`) and `<API_KEY>` with your actual API key.

---

## Authentication

### Basic Authenticated Request

```bash
curl -X GET \
  '<BASE_URL>/api/v1/workflows' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

ALWAYS include both the `accept` and `X-N8N-API-KEY` headers in every request.

---

## Workflows

### List All Active Workflows

```bash
curl -X GET \
  '<BASE_URL>/api/v1/workflows?active=true&limit=100' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### List Workflows by Tag

```bash
curl -X GET \
  '<BASE_URL>/api/v1/workflows?tags=tag-id-1,tag-id-2' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Get Single Workflow (Excluding Pinned Data)

```bash
curl -X GET \
  '<BASE_URL>/api/v1/workflows/1234?excludePinnedData=true' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Create a Workflow

```bash
curl -X POST \
  '<BASE_URL>/api/v1/workflows' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{
    "name": "My API Workflow",
    "nodes": [
      {
        "parameters": {},
        "name": "Start",
        "type": "n8n-nodes-base.manualTrigger",
        "typeVersion": 1,
        "position": [250, 300]
      }
    ],
    "connections": {},
    "settings": {
      "executionOrder": "v1"
    }
  }'
```

### Update a Workflow

```bash
curl -X PUT \
  '<BASE_URL>/api/v1/workflows/1234' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{
    "name": "Updated Workflow Name",
    "nodes": [ /* updated nodes */ ],
    "connections": { /* updated connections */ },
    "settings": { "executionOrder": "v1" }
  }'
```

### Activate a Workflow

```bash
curl -X POST \
  '<BASE_URL>/api/v1/workflows/1234/activate' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Deactivate a Workflow

```bash
curl -X POST \
  '<BASE_URL>/api/v1/workflows/1234/deactivate' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Delete a Workflow

```bash
curl -X DELETE \
  '<BASE_URL>/api/v1/workflows/1234' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Transfer Workflow to Another Project

```bash
curl -X POST \
  '<BASE_URL>/api/v1/workflows/1234/transfer' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{ "destinationProjectId": "project-abc-123" }'
```

### Update Workflow Tags

```bash
curl -X PUT \
  '<BASE_URL>/api/v1/workflows/1234/tags' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '[{ "id": "tag-id-1" }, { "id": "tag-id-2" }]'
```

---

## Executions

### List Failed Executions for a Workflow

```bash
curl -X GET \
  '<BASE_URL>/api/v1/executions?workflowId=1234&status=error&limit=50' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Get Execution with Full Data

```bash
curl -X GET \
  '<BASE_URL>/api/v1/executions/5678?includeData=true' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Retry a Failed Execution

```bash
curl -X POST \
  '<BASE_URL>/api/v1/executions/5678/retry' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Stop a Running Execution

```bash
curl -X POST \
  '<BASE_URL>/api/v1/executions/5678/stop' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Bulk Stop Executions

```bash
curl -X POST \
  '<BASE_URL>/api/v1/executions/stop' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{
    "status": ["running", "waiting"],
    "workflowId": "1234"
  }'
```

### Delete an Execution

```bash
curl -X DELETE \
  '<BASE_URL>/api/v1/executions/5678' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

---

## Credentials

### List All Credentials

```bash
curl -X GET \
  '<BASE_URL>/api/v1/credentials?limit=100' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Get Credential Type Schema

ALWAYS fetch the schema before creating credentials to know the required fields:

```bash
curl -X GET \
  '<BASE_URL>/api/v1/credential-types/httpBasicAuth' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### Create a Credential

```bash
curl -X POST \
  '<BASE_URL>/api/v1/credentials' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{
    "name": "My HTTP Basic Auth",
    "type": "httpBasicAuth",
    "data": {
      "user": "admin",
      "password": "secret123"
    }
  }'
```

### Transfer Credential to Another Project

```bash
curl -X POST \
  '<BASE_URL>/api/v1/credentials/cred-id-123/transfer' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{ "destinationProjectId": "project-abc-123" }'
```

---

## Tags

### Create a Tag

```bash
curl -X POST \
  '<BASE_URL>/api/v1/tags' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{ "name": "production" }'
```

### List All Tags

```bash
curl -X GET \
  '<BASE_URL>/api/v1/tags' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

---

## Variables

### Create a Variable

```bash
curl -X POST \
  '<BASE_URL>/api/v1/variables' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>' \
  -d '{ "key": "API_BASE_URL", "value": "https://api.example.com" }'
```

### List All Variables

```bash
curl -X GET \
  '<BASE_URL>/api/v1/variables' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

---

## Projects

### List All Projects

```bash
curl -X GET \
  '<BASE_URL>/api/v1/projects' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

### List Users in a Project

```bash
curl -X GET \
  '<BASE_URL>/api/v1/projects/project-abc-123/users' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

---

## Pagination

### Full Pagination Loop (Bash)

This pattern fetches ALL results across multiple pages:

```bash
#!/bin/bash
BASE_URL="https://n8n.example.com"
API_KEY="your-api-key"
cursor=""
page=1

while true; do
  echo "Fetching page $page..."

  if [ -z "$cursor" ]; then
    response=$(curl -s "${BASE_URL}/api/v1/workflows?limit=250&active=true" \
      -H 'accept: application/json' \
      -H "X-N8N-API-KEY: ${API_KEY}")
  else
    response=$(curl -s "${BASE_URL}/api/v1/workflows?limit=250&active=true&cursor=${cursor}" \
      -H 'accept: application/json' \
      -H "X-N8N-API-KEY: ${API_KEY}")
  fi

  # Process the data array
  echo "$response" | jq '.data[] | .name'

  # Get next cursor
  cursor=$(echo "$response" | jq -r '.nextCursor // empty')

  # Break if no more pages
  if [ -z "$cursor" ]; then
    echo "All pages fetched."
    break
  fi

  page=$((page + 1))
done
```

### Pagination in Python

```python
import requests

BASE_URL = "https://n8n.example.com"
API_KEY = "your-api-key"
headers = {
    "accept": "application/json",
    "X-N8N-API-KEY": API_KEY,
}

all_workflows = []
cursor = None

while True:
    params = {"limit": 250, "active": "true"}
    if cursor:
        params["cursor"] = cursor

    response = requests.get(
        f"{BASE_URL}/api/v1/workflows",
        headers=headers,
        params=params,
    )
    response.raise_for_status()
    result = response.json()

    all_workflows.extend(result["data"])

    cursor = result.get("nextCursor")
    if not cursor:
        break

print(f"Total workflows: {len(all_workflows)}")
```

### Pagination in JavaScript/TypeScript

```typescript
const BASE_URL = "https://n8n.example.com";
const API_KEY = "your-api-key";

async function fetchAllWorkflows(): Promise<any[]> {
  const allWorkflows: any[] = [];
  let cursor: string | null = null;

  while (true) {
    const params = new URLSearchParams({ limit: "250", active: "true" });
    if (cursor) params.set("cursor", cursor);

    const response = await fetch(
      `${BASE_URL}/api/v1/workflows?${params}`,
      {
        headers: {
          accept: "application/json",
          "X-N8N-API-KEY": API_KEY,
        },
      }
    );

    if (!response.ok) throw new Error(`API error: ${response.status}`);
    const result = await response.json();

    allWorkflows.push(...result.data);

    cursor = result.nextCursor ?? null;
    if (!cursor) break;
  }

  return allWorkflows;
}
```

---

## Security Audit

### Run a Security Audit

```bash
curl -X POST \
  '<BASE_URL>/api/v1/audit' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <API_KEY>'
```

This analyzes the n8n instance for security issues including abandoned workflows, insecure configurations, and credential exposure risks.
