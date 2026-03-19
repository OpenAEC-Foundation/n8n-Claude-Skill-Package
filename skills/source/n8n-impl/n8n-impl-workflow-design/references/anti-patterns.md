# Workflow Design Anti-Patterns

## 1. No Error Workflow Configured

**Anti-pattern**: Deploying a production workflow without an error workflow.

**What happens**: When the workflow fails, the failure is silently logged in the execution history. Nobody is notified. The issue may go undetected for hours or days.

**Fix**: ALWAYS assign an error workflow in the workflow settings of every production workflow. Create a centralized error handler that sends notifications via Slack, email, or another alerting channel.

---

## 2. Copy-Pasting Logic Instead of Sub-Workflows

**Anti-pattern**: Duplicating the same node sequence in multiple workflows instead of extracting it into a sub-workflow.

**What happens**: When the logic needs to change, you must update it in every workflow. Missed updates cause inconsistent behavior and bugs.

**Fix**: ALWAYS extract shared logic into a sub-workflow called via the Execute Workflow node. NEVER duplicate more than 3 nodes across workflows.

---

## 3. No Retry on External API Calls

**Anti-pattern**: Using HTTP Request nodes to call external APIs without configuring `retryOnFail`.

**What happens**: Transient failures (network timeouts, rate limits, 502/503 errors) cause the entire workflow to fail unnecessarily.

**Fix**: ALWAYS enable `retryOnFail` on HTTP Request nodes that call external APIs. Set `maxTries: 3` and `waitBetweenTries: 2000` as a baseline. Increase for known slow APIs.

---

## 4. continueOnFail Without Error Check

**Anti-pattern**: Enabling `continueOnFail` on a node but not checking for errors afterward.

**What happens**: The failed node passes error data to the next node via `$json.error`. The next node processes the error data as if it were valid data, causing silent data corruption or unexpected results.

**Fix**: ALWAYS follow a `continueOnFail` node with an IF node that checks `{{ $json.error }}` is not empty. Route error items to a separate handling path.

```
Node (continueOnFail=true) → IF ($json.error isNotEmpty?)
    → [true] → Handle error (log, notify, fallback)
    → [false] → Continue processing
```

---

## 5. Giant Monolithic Workflows

**Anti-pattern**: Building a single workflow with 30+ nodes that handles multiple responsibilities.

**What happens**: The workflow becomes impossible to debug, test, or maintain. A failure in one part affects the entire workflow. The canvas becomes unreadable.

**Fix**: ALWAYS split workflows when they exceed 20 nodes. Use sub-workflows to separate concerns:
- One workflow per responsibility (fetch data, transform, load, notify)
- Use Execute Workflow nodes to chain them
- Each sub-workflow is independently testable

---

## 6. Using IF Chains Instead of Switch

**Anti-pattern**: Chaining 3+ IF nodes to route items to different paths based on the same field.

```
IF (status = pending) → [true] → Handle Pending
                      → [false] → IF (status = processing) → [true] → Handle Processing
                                                            → [false] → IF (status = shipped) → ...
```

**What happens**: The workflow becomes deeply nested, hard to read, and error-prone. Adding a new route requires rewiring multiple nodes.

**Fix**: ALWAYS use a Switch node when routing to 3+ destinations based on the same field. Each output is clearly labeled and independently wired.

---

## 7. No Execution Timeout

**Anti-pattern**: Running production workflows without setting `executionTimeout` in workflow settings.

**What happens**: A workflow that gets stuck (e.g., waiting for an API that never responds, infinite loop in Code node) runs indefinitely, consuming resources and potentially blocking other executions.

**Fix**: ALWAYS set `executionTimeout` in workflow settings for production workflows. A reasonable default is 300 seconds (5 minutes). For long-running batch jobs, set an appropriate higher limit.

---

## 8. Processing All Items at Once with Rate-Limited APIs

