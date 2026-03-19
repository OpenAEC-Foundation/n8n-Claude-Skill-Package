# Code Node — Anti-Patterns & Common Mistakes

## CRITICAL: Restriction Violations

### 1. Filesystem Access in Code Node

```javascript
// WRONG — causes runtime error
const fs = require('fs');
const data = fs.readFileSync('/path/to/file.txt');
```

```javascript
// CORRECT — use Read/Write Files From Disk node before the Code node,
// then access the binary data in Code node:
const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');
const content = buffer.toString('utf-8');
```

**Why:** The Code node runs in a sandboxed environment. Filesystem modules are blocked. ALWAYS use Read/Write Files From Disk nodes for file I/O.

---

### 2. HTTP Requests in Code Node

```javascript
// WRONG — fetch/axios/http are NOT available
const response = await fetch('https://api.example.com/data');
const data = await axios.get('https://api.example.com/data');
```

```javascript
// CORRECT — use HTTP Request node before the Code node,
// then access its output:
const apiData = $("HTTP Request").first().json;
return [{ json: { result: apiData } }];
```

**Why:** The sandbox blocks all network access. ALWAYS use the HTTP Request node for API calls and access its output via `$("node name")`.

---

### 3. Using `$itemIndex` in Code Node

```javascript
// WRONG — $itemIndex is NOT available in Code node
const index = $itemIndex;
```

```javascript
// CORRECT — track index manually
for (let i = 0; i < items.length; i++) {
  items[i].json.index = i;
}
return items;

// Or with forEach
items.forEach((item, index) => {
  item.json.position = index;
});
return items;
```

**Why:** `$itemIndex` is an expression-only variable. It does not exist in the Code node context.

---

### 4. Using `$secrets` in Code Node

```javascript
// WRONG — $secrets is NOT accessible in Code node
const apiKey = $secrets.myApiKey;
```

```javascript
// CORRECT — pass secrets through a preceding Set node or credential-based node
// In a Set node before the Code node, set:
//   secretValue = {{ $secrets.myApiKey }}
// Then in Code node:
const apiKey = items[0].json.secretValue;
```

**Why:** External secrets provider is not exposed to the Code node sandbox. Pass needed values through input items.

---

### 5. Using `$parameter` in Code Node

```javascript
// WRONG — $parameter is NOT available in Code node
const setting = $parameter.someConfig;
```

```javascript
// CORRECT — hardcode values or pass through input
const setting = "myConfigValue";
// Or access from a preceding Set node
const setting = items[0].json.configValue;
```

**Why:** `$parameter` references the current node's UI configuration, which is not exposed in the Code node sandbox.

---

## Return Format Errors

### Missing `json` Wrapper

```javascript
// WRONG — returns raw object without json key
return items.map(item => ({
  name: item.json.name,
  value: item.json.value
}));
// Result: data silently lost or error thrown
```

```javascript
// CORRECT — ALWAYS wrap in {json: {...}}
return items.map(item => ({
  json: {
    name: item.json.name,
    value: item.json.value
  }
}));
```

---

### Returning Single Object in All-Items Mode

```javascript
// WRONG — all-items mode MUST return an array
return { json: { total: items.length } };
```

```javascript
// CORRECT — wrap in array brackets
return [{ json: { total: items.length } }];
```

---

### Returning Array in Each-Item Mode

```javascript
// WRONG — each-item mode expects a single object
return [{ json: { processed: true } }];
```

```javascript
// CORRECT — return single object without array
return { json: { processed: true } };
```

---

## Python-Specific Mistakes

### Dot Notation for Item Access

```python
# WRONG — Python dict does not support dot notation
name = _item.json.name
# AttributeError: 'dict' object has no attribute 'json'
```

```python
# CORRECT — ALWAYS use bracket notation in Python
name = _item["json"]["name"]
```

---

### Using `len()` vs `.length`

```python
# WRONG — JavaScript syntax in Python
count = _items.length
```

```python
# CORRECT — use Python's len()
count = len(_items)
```

---

### Importing External Libraries on Cloud

