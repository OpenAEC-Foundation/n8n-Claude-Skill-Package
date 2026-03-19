# Error Handling Anti-Patterns

## Anti-Pattern 1: No Error Workflow Assigned

**Wrong**:
```
[Schedule Trigger] → [HTTP Request] → [Process Data] → [Save to DB]
(No error workflow configured)
```

**Problem**: When any node fails, the execution silently fails. The only way to discover the failure is by manually checking the Executions tab. In production, this means data loss goes unnoticed for hours or days.

**Fix**: ALWAYS assign an Error Workflow in Workflow Settings for every production workflow.

```
Workflow Settings > Error Workflow > Select "Error Handler Workflow"
```

---

## Anti-Pattern 2: continueOnFail Without Error Check

**Wrong**:
```
[HTTP Request (continueOnFail=true)] → [Process Data]
```

**Problem**: The Process Data node receives `$json.error` instead of the expected API response. It either crashes on missing fields or silently processes garbage data.

**Fix**: ALWAYS add an IF node after any continueOnFail node to check for errors.

```
[HTTP Request (continueOnFail=true)] → [IF ($json.error)] → [Fallback]
                                                            → [Process Data]
```

---

## Anti-Pattern 3: Retry on Non-Transient Errors

**Wrong**:
```
[HTTP Request (retryOnFail=true, maxTries=5)]
URL: https://api.example.com/endpoint
Auth: expired API key
```

**Problem**: Retrying with expired credentials will fail every single time. This wastes time (5 retries x wait time) and may trigger rate limits or account lockouts on the remote service.

**Fix**: NEVER enable retry for errors that are deterministic (auth failures, validation errors, 404s). Only retry for transient errors (timeouts, 429, 503).

---

## Anti-Pattern 4: Using Error Workflow as Primary Logic

**Wrong**:
```
Main Workflow: [Trigger] → [Process] → [intentionally fails]
Error Workflow: [Error Trigger] → [Do the actual important work here]
```

**Problem**: Error workflows are a safety net, not a control flow mechanism. Error data is limited compared to normal node output. This pattern is fragile, hard to debug, and loses data context.

**Fix**: Use normal workflow logic (IF nodes, Switch nodes) for control flow. Reserve error workflows for notification and recovery only.

---

## Anti-Pattern 5: Infinite Error Loop

**Wrong**:
```
Workflow A: Error Workflow = Workflow B
Workflow B: Error Workflow = Workflow A
```

Or:
```
Workflow A: Error Workflow = Workflow A (itself)
```

**Problem**: If the error workflow itself fails, it triggers the other error workflow, which triggers back, creating an infinite loop. n8n prevents a workflow from being its own error workflow, but cross-referencing loops are possible.

**Fix**: ALWAYS use a dedicated, simple error workflow that is unlikely to fail (e.g., just sends a Slack message). NEVER chain error workflows in a cycle.

---

## Anti-Pattern 6: No Execution Timeout in Production

**Wrong**:
```
Environment: EXECUTIONS_TIMEOUT=-1 (default, disabled)
```

**Problem**: A workflow that enters an infinite loop or waits indefinitely on an unresponsive API will run forever, consuming resources and potentially blocking other executions.

**Fix**: ALWAYS set `EXECUTIONS_TIMEOUT` to a reasonable value in production.

```
EXECUTIONS_TIMEOUT=600      # 10 minutes default
EXECUTIONS_TIMEOUT_MAX=3600 # 1 hour maximum
```

---

## Anti-Pattern 7: Ignoring Error Data in Error Workflow

**Wrong**:
```
[Error Trigger] → [Slack: "A workflow failed"]
```

**Problem**: The notification tells you something failed but not what, where, or why. You must manually dig through the Executions tab to find the issue.

**Fix**: ALWAYS include error context in notifications:

```
[Error Trigger] → [Slack: "{{ $json.workflow.name }} failed at {{ $json.execution.lastNodeExecuted }}: {{ $json.execution.error.message }} — {{ $json.execution.url }}"]
```

---

## Anti-Pattern 8: continueOnFail on Critical Nodes

**Wrong**:
```
[Get Payment Data (continueOnFail=true)] → [Process Payment] → [Send Receipt]
```

**Problem**: If the payment data node fails, the workflow continues with `$json.error` instead of payment data. The Process Payment node either crashes or processes invalid data, potentially charging wrong amounts or failing silently.

**Fix**: NEVER enable continueOnFail on nodes where failure means downstream data is invalid. Let it fail and handle via Error Workflow instead.

---

## Anti-Pattern 9: Excessive Retry Configuration

**Wrong**:
```
[HTTP Request]
  retryOnFail: true
  maxTries: 20
  waitBetweenTries: 30000  # 30 seconds
```

**Problem**: 20 retries with 30-second waits = 10 minutes of retrying a single node. This blocks the execution, wastes resources, and may trigger rate limits. If the service is truly down, no amount of retrying will help.

**Fix**: Use reasonable retry values. For most APIs:
- maxTries: 3-5
- waitBetweenTries: 1000-5000ms

If the service needs more than 5 retries, the problem is not transient.

---

## Anti-Pattern 10: Not Saving Execution Data on Error

**Wrong**:
```
Environment: EXECUTIONS_DATA_SAVE_ON_ERROR=none
```

**Problem**: When a workflow fails, there is no execution record to inspect. You cannot see what data was processed, which node failed, or what the error was. Post-mortem debugging becomes impossible.

**Fix**: ALWAYS keep `EXECUTIONS_DATA_SAVE_ON_ERROR=all` in production. Execution data is essential for debugging.

---

## Anti-Pattern 11: Multiple Error Workflows Without Strategy

**Wrong**:
```
Workflow 1: Error Workflow = "Error Handler A"
Workflow 2: Error Workflow = "Error Handler B"
Workflow 3: Error Workflow = "Error Handler C"
...
Workflow 20: Error Workflow = "Error Handler T"
```

**Problem**: Each production workflow has its own error workflow. Maintaining 20+ error workflows is unsustainable. When notification channels change (e.g., migrate from Slack to Teams), you must update every error workflow individually.

**Fix**: Use ONE shared error workflow for standard notifications. The Error Trigger data includes `workflow.name` and `workflow.id`, so one workflow can handle errors from all sources. Only create separate error workflows when genuinely different recovery logic is needed.

---

## Anti-Pattern 12: Swallowing Errors with Empty continueOnFail

**Wrong**:
```
[HTTP Request (continueOnFail=true)] → [Next Node]
(No IF check, no logging, error data is silently discarded)
```

**Problem**: The error is captured in `$json.error` but no node ever reads it. The workflow continues as if nothing happened, but downstream nodes receive error data instead of expected data. Failures are invisible.

**Fix**: If you use continueOnFail, you MUST handle the error — either log it, send a notification, or use a fallback value. If you do not need to handle the error, do not use continueOnFail.
