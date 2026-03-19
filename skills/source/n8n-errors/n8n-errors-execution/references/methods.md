# Error Handling Methods Reference

## Error Trigger Node

The Error Trigger node is a trigger node that starts an error workflow when another workflow fails.

### Configuration

- **Node type**: `n8n-nodes-base.errorTrigger`
- **Placement**: ALWAYS use as the first node in an error workflow
- **No parameters**: The node receives error data automatically
- **One error workflow can serve multiple production workflows**

### Error Data Output

The Error Trigger node outputs a single item with this structure:

| Field | Type | Description |
|-------|------|-------------|
| `execution.id` | string | ID of the failed execution |
| `execution.url` | string | Direct URL to inspect the failed execution |
| `execution.retryOf` | string/null | ID of execution this was a retry of |
| `execution.error.message` | string | Human-readable error description |
| `execution.error.stack` | string | Full error stack trace |
| `execution.lastNodeExecuted` | string | Name of the node that failed |
| `execution.mode` | string | How the workflow was triggered (trigger, webhook, manual, etc.) |
| `workflow.id` | string | ID of the failed workflow |
| `workflow.name` | string | Name of the failed workflow |

### Accessing Error Data in Expressions

```
{{ $json.execution.error.message }}
{{ $json.execution.lastNodeExecuted }}
{{ $json.workflow.name }}
{{ $json.execution.url }}
{{ $json.execution.id }}
```

---

## Stop And Error Node

The Stop And Error node deliberately causes a workflow execution to fail with a custom error message.

### Configuration

| Parameter | Type | Description |
|-----------|------|-------------|
| Error Type | dropdown | "Error Message" (string) or "Error Object" (JSON) |
| Error Message | string | Custom error message (when Error Type = "Error Message") |
| Error Object | JSON | Custom error object (when Error Type = "Error Object") |

### Behavior

- Immediately halts workflow execution
- Triggers the configured Error Workflow (if one is set)
- The custom error message appears in `execution.error.message` in the Error Trigger node
- Execution status is set to "error"

### Use Cases

- Validate input data and fail with descriptive message if invalid
- Enforce business rules (e.g., "Order total exceeds maximum allowed")
- Convert conditional checks into actionable error notifications

---

## continueOnFail Setting

Per-node setting that prevents a node failure from stopping the entire workflow.

### Configuration

- **Location**: Node Settings tab > Continue On Fail
- **Type**: Boolean (on/off)
- **Default**: Off (disabled)

### Behavior When Enabled

1. Node attempts execution normally
2. If the node fails, instead of stopping the workflow:
   - The error is captured in `$json.error` on the output
   - The workflow continues to the next connected node
   - The error object contains `message` and `description` fields

### Error Output Structure

When a node fails with continueOnFail enabled:

```json
{
  "error": {
    "message": "The error message from the failed node",
    "description": "Additional error details (if available)"
  }
}
```

### Checking for Errors in Next Node

Use an IF node to branch on error:

- **Condition**: `{{ $json.error }}` exists (is not empty)
- **True branch**: Error handling/fallback logic
- **False branch**: Normal success processing

### Important Notes

- ALWAYS pair with an IF node to handle both error and success paths
- The error output replaces the normal node output entirely
- If the node was supposed to produce multiple items, only one item with the error is output
- continueOnFail does NOT retry — it immediately captures the first failure

---

## Retry on Fail Configuration

Per-node setting that automatically retries a failed node before reporting failure.

### Configuration

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| Retry On Fail | boolean | false | Enable automatic retries |
| Max Tries | number | 3 | Maximum number of retry attempts |
| Wait Between Tries (ms) | number | 1000 | Milliseconds to wait between retries |

- **Location**: Node Settings tab > Retry On Fail

### Behavior

1. Node attempts execution
2. If execution fails AND Retry On Fail is enabled:
   - Wait for `waitBetweenTries` milliseconds
   - Retry execution (up to `maxTries` times)
   - If any retry succeeds, workflow continues normally
   - If ALL retries fail, the error propagates (stops workflow or goes to Error Workflow)

### When to Use Retry

| Scenario | Recommended Config |
|----------|-------------------|
| External API rate limiting | maxTries=5, wait=2000ms |
| Temporary network issues | maxTries=3, wait=1000ms |
| Database connection drops | maxTries=3, wait=5000ms |
| Third-party service maintenance | maxTries=3, wait=10000ms |

### When NOT to Use Retry

- Authentication failures (401/403) — retrying with same credentials will always fail
- Schema/validation errors (400) — retrying with same data will always fail
- Resource not found (404) — retrying will not create the resource
- Permission errors — retrying will not grant permissions

---

## Execution Timeout Configuration

### Instance-Level Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_TIMEOUT` | `-1` (disabled) | Default timeout for all workflows (seconds) |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Maximum timeout users can set per workflow (seconds) |

### Per-Workflow Setting

- **Location**: Workflow Settings > Timeout After
- **Type**: Number (seconds)
- **Constraint**: Cannot exceed `EXECUTIONS_TIMEOUT_MAX`
- **Override**: Per-workflow setting overrides the instance default

### Timeout Behavior

1. When a workflow exceeds its timeout:
   - Execution is forcefully stopped
   - Execution status is set to "error"
   - Error message: "Execution was stopped because it reached the timeout"
   - Error Workflow fires (if configured)

### Recommended Timeout Values

| Workflow Type | Timeout | Rationale |
|---------------|---------|-----------|
| API-triggered (webhook) | 30-120s | Users expect fast responses |
| Scheduled data sync | 300-600s | May process large datasets |
| Report generation | 600-1800s | Complex aggregation takes time |
| File processing | 1800-3600s | Large files need time |

---

## Workflow Settings for Error Handling

### Accessing Workflow Settings

1. Open the workflow in the editor
2. Click the three-dot menu (top right) or press Settings
3. Navigate to the **Error Workflow** dropdown

### Error Workflow Assignment

- Select any workflow that starts with an Error Trigger node
- One error workflow can be assigned to multiple production workflows
- A workflow CANNOT be its own error workflow
- If no error workflow is assigned, errors are only visible in the Executions tab

### Auto-Deactivation Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_WORKFLOW_AUTODEACTIVATION_ENABLED` | `false` | Auto-unpublish after repeated crashes |
| `N8N_WORKFLOW_AUTODEACTIVATION_MAX_LAST_EXECUTIONS` | `3` | Consecutive failures before auto-deactivation |

When enabled, n8n automatically deactivates workflows that fail repeatedly, preventing infinite error loops.

---

## Execution Data Inspection

### Via UI

1. Open the **Executions** tab in the workflow editor
2. Filter by status: Success, Error, Waiting, Running
3. Click on a failed execution to see:
   - Which node failed (highlighted in red)
   - Input data to the failed node
   - Error message and stack trace
   - Output of all previous successful nodes

### Via REST API

```
GET /api/v1/executions?status=error&workflowId=<ID>&includeData=true
GET /api/v1/executions/<executionId>?includeData=true
POST /api/v1/executions/<executionId>/retry
```

### Execution Data Retention

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_DATA_SAVE_ON_ERROR` | `all` | Save execution data on error |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `all` | Save execution data on success |
| `EXECUTIONS_DATA_PRUNE` | `true` | Auto-delete old executions |
| `EXECUTIONS_DATA_MAX_AGE` | `336` | Max age in hours (14 days) |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` | Max stored executions |

ALWAYS keep `EXECUTIONS_DATA_SAVE_ON_ERROR` set to `all` in production to enable post-mortem debugging.
