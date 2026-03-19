# n8n Expression Anti-Patterns

## Critical: Variable Availability Errors

### NEVER use $itemIndex in Code node

**Wrong:**
```js
// Code node — WILL FAIL
const index = $itemIndex;
const item = items[$itemIndex];
```

**Correct:**
```js
// Code node — use loop counter
for (let i = 0; i < items.length; i++) {
    const item = items[i];
    item.json.position = i;
}
return items;
```

**Why:** `$itemIndex` is NOT available in the Code node. It only works in expression fields on regular node parameters.

---

### NEVER use $secrets in Code node

**Wrong:**
```js
// Code node — WILL FAIL
const apiKey = $secrets.vault.apiKey;
```

**Correct (option A):** Use `$env` or `$vars` instead:
```js
// Code node
const apiKey = $env.API_KEY;
const token = $vars.authToken;
```

**Correct (option B):** Pass via node parameter expression, then access in Code node:
```
// In a Set node before the Code node, create a field:
// apiKey = {{ $secrets.vault.apiKey }}
// Then in Code node:
const apiKey = $json.apiKey;
```

**Why:** `$secrets` is restricted to expression fields only. It is deliberately blocked in the Code node for security.

---

## Critical: JMESPath Parameter Order

### NEVER reverse $jmespath parameters

**Wrong:**
```
{{ $jmespath("[*].name", $json.data) }}
```

**Correct:**
```
{{ $jmespath($json.data, "[*].name") }}
```

**Why:** The n8n `$jmespath()` function takes `(object, searchString)`. This is the opposite of the JMESPath specification's `search(searchString, object)`. The n8n implementation follows the JMESPath JavaScript library convention.

---

## Date/Time Mistakes

### NEVER use new Date() in expressions

**Wrong:**
```
{{ new Date().toISOString() }}
{{ new Date($json.timestamp) }}
```

**Correct:**
```
{{ $now.toISO() }}
{{ DateTime.fromISO($json.timestamp) }}
{{ DateTime.fromMillis($json.unixMs) }}
{{ $json.dateString.toDateTime() }}
```

**Why:** `new Date()` does not respect the workflow's timezone settings. Luxon's `$now`, `$today`, and `DateTime` methods handle timezone awareness consistently.

---

### NEVER assume $now and $today are the same

**Wrong:**
```
// Expecting $today to include current time
{{ $today.format("HH:mm") }}
// Returns: "00:00" — NOT the current time
```

**Correct:**
```
// $today = midnight, $now = current moment
{{ $now.format("HH:mm") }}     // Current time: "14:32"
{{ $today.format("HH:mm") }}   // Always: "00:00"
```

**Why:** `$today` is midnight at the start of the current day. Use `$now` when you need the current time.

---

## Node Reference Mistakes

### NEVER reference a node that may not have executed

**Wrong:**
```
{{ $("Optional Branch").first().json.value }}
// Throws error if "Optional Branch" was skipped
```

**Correct:**
```
{{ $("Optional Branch").isExecuted
    ? $("Optional Branch").first().json.value
    : "default" }}
```

**Why:** If a node is in a branch that was not taken (e.g., after an IF node), referencing it directly throws an error. ALWAYS check `.isExecuted` first.

---

### NEVER misspell node names in $() references

**Wrong:**
```
{{ $("HTTP request").first().json.data }}
// Case matters! The actual node name is "HTTP Request"
```

**Correct:**
```
{{ $("HTTP Request").first().json.data }}
```

**Why:** Node name references in `$()` are case-sensitive and must match exactly, including spaces and special characters.

---

### NEVER use .item in Code node for paired items

**Wrong:**
```js
// Code node — .item may not resolve correctly
const original = $("Webhook").item.json.body;
```

**Correct:**
```js
// Code node — use .itemMatching() with explicit index
for (let i = 0; i < items.length; i++) {
    const original = $("Webhook").itemMatching(i);
    items[i].json.webhookBody = original.json.body;
}
return items;
```

**Why:** In the Code node, `.item` does not have the same automatic item linking context as expression fields. ALWAYS use `.itemMatching(index)` in Code nodes.

---

## Data Access Mistakes

### NEVER assume dot notation works for all field names

**Wrong:**
```
{{ $json.field-name }}
// JavaScript interprets this as $json.field minus name
```

**Correct:**
```
{{ $json["field-name"] }}
{{ $json["field with spaces"] }}
{{ $json["123startsWithNumber"] }}
```

