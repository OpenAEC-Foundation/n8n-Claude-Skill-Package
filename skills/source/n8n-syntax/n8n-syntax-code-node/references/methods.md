# Code Node — Variables, Modules & Return Format Reference

## Variables by Execution Mode

### Run Once for All Items

#### JavaScript

| Variable | Type | Description |
|----------|------|-------------|
| `items` | `Array<{json: object, binary?: object}>` | All input items — primary access point |
| `$input.all()` | `Array<Item>` | All input items (alternative to `items`) |
| `$input.first()` | `Item` | First input item |
| `$input.last()` | `Item` | Last input item |

#### Python

| Variable | Type | Description |
|----------|------|-------------|
| `_items` | `list[dict]` | All input items — primary access point |
| `_input.all()` | `list[dict]` | All input items (alternative) |
| `_input.first()` | `dict` | First input item |
| `_input.last()` | `dict` | Last input item |

### Run Once for Each Item

#### JavaScript

| Variable | Type | Description |
|----------|------|-------------|
| `$input.item` | `{json: object, binary?: object}` | Current item being processed |
| `$json` | `object` | Shorthand for `$input.item.json` |
| `$binary` | `object` | Shorthand for `$input.item.binary` |

#### Python

| Variable | Type | Description |
|----------|------|-------------|
| `_item` | `dict` | Current item: `_item["json"]` for data |

### Available in Both Modes

#### JavaScript

| Variable | Returns | Description |
|----------|---------|-------------|
| `$("<node>").all(branchIndex?, runIndex?)` | `Array<Item>` | All items from named node output |
| `$("<node>").first(branchIndex?, runIndex?)` | `Item` | First item from named node |
| `$("<node>").last(branchIndex?, runIndex?)` | `Item` | Last item from named node |
| `$("<node>").itemMatching(index)` | `Item` | Trace back to matching item in named node |
| `$("<node>").isExecuted` | `boolean` | Whether named node has executed |
| `$execution.id` | `string` | Unique execution ID |
| `$execution.mode` | `string` | `"test"`, `"production"`, or `"evaluation"` |
| `$execution.resumeUrl` | `string` | Webhook URL to resume at Wait node |
| `$execution.customData.set(key, val)` | `void` | Set custom execution data |
| `$execution.customData.setAll(obj)` | `void` | Set multiple custom data pairs |
| `$execution.customData.get(key)` | `any` | Get custom execution data |
| `$execution.customData.getAll()` | `object` | Get all custom execution data |
| `$workflow.id` | `string` | Workflow ID |
| `$workflow.name` | `string` | Workflow name |
| `$workflow.active` | `boolean` | Whether workflow is active |
| `$now` | `DateTime` | Current moment (Luxon), respects workflow timezone |
| `$today` | `DateTime` | Midnight today (Luxon), respects instance timezone |
| `$env` | `object` | n8n instance environment variables |
| `$vars` | `object` | User-defined variables (all values are strings) |
| `$prevNode.name` | `string` | Name of previous node |
| `$prevNode.outputIndex` | `number` | Output connector index from previous node |
| `$prevNode.runIndex` | `number` | Run index of previous node |
| `$runIndex` | `number` | How many times current node executed (zero-based) |
| `$nodeVersion` | `number` | Current node version |
| `$jmespath(obj, query)` | `any` | JMESPath query function |
| `$ifEmpty(value, fallback)` | `any` | Returns fallback if value is empty/null/undefined |
| `$getWorkflowStaticData(type)` | `object` | Persistent data — type: `"global"` or `"node"` |

#### Python Equivalents

| Python | JavaScript | Notes |
|--------|-----------|-------|
| `_execution` | `$execution` | Same properties and methods |
| `_workflow` | `$workflow` | Same properties |
| `_now` | `$now` | DateTime object |
| `_today` | `$today` | DateTime object |
| `_env` | `$env` | Environment variables |
| `_vars` | `$vars` | User-defined variables |
| `_prevNode` | `$prevNode` | Previous node info |
| `_jmespath(obj, query)` | `$jmespath(obj, query)` | JMESPath queries |
| `_getWorkflowStaticData(type)` | `$getWorkflowStaticData(type)` | Static data |

