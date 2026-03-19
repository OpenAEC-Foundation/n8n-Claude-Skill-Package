# Expression Anti-Patterns

> Common mistakes in n8n expressions that lead to errors, unexpected behavior, or hard-to-debug issues.
> Each anti-pattern includes the problem, why it fails, and the correct approach.

---

## AP-01: Using $itemIndex in Code Node

**Anti-pattern:**
```js
// In Code node
const idx = $itemIndex;
```

**Why it fails:** `$itemIndex` is explicitly excluded from the Code node sandbox. It is ONLY available in expression fields of regular nodes.

**Correct approach:**
```js
// Run Once for All Items mode — use loop index
for (let i = 0; i < items.length; i++) {
  items[i].json.position = i;
}
return items;
```

---

## AP-02: Using $secrets in Code Node

**Anti-pattern:**
```js
// In Code node
const key = $secrets.provider.apiKey;
```

**Why it fails:** `$secrets` is blocked in Code node for security. The sandboxed environment does not have access to the external secrets provider.

**Correct approach:**
```js
// Option 1: Use environment variables
const key = $env.MY_API_KEY;

// Option 2: Pass secret via a Set node expression BEFORE the Code node
// Set node field: {{ $secrets.provider.apiKey }}
// Code node: const key = $json.apiKey;
```

---

## AP-03: Swapping JMESPath Parameters

**Anti-pattern:**
```js
{{ $jmespath("[*].name", $json.users) }}
```

**Why it fails:** n8n's `$jmespath()` takes `(object, searchString)` — the OPPOSITE of the JMESPath specification's `search(searchString, object)`. Swapping parameters causes silent wrong results or errors.

**Correct approach:**
```js
{{ $jmespath($json.users, "[*].name") }}
```

---

## AP-04: Accessing Nested Properties Without Null Checks

**Anti-pattern:**
```js
{{ $json.order.customer.address.city }}
```

**Why it fails:** If any intermediate property (`order`, `customer`, `address`) is null or undefined, the entire expression throws a `TypeError`.

**Correct approach:**
```js
// Optional chaining
{{ $json.order?.customer?.address?.city }}

// With fallback
{{ $ifEmpty($json.order?.customer?.address?.city, "Unknown") }}
```

---

## AP-05: Using $("<Node>").item in Code Node

**Anti-pattern:**
```js
// In Code node
const email = $("Customer Lookup").item.json.email;
```

**Why it fails:** `.item` relies on paired item tracking, which does not work reliably in Code node context. The Code node processes items differently from expression evaluation.

**Correct approach:**
```js
// Use .itemMatching() in Code node
const email = $("Customer Lookup").itemMatching(0).json.email;

// Or use explicit position access
const email = $("Customer Lookup").first().json.email;
```

---

## AP-06: Relying on Static Data During Manual Testing

**Anti-pattern:**
```js
const staticData = $getWorkflowStaticData('global');
const token = staticData.oauthToken;
// Expects token to exist during "Test Workflow" click
```

**Why it fails:** Static data is ONLY populated and persisted during production execution (via trigger/webhook). Manual test execution returns an empty object.

**Correct approach:**
```js
const staticData = $getWorkflowStaticData('global');
const token = staticData.oauthToken || "MISSING_TOKEN_USE_PRODUCTION";
// ALWAYS provide fallback; test with active workflow via webhook/trigger
```

---

## AP-07: Using new Date() Instead of Luxon

**Anti-pattern:**
```js
{{ new Date().toISOString() }}
```

**Why it fails:** `new Date()` does not respect the workflow timezone setting. It uses the server's local time, which may differ from the configured workflow timezone.

**Correct approach:**
```js
// Use $now (respects workflow timezone)
{{ $now.toISO() }}

// For specific formatting
{{ $now.format("yyyy-MM-dd HH:mm:ss") }}

// For midnight today
{{ $today.toISO() }}
```

---

## AP-08: String Date Comparison

**Anti-pattern:**
```js
{{ $json.startDate > $json.endDate }}
```

**Why it fails:** String comparison is lexicographic, not chronological. `"2024-01-15" > "2024-02-01"` evaluates to `true` because `"1" > "0"` in the fifth character position — which is logically wrong.

**Correct approach:**
```js
{{ $json.startDate.toDateTime() > $json.endDate.toDateTime() }}
```

---

## AP-09: Dot Notation in Python Code Node

**Anti-pattern:**
```python
name = item.json.name
city = _json.address.city
```

**Why it fails:** Python Code node items are dictionaries, not objects with attribute access. Dot notation causes `AttributeError`.

**Correct approach:**
```python
name = item["json"]["name"]
city = _json["address"]["city"]

# Defensive with .get()
city = _json.get("address", {}).get("city", "Unknown")
```

---

## AP-10: Using $response Outside HTTP Request Node

**Anti-pattern:**
```js
// In a Set node after HTTP Request node
{{ $response.statusCode }}
```

**Why it fails:** `$response` is ONLY available within HTTP Request node parameter fields (e.g., pagination settings, response filtering). It does not propagate to downstream nodes.