**Anti-pattern**: Sending hundreds of items to an HTTP Request node that calls a rate-limited API without using Loop Over Items.

**What happens**: The API returns 429 (Too Many Requests) errors. Even with retry, the workflow may exhaust retries and fail.

**Fix**: ALWAYS use the Loop Over Items node when calling rate-limited APIs. Set batch size to match the API's rate limit. Add a Wait node in the loop body if needed.

---

## 9. Merge By Position with Unequal Item Counts

**Anti-pattern**: Using "Merge By Position" mode when the two inputs may produce different numbers of items.

**What happens**: Items without a matching position pair are silently dropped. Data loss occurs without any error or warning.

**Fix**: Use "Merge By Fields" with a common key field when combining data from different sources. Only use "Merge By Position" when you are certain both inputs produce the same number of items in the same order.

---

## 10. Hardcoded Timezone in Expressions

**Anti-pattern**: Using hardcoded timezone strings in Code nodes or expressions instead of setting the workflow timezone.

```javascript
// WRONG
const now = DateTime.now().setZone('America/New_York');
```

**What happens**: When the workflow is moved to a different environment or the business timezone changes, the hardcoded values must be found and updated throughout the workflow.

**Fix**: ALWAYS set the timezone in workflow settings (`settings.timezone`). Use `$now` in expressions which automatically respects the workflow timezone. Set `GENERIC_TIMEZONE` on the n8n instance for consistent defaults.

---

## 11. No Fallback Output on Switch Node

**Anti-pattern**: Configuring a Switch node with specific rules but no fallback output.

**What happens**: Items that do not match any rule are silently dropped. No error is raised, and data is lost without notification.

**Fix**: ALWAYS configure a fallback output on Switch nodes. Route unmatched items to a logging node or Stop And Error node to surface unexpected data values.

---

## 12. Using Sub-Workflow Without Caller Policy

**Anti-pattern**: Creating sub-workflows without explicitly setting `callerPolicy` in the workflow settings.

**What happens**: The default policy (`workflowsFromSameOwner`) may be too restrictive or too permissive depending on your team setup. On shared instances, unauthorized workflows may call your sub-workflow, or authorized callers may be blocked.

**Fix**: ALWAYS set `callerPolicy` explicitly on every sub-workflow:
- Use `workflowsFromAList` + `callerIds` for production sub-workflows
- Use `none` for workflows that should NEVER be called as sub-workflows
- NEVER use `any` in production unless specifically required

---

## 13. Ignoring Pinned Data for Testing

**Anti-pattern**: Testing workflows by running them against live data or production APIs.

**What happens**: Test runs may modify production data, trigger real notifications, or consume API quotas. Debugging becomes difficult because the input data changes between runs.

**Fix**: ALWAYS use pinned data for testing workflows. Pin the output of trigger nodes and external data sources with representative test data. This makes tests repeatable and prevents unintended side effects.

---

## 14. Wait Node Without Timeout Consideration

**Anti-pattern**: Using a Wait node with webhook resume but no workflow execution timeout.

**What happens**: If the external system never sends the webhook callback, the execution stays in "waiting" state indefinitely, consuming system resources. Over time, many stuck executions accumulate.

**Fix**: ALWAYS set `executionTimeout` on workflows that use Wait nodes. Choose a timeout that is generous enough for the expected response time but not infinite. Combine with an error workflow to alert on timeout.

---

## 15. Wrong Merge Mode Selection

**Quick reference to avoid wrong mode selection:**

| You want to... | Correct mode | WRONG mode |
|----------------|-------------|------------|
| Combine all items into one list | Append | Merge By Position |
| Join by matching field | Merge By Fields | Merge By Position |
| Pair items 1:1 by index | Merge By Position | Append |
| Get all combinations | Multiplex | Merge By Fields |
| Use only one branch result | Choose Branch | Append |
| Complex join logic | SQL Query | Merge By Position |