### NOT Available in Code Node

These variables are available in expressions but NOT in the Code node:

| Variable | Reason | Workaround |
|----------|--------|------------|
| `$itemIndex` | Not exposed in sandbox | Use loop counter: `items.forEach((item, index) => ...)` |
| `$secrets` | Security restriction | Pass values through preceding node output |
| `$parameter` | Not exposed in sandbox | Hardcode or pass as input data |
| `$response` | HTTP Request node only | Use HTTP Request node, read output via `$("HTTP Request").all()` |
| `$pageCount` | HTTP Request node only | Not applicable in Code node |

## Built-in Modules

### n8n Cloud

| Module | Usage |
|--------|-------|
| `crypto` (Node.js built-in) | `const crypto = require('crypto'); crypto.createHash('sha256').update(data).digest('hex');` |
| `moment` (npm package) | `const moment = require('moment'); moment().format('YYYY-MM-DD');` |
| Luxon | Available via `$now`, `$today`, and `DateTime` global |

### Self-hosted (Additional)

| Module | Configuration |
|--------|--------------|
| External npm modules | Enable via `NODE_FUNCTION_ALLOW_EXTERNAL` environment variable |
| Built-in Node.js modules | Enable via `NODE_FUNCTION_ALLOW_BUILTIN` environment variable |

#### Configuration Examples

```bash
# Allow specific external packages
NODE_FUNCTION_ALLOW_EXTERNAL=lodash,axios

# Allow specific built-in modules
NODE_FUNCTION_ALLOW_BUILTIN=fs,path

# Allow all (NOT recommended for production)
NODE_FUNCTION_ALLOW_EXTERNAL=*
NODE_FUNCTION_ALLOW_BUILTIN=*
```

### Python Modules

| Environment | Available |
|-------------|-----------|
| n8n Cloud | Standard library only — NO external imports |
| Self-hosted | Standard library + allowlisted third-party modules |

## Return Format — Detailed Rules

### Run Once for All Items

MUST return an **array** of objects, each with a `json` key:

```javascript
// JavaScript
return [
  { json: { id: 1, name: "Alice" } },
  { json: { id: 2, name: "Bob" } }
];
```

```python
# Python
return [
  {"json": {"id": 1, "name": "Alice"}},
  {"json": {"id": 2, "name": "Bob"}}
]
```

### Run Once for Each Item

MUST return a **single object** with a `json` key:

```javascript
// JavaScript
return { json: { id: 1, name: "Alice", processed: true } };
```

```python
# Python
return {"json": {"id": 1, "name": "Alice", "processed": True}}
```

### Including Binary Data in Return

```javascript
// JavaScript — return item with both json and binary data
return [{
  json: { fileName: "report.pdf", size: 1024 },
  binary: {
    data: await this.helpers.prepareBinaryData(
      buffer,        // Buffer object
      'report.pdf',  // filename
      'application/pdf'  // MIME type
    )
  }
}];
```

### Returning Empty Output

```javascript
// JavaScript — return empty array to output zero items
return [];
```

```python
# Python — return empty list
return []
```

### Filtering Items via Return

```javascript
// JavaScript — only return items that match a condition
return items.filter(item => item.json.status === 'active').map(item => ({
  json: item.json
}));
```

## Static Data (Persistent Across Executions)

```javascript
// Global — accessible by every node in the workflow
const staticData = $getWorkflowStaticData('global');
staticData.lastRun = new Date().toISOString();
staticData.processedCount = (staticData.processedCount || 0) + items.length;

// Node-specific — only accessible by THIS Code node
const nodeData = $getWorkflowStaticData('node');
nodeData.seenIds = nodeData.seenIds || [];
```

**Limitations:**
- NOT available during manual test runs — requires active workflow with trigger
- Data persists only on successful execution completion
- Keep data small — not designed for large payloads
- May be unreliable during high-frequency concurrent executions
