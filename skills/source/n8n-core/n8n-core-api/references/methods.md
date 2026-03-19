# n8n REST API -- Complete Endpoint Reference

All endpoints are under `/api/v1/`. Authentication via `X-N8N-API-KEY` header is required for ALL endpoints.

---

## Workflows

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/workflows` | `workflow:list` | List workflows (paginated). Query params: `limit`, `cursor`, `active`, `tags`, `name`, `projectId`, `excludePinnedData` |
| POST | `/workflows` | `workflow:create` | Create a new workflow. Body: workflow JSON |
| GET | `/workflows/:id` | `workflow:read` | Get single workflow by ID. Query: `excludePinnedData` |
| PUT | `/workflows/:id` | `workflow:update` | Update workflow content. Body: workflow JSON |
| DELETE | `/workflows/:id` | `workflow:delete` | Delete workflow permanently |
| GET | `/workflows/:id/versions/:versionId` | `workflow:read` | Get specific workflow version |
| POST | `/workflows/:id/activate` | `workflow:activate` | Activate a workflow (enables triggers) |
| POST | `/workflows/:id/deactivate` | `workflow:deactivate` | Deactivate a workflow (disables triggers) |
| POST | `/workflows/:id/transfer` | `workflow:move` | Transfer workflow to another project. Body: `{ "destinationProjectId": "<id>" }` |
| GET | `/workflows/:id/tags` | `workflowTags:list` | Get tags assigned to a workflow |
| PUT | `/workflows/:id/tags` | `workflowTags:update` | Update workflow tags. Body: array of tag IDs |

### Workflow Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `active` | boolean | Filter by active/inactive status |
| `tags` | string | Comma-separated tag IDs to filter by |
| `name` | string | Filter by workflow name (partial match) |
| `projectId` | string | Filter by project ID |
| `excludePinnedData` | boolean | Exclude pinned data from response (reduces payload size) |
| `limit` | number | Page size (default: 100, max: 250) |
| `cursor` | string | Pagination cursor from previous response |

---

## Executions

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/executions` | `execution:list` | List executions (paginated). Query params: `lastId`, `limit`, `status`, `workflowId`, `projectId`, `includeData` |
| GET | `/executions/:id` | `execution:read` | Get single execution. Query: `includeData` |
| DELETE | `/executions/:id` | `execution:delete` | Delete execution. NEVER delete a running execution -- stop it first |
| POST | `/executions/:id/retry` | `execution:retry` | Retry a failed execution |
| POST | `/executions/:id/stop` | `execution:stop` | Stop a single running execution |
| POST | `/executions/stop` | `execution:stop` | Stop multiple executions. Body: `{ "status": [], "workflowId": "", "startedAfter": "", "startedBefore": "" }` |
| GET | `/executions/:id/tags` | `executionTags:list` | Get tags assigned to an execution |
| PATCH | `/executions/:id/tags` | `executionTags:update` | Update execution tags |

### Execution Query Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `lastId` | string | Return executions after this ID (alternative pagination) |
| `status` | string | Filter by status: `error`, `success`, `waiting`, `running`, `crashed` |
| `workflowId` | string | Filter executions by workflow ID |
| `projectId` | string | Filter executions by project ID |
| `includeData` | boolean | Include full execution data in response (larger payload) |
| `limit` | number | Page size (default: 100, max: 250) |
| `cursor` | string | Pagination cursor from previous response |

---

## Credentials

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/credentials` | `credential:list` | List credentials (paginated). Query params: `limit`, `cursor` |
| POST | `/credentials` | `credential:create` | Create new credential. Body: credential JSON with type and data |
| PUT | `/credentials/:id` | `credential:update` | Update credential data |
| DELETE | `/credentials/:id` | `credential:delete` | Delete credential |
| POST | `/credentials/:id/transfer` | `credential:move` | Transfer credential to another project. Body: `{ "destinationProjectId": "<id>" }` |
| GET | `/credential-types/:credentialTypeName` | — | Get credential type schema (field definitions, required fields) |

### Credential Body Structure

```json
{
  "name": "My API Credential",
  "type": "httpBasicAuth",
  "data": {
    "user": "username",
    "password": "secret"
  }
}
```

ALWAYS use the `/credential-types/:typeName` endpoint to discover the correct field names before creating credentials programmatically.

---

## Users

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users` | List all users |
| GET | `/users/:id` | Get single user by ID |
| — | `/users/:id/role` | Manage user role (enterprise) |

---

## Tags

| Method | Path | Description |
|--------|------|-------------|
| GET | `/tags` | List all tags |
| POST | `/tags` | Create a new tag. Body: `{ "name": "<tag-name>" }` |
| GET | `/tags/:id` | Get single tag by ID |
| PUT | `/tags/:id` | Update tag. Body: `{ "name": "<new-name>" }` |
| DELETE | `/tags/:id` | Delete tag |

---

## Variables

| Method | Path | Description |
|--------|------|-------------|
| GET | `/variables` | List all variables |
| POST | `/variables` | Create a variable. Body: `{ "key": "<key>", "value": "<value>" }` |
| GET | `/variables/:id` | Get single variable by ID |
| PUT | `/variables/:id` | Update variable. Body: `{ "key": "<key>", "value": "<value>" }` |
| DELETE | `/variables/:id` | Delete variable |

---

## Projects

| Method | Path | Description |
|--------|------|-------------|
| GET | `/projects` | List all projects |
| POST | `/projects` | Create a project |
| GET | `/projects/:projectId` | Get single project |
| PUT | `/projects/:projectId` | Update project |
| DELETE | `/projects/:projectId` | Delete project |
| GET | `/projects/:projectId/users` | List users in a project |
| PUT | `/projects/:projectId/users/:userId` | Add/update user in project |

---

## Other Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/audit` | Run a security audit on the n8n instance |
| POST | `/source-control/pull` | Pull latest changes from source control |
| CRUD | `/data-tables/*` | Data table operations (rows, update, upsert, delete) |

---

## Response Format

All list endpoints return:

```json
{
  "data": [ /* array of resources */ ],
  "nextCursor": "MTIzNA=="
}
```

Single resource endpoints return the resource object directly:

```json
{
  "id": "123",
  "name": "My Workflow",
  "active": true,
  "nodes": [ /* ... */ ],
  "connections": { /* ... */ }
}
```

### Error Response Format

```json
{
  "message": "Description of the error"
}
```

Common HTTP status codes:

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 400 | Bad request (invalid parameters) |
| 401 | Unauthorized (missing or invalid API key) |
| 403 | Forbidden (insufficient scope) |
| 404 | Resource not found |
| 409 | Conflict (e.g., deleting a running execution) |
| 500 | Internal server error |
