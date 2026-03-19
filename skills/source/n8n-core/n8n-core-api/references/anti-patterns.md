# n8n REST API -- Anti-Patterns

Common mistakes when using the n8n REST API, with explanations and correct alternatives.

---

## AP-1: Using Offset-Based Pagination

### Wrong

```bash
# WRONG: n8n does NOT support offset-based pagination
curl '<BASE_URL>/api/v1/workflows?offset=100&limit=50' \
  -H 'X-N8N-API-KEY: <key>'
```

### Why It Fails

The n8n API uses cursor-based pagination exclusively. The `offset` parameter is NOT supported for pagination. Passing `offset` does not paginate -- it is ignored or causes unexpected results.

### Correct

```bash
# First page
curl '<BASE_URL>/api/v1/workflows?limit=50' \
  -H 'X-N8N-API-KEY: <key>'
# Response: { "data": [...], "nextCursor": "MTIzNA==" }

# Next page using cursor
curl '<BASE_URL>/api/v1/workflows?limit=50&cursor=MTIzNA==' \
  -H 'X-N8N-API-KEY: <key>'
```

ALWAYS use the `nextCursor` value from the response to fetch subsequent pages.

---

## AP-2: Hardcoding API Keys

### Wrong

```python
# WRONG: API key hardcoded in source code
headers = {"X-N8N-API-KEY": "n8n_api_abc123def456"}
```

### Why It Fails

Hardcoded API keys end up in version control, CI logs, and error reports. Anyone with repository access gains full API access to your n8n instance.

### Correct

```python
import os

# Correct: Load from environment variable
api_key = os.environ["N8N_API_KEY"]
headers = {"X-N8N-API-KEY": api_key}
```

ALWAYS store API keys in environment variables or a secrets manager. NEVER commit API keys to source control.

---

## AP-3: Deleting Running Executions

### Wrong

```bash
# WRONG: Attempting to delete a running execution
curl -X DELETE '<BASE_URL>/api/v1/executions/5678' \
  -H 'X-N8N-API-KEY: <key>'
# Returns 409 Conflict
```

### Why It Fails

The API does not allow deletion of executions that are currently running. You receive a 409 Conflict error.

### Correct

```bash
# Step 1: Stop the execution first
curl -X POST '<BASE_URL>/api/v1/executions/5678/stop' \
  -H 'X-N8N-API-KEY: <key>'

# Step 2: Then delete it
curl -X DELETE '<BASE_URL>/api/v1/executions/5678' \
  -H 'X-N8N-API-KEY: <key>'
```

ALWAYS stop a running execution before attempting to delete it.

---

## AP-4: Requesting More Than 250 Items Per Page

### Wrong

```bash
# WRONG: limit > 250 is silently capped
curl '<BASE_URL>/api/v1/workflows?limit=1000' \
  -H 'X-N8N-API-KEY: <key>'
# Returns at most 250 items, NOT 1000
```

### Why It Fails

The maximum page size is 250. Values above 250 are silently capped -- no error is returned, but you receive fewer items than expected. Code that assumes it received all 1000 items will miss data.

### Correct

```bash
# Use limit=250 and paginate with cursor
cursor=""
while true; do
  response=$(curl -s "<BASE_URL>/api/v1/workflows?limit=250&cursor=$cursor" \
    -H 'X-N8N-API-KEY: <key>')
  # Process data...
  cursor=$(echo "$response" | jq -r '.nextCursor // empty')
  [ -z "$cursor" ] && break
done
```

ALWAYS use `limit=250` (the maximum) combined with cursor-based pagination to fetch all results.

---

## AP-5: Creating Credentials Without Checking the Type Schema

### Wrong

```bash
# WRONG: Guessing field names for a credential type
curl -X POST '<BASE_URL>/api/v1/credentials' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <key>' \
  -d '{
    "name": "My Slack",
    "type": "slackApi",
    "data": { "token": "xoxb-..." }
  }'
# May fail: wrong field names for the credential type
```

### Why It Fails

Each credential type has a specific schema with exact field names. Guessing field names results in credentials that appear valid but do not work when used by nodes. The field might be `accessToken` instead of `token`, or additional required fields may be missing.

### Correct

```bash
# Step 1: Fetch the credential type schema
curl '<BASE_URL>/api/v1/credential-types/slackApi' \
  -H 'X-N8N-API-KEY: <key>'
# Returns: field definitions with names, types, required flags

# Step 2: Create credential using correct field names from schema
curl -X POST '<BASE_URL>/api/v1/credentials' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <key>' \
  -d '{
    "name": "My Slack",
    "type": "slackApi",
    "data": { "accessToken": "xoxb-..." }
  }'
```

