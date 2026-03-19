# Error Handling Examples

## Example 1: Basic Error Workflow with Slack Notification

### Error Workflow (receives errors from any assigned workflow)

```
[Error Trigger] → [Slack]
```

**Error Trigger node**: No configuration needed — it receives error data automatically.

**Slack node configuration**:
- Channel: `#workflow-errors`
- Message:

```
Workflow "{{ $json.execution.lastNodeExecuted }}" failed in "{{ $json.workflow.name }}"

Error: {{ $json.execution.error.message }}
Execution: {{ $json.execution.url }}
Mode: {{ $json.execution.mode }}
```

**Main workflow assignment**:
1. Open the main workflow > Workflow Settings
2. Set **Error Workflow** to the error workflow created above

---

## Example 2: Error Workflow with Rich Context

### Error Workflow with conditional routing by error severity

```
[Error Trigger] → [Switch] → [Slack] (critical errors)
                            → [Email] (warning-level errors)
                            → [HTTP Request] (log to external service)
```

**Switch node conditions**:
- Route 1 (critical): `{{ $json.execution.error.message }}` contains "timeout" OR "out of memory"
- Route 2 (warning): `{{ $json.execution.error.message }}` contains "rate limit" OR "temporarily unavailable"
- Route 3 (all): Fallback — log everything to external monitoring

---

## Example 3: continueOnFail with Fallback Value

### Scenario: Fetch user data from API, use cached data if API fails

```
[Schedule Trigger] → [HTTP Request (continueOnFail=true)] → [IF] → [Set: use API data]
                                                                  → [Set: use fallback data]
```

**HTTP Request node**:
- Settings tab > Continue On Fail: **Enabled**
- URL: `https://api.example.com/users`

**IF node**:
- Condition: `{{ $json.error }}` is not empty
- True (error path): Connect to Set node with fallback/cached data
- False (success path): Connect to Set node processing API response

**Set node (fallback)**:
```json
{
  "users": [],
  "source": "fallback",
  "error": "{{ $json.error.message }}"
}
```

---

## Example 4: continueOnFail with Error Logging

### Scenario: Process batch of items, log failures, continue with successes

```
[Get Items] → [Split In Batches] → [HTTP Request (continueOnFail=true)] → [IF] → [Merge successes]
                                                                                 → [Log failures]
```

**HTTP Request node**:
- Settings tab > Continue On Fail: **Enabled**
- Process each item individually against external API

**IF node**:
- Condition: `{{ $json.error }}` is not empty
- True (error): Route to a Set node that logs the failed item and error
- False (success): Route to downstream processing

This pattern ensures one failed item does not stop the entire batch.

---

## Example 5: Retry on Fail for External API

### Scenario: Call a rate-limited API with automatic retries

```
[Schedule Trigger] → [HTTP Request (retryOnFail)] → [Process Data]
```

**HTTP Request node settings**:
- Settings tab > Retry On Fail: **Enabled**
- Max Tries: `5`
- Wait Between Tries: `2000` (2 seconds)

This handles:
- HTTP 429 (Too Many Requests) — retry after wait
- HTTP 503 (Service Unavailable) — retry after wait
- Network timeouts — retry after wait

If all 5 retries fail, the error propagates to the Error Workflow.

---

## Example 6: Retry Combined with continueOnFail

### Scenario: Best-effort API call — retry first, then fall back gracefully

```
[Trigger] → [HTTP Request (retry=3, continueOnFail=true)] → [IF] → [Success path]
                                                                   → [Fallback path]
```

**HTTP Request node settings**:
- Retry On Fail: **Enabled** (maxTries=3, wait=1000ms)
- Continue On Fail: **Enabled**

**Execution order**:
1. Node attempts execution
2. On failure, retries up to 3 times
3. If all retries fail, continueOnFail captures the error in `$json.error`
4. IF node routes to fallback path

This is the most resilient pattern for non-critical external calls.

---

## Example 7: Stop And Error for Validation

### Scenario: Validate incoming webhook data before processing

```
[Webhook] → [IF (validate)] → [Process Data]
                             → [Stop And Error]
```

**IF node**:
- Condition: `{{ $json.body.email }}` is not empty AND `{{ $json.body.amount }}` > 0
- True: Continue to processing
- False: Route to Stop And Error

**Stop And Error node**:
- Error Type: Error Message
- Error Message: `Validation failed: missing email or invalid amount in webhook payload`

**Result**: The Error Workflow fires with the custom validation message, making it clear exactly what went wrong.

---

## Example 8: Stop And Error with Error Object

### Scenario: Return structured error data to the error workflow

```
[Webhook] → [Code (validate)] → [IF] → [Process]
                                      → [Stop And Error]
```

**Stop And Error node**:
- Error Type: Error Object
- Error Object:

```json
{
  "code": "VALIDATION_ERROR",
  "field": "email",
  "message": "Email address is required but was not provided",
  "receivedData": "{{ JSON.stringify($json.body) }}"
}
```

The Error Workflow receives this structured object, enabling automated error categorization and routing.

---

## Example 9: Execution Timeout with Notification

### Scenario: Ensure long-running workflows do not run forever

**Environment variable** (instance-level):
```
EXECUTIONS_TIMEOUT=600
EXECUTIONS_TIMEOUT_MAX=3600
```

**Per-workflow override** (for a known slow workflow):
1. Open Workflow Settings
2. Set Timeout After: `1800` (30 minutes)

**Error Workflow** catches the timeout:
```
[Error Trigger] → [IF (message contains "timeout")] → [Slack: "Workflow timed out"]
                                                      → [Slack: "Workflow failed (other)"]
```

---

## Example 10: Complete Production Error Handling Setup

### Main workflow with all 4 patterns applied

```
[Schedule Trigger]
  → [HTTP Request 1 (retry=3, wait=2000)]        ← Pattern 3: Retry transient failures
  → [HTTP Request 2 (continueOnFail=true)]        ← Pattern 2: Graceful degradation
  → [IF ($json.error)]
    → True: [Set: fallback data]
    → False: [Process API response]
  → [Code: validate results]
  → [IF (valid)]
    → True: [Save to Database]
    → False: [Stop And Error: "Validation failed"]  ← Pattern 4: Deliberate failure
  → Workflow Settings: Error Workflow assigned       ← Pattern 1: Catch-all notification
```

This workflow:
1. Retries the first API call automatically (transient errors)
2. Falls back gracefully if the second API call fails
3. Validates processed data and fails explicitly if invalid
4. Sends notifications via Error Workflow for any unhandled failure