```python
# WRONG — external imports fail on n8n Cloud
import pandas as pd
import requests
```

```python
# CORRECT — use only standard library on n8n Cloud
import json
import re
from datetime import datetime
```

---

### Boolean Values

```python
# WRONG — JavaScript booleans in Python
return {"json": {"active": true, "deleted": false}}
# NameError: name 'true' is not defined
```

```python
# CORRECT — Python booleans are capitalized
return {"json": {"active": True, "deleted": False}}
```

---

## Variable Scope Mistakes

### Using Expression-Only Variables

```javascript
// WRONG — these are expression-only, NOT available in Code node
const idx = $itemIndex;      // undefined
const secret = $secrets.key;  // undefined
const param = $parameter.x;   // undefined
```

```javascript
// CORRECT — use Code node alternatives
// For index: use loop counter
// For secrets: pass through input items
// For parameters: hardcode or pass through input
```

---

### Accessing `$json` in All-Items Mode

```javascript
// WRONG — $json refers to single-item context
// In "Run Once for All Items" mode, $json is NOT the primary access
const name = $json.name; // may be undefined or refer to wrong item
```

```javascript
// CORRECT — iterate over items array
for (const item of items) {
  const name = item.json.name;
}
```

---

## Binary Data Mistakes

### Direct Binary Buffer Access

```javascript
// WRONG — deprecated, may return incomplete data
const data = items[0].binary.data.data;
```

```javascript
// CORRECT — ALWAYS use getBinaryDataBuffer
const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');
```

---

### Binary Data in Python

```python
# WRONG — getBinaryDataBuffer is NOT supported in Python
buffer = this.helpers.getBinaryDataBuffer(0, 'data')
```

```python
# CORRECT — for binary data processing, use JavaScript Code node
# or use dedicated binary nodes (Convert to File, Extract From File)
# Python Code node cannot handle binary buffers
```

---

## Performance Anti-Patterns

### Using Code Node for Simple Operations

```javascript
// WRONG — using Code node for what native nodes do better
// Filtering:
return items.filter(item => item.json.active);
// Renaming fields:
return items.map(item => ({
  json: { userName: item.json.name }
}));
```

```
CORRECT — use native nodes instead:
- Filter node → for filtering items
- Edit Fields (Set) node → for renaming/adding fields
- Sort node → for sorting
- Remove Duplicates node → for deduplication

Native nodes are faster and maintain automatic item linking.
```

---

### Date Handling with `new Date()`

```javascript
// WRONG — new Date() ignores workflow timezone
const now = new Date();
const formatted = now.toISOString();
```

```javascript
// CORRECT — use Luxon via $now (respects workflow timezone)
const now = $now;
const formatted = $now.toISO();
const custom = $now.toFormat('yyyy-MM-dd HH:mm:ss');
```

---

## Static Data Mistakes

### Relying on Static Data During Testing

```javascript
// WRONG — static data does not persist in manual test runs
const staticData = $getWorkflowStaticData('global');
staticData.counter = (staticData.counter || 0) + 1;
// Counter resets every test run — never increments
```

```javascript
// CORRECT — static data only works in active workflows with triggers
// For testing: use hardcoded test values
// For production: static data works as expected when workflow is active
const staticData = $getWorkflowStaticData('global');
if ($execution.mode === 'test') {
  // Use fallback for testing
  return [{ json: { counter: 'N/A (test mode)' } }];
}
staticData.counter = (staticData.counter || 0) + 1;
return [{ json: { counter: staticData.counter } }];
```

---

### Storing Large Data in Static Data

```javascript
// WRONG — static data is not designed for large payloads
const staticData = $getWorkflowStaticData('global');
staticData.allRecords = items.map(i => i.json); // Could be thousands of records
```

```javascript
// CORRECT — store only small tracking data
const staticData = $getWorkflowStaticData('global');
staticData.lastProcessedId = items[items.length - 1].json.id;
staticData.totalProcessed = (staticData.totalProcessed || 0) + items.length;
// Use a database for large data storage
```