**Why:** Dot notation only works for valid JavaScript identifiers. Fields containing dashes, spaces, or starting with numbers require bracket notation.

---

### NEVER access nested properties without null checks

**Wrong:**
```
{{ $json.response.data.items[0].name }}
// Throws if response, data, items, or [0] is undefined
```

**Correct:**
```
{{ $json.response?.data?.items?.[0]?.name ?? "N/A" }}
```

**Or with $ifEmpty:**
```
{{ $ifEmpty($json.response?.data?.items?.[0]?.name, "N/A") }}
```

**Why:** Deep property access without optional chaining throws an error if any intermediate property is `null` or `undefined`.

---

### NEVER confuse $input.all() with $json

**Wrong:**
```
// Expecting $json to contain all items
{{ $json.length }}
// $json is ONE item's data, not an array of all items
```

**Correct:**
```
// Count all items
{{ $input.all().length }}

// Access current single item
{{ $json.fieldName }}
```

**Why:** `$json` is shorthand for the current item's JSON data. To work with all items, use `$input.all()`.

---

## Static Data Mistakes

### NEVER use static data during manual test execution

**Wrong:**
```js
// In Code node during manual test run
const staticData = $getWorkflowStaticData('global');
staticData.counter = (staticData.counter || 0) + 1;
// Counter will NOT persist — requires active workflow
```

**Correct approach:** Static data only persists when the workflow is active and triggered by a trigger/webhook. During manual test executions, static data behaves like a temporary empty object.

**Why:** Workflow static data is only saved on successful execution of an active workflow. Manual test runs do not persist changes.

---

### NEVER store large datasets in static data

**Wrong:**
```js
const staticData = $getWorkflowStaticData('global');
staticData.allRecords = items.map(i => i.json);
// Storing thousands of records = performance problems
```

**Correct:**
```js
const staticData = $getWorkflowStaticData('global');
// Store only what you need: IDs, timestamps, counters
staticData.lastProcessedId = items[items.length - 1].json.id;
staticData.lastRunTimestamp = $now.toISO();
```

**Why:** Static data is stored in the n8n database and loaded on every execution. Large objects cause performance degradation and may be unreliable during high-frequency executions.

---

## Expression Context Mistakes

### NEVER use $response outside HTTP Request node

**Wrong:**
```
// In a Set node after HTTP Request
{{ $response.statusCode }}
// $response is NOT available here
```

**Correct:**
```
// In the HTTP Request node's own "Next Page URL" or similar parameter
{{ $response.statusCode }}

// Or access the output in a subsequent node via $json
{{ $json.statusCode }}
```

**Why:** `$response` is only available within the HTTP Request node's own parameter fields (e.g., pagination settings). In subsequent nodes, use `$json` or `$("HTTP Request").item.json`.

---

### NEVER use $pageCount outside HTTP Request pagination

**Wrong:**
```
// In a regular node expression
{{ $pageCount }}
// undefined — only exists in HTTP Request pagination context
```

**Why:** `$pageCount` is exclusively available within the HTTP Request node when pagination is configured.

---

## IIFE Mistakes

### NEVER forget the outer {{ }} wrapper for IIFE

**Wrong:**
```
(function() { return $json.value * 2; })()
```

**Correct:**
```
{{ (function() { return $json.value * 2; })() }}
```

**Why:** All expressions in n8n parameter fields MUST be wrapped in `{{ }}`.

---

### NEVER use IIFE when a simple expression suffices

**Wrong:**
```
{{ (function() { return $json.name; })() }}
```

**Correct:**
```
{{ $json.name }}
```

**Why:** IIFEs add unnecessary complexity. Only use them when you need multiple statements, variable declarations, or control flow that cannot be expressed inline.

---

## Python Code Node Mistakes

### NEVER use $ prefix in Python Code node

**Wrong:**
```python
# Python Code node
data = $json.field
env = $env.API_KEY
```

**Correct:**
```python
# Python Code node — use _ prefix
data = _json["field"]
env = _env["API_KEY"]
```

**Why:** Python Code nodes use `_` (underscore) prefix instead of `$` (dollar sign). Additionally, Python requires bracket notation for property access.

---

### NEVER use dot notation for item access in Python

**Wrong:**
```python
value = _json.fieldName
```

**Correct:**
```python
value = _json["fieldName"]
```

**Why:** Python item data uses dictionary-style access. Dot notation is not supported for n8n item data in Python Code nodes.