ALWAYS fetch the credential type schema via `GET /credential-types/:typeName` before creating credentials programmatically.

---

## AP-6: Not Handling Pagination at All

### Wrong

```python
# WRONG: Assuming a single request returns all results
response = requests.get(f"{base}/api/v1/executions", headers=headers)
all_executions = response.json()["data"]
# Missing executions beyond the first 100!
```

### Why It Fails

List endpoints return a maximum of 250 results per request (default: 100). If your instance has more items, you silently miss them. This leads to incomplete dashboards, monitoring gaps, and data loss in migration scripts.

### Correct

```python
all_executions = []
cursor = None
while True:
    params = {"limit": 250}
    if cursor:
        params["cursor"] = cursor
    response = requests.get(f"{base}/api/v1/executions", headers=headers, params=params)
    result = response.json()
    all_executions.extend(result["data"])
    cursor = result.get("nextCursor")
    if not cursor:
        break
```

ALWAYS implement cursor-based pagination loops when fetching list endpoints. NEVER assume a single request returns all results.

---

## AP-7: Activating a Workflow Immediately After Creation in One Call

### Wrong

```bash
# WRONG: Trying to create an active workflow in one step
curl -X POST '<BASE_URL>/api/v1/workflows' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <key>' \
  -d '{ "name": "Test", "active": true, "nodes": [...] }'
# The "active" field in the body may be ignored
```

### Why It Fails

Setting `active: true` in the create request body does not reliably activate the workflow. The activation must be done as a separate API call.

### Correct

```bash
# Step 1: Create the workflow
response=$(curl -s -X POST '<BASE_URL>/api/v1/workflows' \
  -H 'Content-Type: application/json' \
  -H 'X-N8N-API-KEY: <key>' \
  -d '{ "name": "Test", "nodes": [...], "connections": {} }')

workflow_id=$(echo "$response" | jq -r '.id')

# Step 2: Activate it separately
curl -X POST "<BASE_URL>/api/v1/workflows/${workflow_id}/activate" \
  -H 'X-N8N-API-KEY: <key>'
```

ALWAYS create the workflow first, then activate it in a separate POST request to `/workflows/:id/activate`.

---

## AP-8: Using the API on Free Trial Instances

### Wrong

```bash
# WRONG: Attempting API access on a free trial
curl '<BASE_URL>/api/v1/workflows' \
  -H 'X-N8N-API-KEY: <key>'
# Returns 403 or feature not available
```

### Why It Fails

The n8n Public REST API requires a paid plan. Free trial instances do not have API access enabled. No amount of API key configuration will make it work.

### Correct

Upgrade to a paid n8n plan before attempting programmatic API access. For self-hosted instances, the API is available by default unless explicitly disabled via `N8N_PUBLIC_API_DISABLED=true`.

---

## AP-9: Ignoring Error Responses

### Wrong

```python
# WRONG: Not checking response status
response = requests.delete(f"{base}/api/v1/workflows/{wf_id}", headers=headers)
print("Deleted successfully")  # May not be true!
```

### Why It Fails

The API returns meaningful HTTP status codes (401, 403, 404, 409, 500). Ignoring these means your code silently fails -- reporting success when the operation actually failed.

### Correct

```python
response = requests.delete(f"{base}/api/v1/workflows/{wf_id}", headers=headers)
if response.status_code == 200:
    print("Deleted successfully")
elif response.status_code == 404:
    print(f"Workflow {wf_id} not found")
elif response.status_code == 401:
    print("Invalid API key")
else:
    print(f"Unexpected error: {response.status_code} - {response.json().get('message', '')}")
```

ALWAYS check HTTP status codes and handle error responses appropriately. NEVER assume API calls succeed.

---

## AP-10: Disabling the API Playground in Development

### Wrong

```env
# WRONG: Disabling Swagger UI during development
N8N_PUBLIC_API_SWAGGERUI_DISABLED=true
```

### Why It Fails

The API playground (Scalar UI at `/api/v1/docs`) is the fastest way to explore and test API endpoints during development. Disabling it forces developers to rely on external documentation, slowing down development and increasing errors.

### Correct

Keep the API playground enabled during development:

```env
# Development: keep playground enabled (default)
# N8N_PUBLIC_API_SWAGGERUI_DISABLED=false

# Production: disable for security
N8N_PUBLIC_API_SWAGGERUI_DISABLED=true
```

ALWAYS keep the API playground enabled in development environments. Only disable it in production for security hardening.