**Correct approach:**
```js
// HTTP Request node outputs response as regular $json
{{ $json.statusCode }}
// Or access specific response fields that the node outputs
```

---

## AP-11: Assuming All Fields Exist

**Anti-pattern:**
```js
{{ $json.items.length }}
// Or
{{ $json.results.map(r => r.name).join(", ") }}
```

**Why it fails:** If `items` or `results` is undefined (e.g., API returned no results field), accessing `.length` or `.map()` throws a TypeError.

**Correct approach:**
```js
{{ ($json.items || []).length }}
// Or
{{ ($json.results || []).map(r => r.name).join(", ") }}

// Better: use $ifEmpty
{{ $ifEmpty($json.items, []).length }}
```

---

## AP-12: Implicit Type Coercion in Comparisons

**Anti-pattern:**
```js
{{ $json.quantity == "5" }}
// Uses loose equality — "5" == 5 is true, but behavior is unreliable
```

**Why it fails:** Loose equality (`==`) performs type coercion that can produce unexpected results. `"0" == false` is `true`, `"" == false` is `true`.

**Correct approach:**
```js
// Explicit type conversion + strict comparison
{{ Number($json.quantity) === 5 }}

// Or convert to same type
{{ String($json.quantity) === "5" }}
```

---

## AP-13: Using Assignment in Expressions

**Anti-pattern:**
```js
{{ $json.total = $json.price * $json.quantity }}
```

**Why it fails:** Expressions are evaluated as read-only return values. Assignment operators cause `SyntaxError: Invalid left-hand side in assignment`.

**Correct approach:**
```js
// Expressions compute and return values — no assignment needed
{{ $json.price * $json.quantity }}

// For complex logic, use IIFE
{{ (function() {
  const total = $json.price * $json.quantity;
  const tax = total * 0.21;
  return total + tax;
})() }}
```

---

## AP-14: Importing External Libraries in Cloud Python

**Anti-pattern:**
```python
import requests
import pandas as pd
response = requests.get("https://api.example.com/data")
```

**Why it fails:** n8n Cloud Python Code node does NOT support external library imports. Only the Python standard library is available.

**Correct approach:**
```
Use the HTTP Request node for API calls (before the Code node).
Use native n8n nodes for data manipulation.
On self-hosted: external libraries can be allowlisted via configuration.
```

---

## AP-15: Using items.length in Python

**Anti-pattern:**
```python
count = _items.length
```

**Why it fails:** Python lists use `len()`, not `.length` (which is a JavaScript property).

**Correct approach:**
```python
count = len(_items)
```

---

## AP-16: Storing Large Data in Static Data

**Anti-pattern:**
```js
const staticData = $getWorkflowStaticData('global');
staticData.allResults = items; // Storing entire dataset
```

**Why it fails:** Static data is meant for small persistence (IDs, timestamps, counters). Large data causes performance issues and may exceed storage limits. Static data is also unreliable during high-frequency parallel executions.

**Correct approach:**
```js
const staticData = $getWorkflowStaticData('global');
staticData.lastProcessedId = items[items.length - 1].json.id;
staticData.lastRunTimestamp = new Date().toISOString();
// Store ONLY small reference values
```

---

## AP-17: Making HTTP Requests in Code Node

**Anti-pattern:**
```js
// In Code node
const response = await fetch("https://api.example.com/data");
```

**Why it fails:** The Code node sandbox blocks direct HTTP requests. `fetch`, `axios`, and other HTTP clients are not available.

**Correct approach:**
```
Use the HTTP Request node before or after the Code node.
Pass data between nodes using the standard n8n item format.
```

---

## Summary: Quick Reference

| # | Anti-Pattern | Correct Alternative |
|---|-------------|-------------------|
| AP-01 | `$itemIndex` in Code node | Loop index or `items.indexOf()` |
| AP-02 | `$secrets` in Code node | `$env` or pass via Set node |
| AP-03 | Swapped JMESPath params | `$jmespath(object, search)` |
| AP-04 | Deep access without null checks | Optional chaining `?.` |
| AP-05 | `.item` in Code node | `.itemMatching(index)` |
| AP-06 | Static data in manual test | Fallback values; test via trigger |
| AP-07 | `new Date()` | `$now` or `$today` (Luxon) |
| AP-08 | String date comparison | `.toDateTime()` then compare |
| AP-09 | Dot notation in Python | Bracket notation `["key"]` |
| AP-10 | `$response` outside HTTP node | Access via `$json` downstream |
| AP-11 | Assuming fields exist | Default with `$ifEmpty()` or `\|\|` |
| AP-12 | Loose equality `==` | Explicit conversion + `===` |
| AP-13 | Assignment `=` in expression | Return computed value directly |
| AP-14 | External imports in Cloud Python | Use n8n nodes instead |
| AP-15 | `items.length` in Python | `len(_items)` |
| AP-16 | Large data in static data | Store only small references |
| AP-17 | HTTP requests in Code node | Use HTTP Request node |
