# n8n Anti-Pattern Catalog

> Consolidated from all skill areas. Each anti-pattern has an ID, category, description, and correction.

---

## Expression Anti-Patterns

### AP-E001: Using `new Date()` Instead of Luxon

**Category**: Expressions / DateTime
**Severity**: High

NEVER use `new Date()` in n8n expressions or Code nodes. It ignores the workflow timezone setting.

- **Wrong**: `{{ new Date().toISOString() }}`
- **Correct**: `{{ $now.toISO() }}` or `{{ DateTime.fromISO("2024-01-01") }}`

### AP-E002: Reversed JMESPath Arguments

**Category**: Expressions / JMESPath
**Severity**: High

NEVER pass the search string as the first argument to `$jmespath`. n8n uses `(object, searchString)` order, which differs from the JMESPath spec's `search(expression, data)`.

- **Wrong**: `{{ $jmespath("[*].name", $json.items) }}`
- **Correct**: `{{ $jmespath($json.items, "[*].name") }}`

### AP-E003: Referencing Non-Existent Node Names

**Category**: Expressions / Node References
**Severity**: Critical

NEVER use `$("<NodeName>")` with a node name that does not exist in the workflow. This causes a runtime error.

- **Wrong**: `{{ $("HTTP Request 1").item.json.data }}` (node was renamed)
- **Correct**: Verify the exact node name in the workflow before referencing

### AP-E004: Using Moment.js Format Tokens

**Category**: Expressions / DateTime
**Severity**: Medium

NEVER use Moment.js format tokens (e.g., `YYYY-MM-DD`) with Luxon DateTime objects. Luxon uses different tokens.

- **Wrong**: `{{ $now.format("YYYY-MM-DD") }}`
- **Correct**: `{{ $now.format("yyyy-MM-dd") }}`

### AP-E005: Assuming $vars Returns Non-String Types

**Category**: Expressions / Variables
**Severity**: Medium

NEVER assume `$vars` values are anything other than strings. All n8n variables are stored as strings.

- **Wrong**: `{{ $vars.maxRetries > 3 }}` (string comparison)
- **Correct**: `{{ parseInt($vars.maxRetries) > 3 }}`

### AP-E006: Using $input.all Without Parentheses

**Category**: Expressions / Method Calls
**Severity**: Medium

NEVER reference `$input.all` without calling it as a function. It returns the function itself, not the items.

- **Wrong**: `{{ $input.all }}`
- **Correct**: `{{ $input.all() }}`

---

## Code Node Anti-Patterns

### AP-C001: Using $itemIndex in Code Node

**Category**: Code Node / Variables
**Severity**: Critical

NEVER use `$itemIndex` in a Code node — it is not available and causes a ReferenceError.

- **Wrong**: `const idx = $itemIndex;`
- **Correct**: Use the loop variable `i` or `$input.item` index

### AP-C002: Using $secrets in Code Node

**Category**: Code Node / Variables
**Severity**: Critical

NEVER use `$secrets` in a Code node — it is not available. Use `$env` or credential references instead.

- **Wrong**: `const key = $secrets.provider.apiKey;`
- **Correct**: `const key = $env.API_KEY;` or use credential node

### AP-C003: Missing JSON Wrapper in Return

**Category**: Code Node / Return Format
**Severity**: Critical

NEVER return items without the `json` key wrapper. Every item MUST have `{ json: { ... } }`.

- **Wrong**: `return [{ name: "test" }];`
- **Correct**: `return [{ json: { name: "test" } }];`

### AP-C004: Direct Binary Data Access

**Category**: Code Node / Binary Data
**Severity**: High

NEVER access binary data directly via `items[0].binary.data.data`. ALWAYS use the helper method.

- **Wrong**: `const buffer = Buffer.from(items[0].binary.data.data, 'base64');`
- **Correct**: `const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');`

### AP-C005: Making HTTP Requests in Code Node

**Category**: Code Node / Architecture
**Severity**: High

NEVER make HTTP requests inside a Code node. Use the HTTP Request node instead. The Code node sandbox restricts network access.

- **Wrong**: `const res = await fetch('https://api.example.com/data');`
- **Correct**: Use HTTP Request node before the Code node

### AP-C006: File System Access in Code Node

**Category**: Code Node / Architecture
**Severity**: High

NEVER access the file system in a Code node. Use Read/Write Files from Disk nodes instead.

- **Wrong**: `const fs = require('fs'); const data = fs.readFileSync('/path');`
- **Correct**: Use Read Binary File or Read/Write Files From Disk node

### AP-C007: Python Dot Notation on Items

**Category**: Code Node / Python
**Severity**: Critical

NEVER use dot notation to access item properties in Python Code nodes. Python dictionaries require bracket notation.

- **Wrong**: `name = item.json.name`
- **Correct**: `name = item["json"]["name"]`

### AP-C008: Python items.length

**Category**: Code Node / Python
**Severity**: Medium

NEVER use `.length` in Python — it is JavaScript syntax. Use Python's `len()`.

- **Wrong**: `count = items.length`
- **Correct**: `count = len(items)`

### AP-C009: External Imports on n8n Cloud (Python)

**Category**: Code Node / Python
**Severity**: High

NEVER import third-party Python libraries on n8n Cloud. Only standard library modules are available.

- **Wrong**: `import pandas as pd`
- **Correct**: Use standard library or native n8n nodes

### AP-C010: Using getBinaryDataBuffer in Python

**Category**: Code Node / Python
**Severity**: Critical

NEVER call `getBinaryDataBuffer()` in a Python Code node — it is NOT supported.

- **Wrong**: `buffer = self.helpers.getBinaryDataBuffer(0, 'data')`
- **Correct**: Handle binary data in a JavaScript Code node instead

---

## Credential Anti-Patterns

### AP-CR001: Hardcoded API Keys in Node Parameters

**Category**: Credentials / Security
**Severity**: Critical

NEVER put API keys, tokens, or passwords directly in node parameter values. They appear in workflow JSON, execution logs, and exports.

- **Wrong**: `"Authorization": "Bearer sk-1234567890abcdef"` in header parameters
- **Correct**: Use credential system with `httpHeaderAuth` or custom credential type

### AP-CR002: Missing authenticate Property

**Category**: Credentials / Configuration
**Severity**: Critical

NEVER create an `ICredentialType` without an `authenticate` property. Without it, credentials are never injected into requests.

- **Wrong**: Credential type with only `name`, `displayName`, `properties`
- **Correct**: Include `authenticate: { type: 'generic', properties: { ... } }`

### AP-CR003: Wrong authenticate Type

**Category**: Credentials / Configuration
**Severity**: High

NEVER use a value other than `'generic'` for `authenticate.type`. It is the only supported type.

- **Wrong**: `authenticate: { type: 'bearer', ... }`
- **Correct**: `authenticate: { type: 'generic', properties: { headers: { ... } } }`

### AP-CR004: Missing Credential Test Endpoint

**Category**: Credentials / Usability
**Severity**: Medium

ALWAYS include a `test` property on credential types. Without it, users cannot validate credentials in the UI.

- **Wrong**: No `test` property
- **Correct**: `test: { request: { baseURL: '...', url: '/me' } }`

### AP-CR005: Plain Text Password Fields

**Category**: Credentials / Security
**Severity**: High

NEVER define sensitive credential fields without `typeOptions: { password: true }`. They display as plain text.

- **Wrong**: `{ name: 'apiKey', type: 'string', default: '' }`
- **Correct**: `{ name: 'apiKey', type: 'string', typeOptions: { password: true }, default: '' }`

### AP-CR006: Missing Credential Expression Prefix

**Category**: Credentials / Syntax
**Severity**: Critical

NEVER omit the `=` prefix in credential authenticate expressions. Without it, the expression is treated as a literal string.

- **Wrong**: `'Bearer {{$credentials.apiKey}}'`
- **Correct**: `'=Bearer {{$credentials.apiKey}}'`

---

## Node Type Anti-Patterns

### AP-N001: Single Array Return from execute()

**Category**: Node Type / Return Format
**Severity**: Critical

NEVER return a single-dimensional array from `execute()`. The return type is `INodeExecutionData[][]` — items MUST be wrapped in an outer array.

- **Wrong**: `return returnData;`
- **Correct**: `return [returnData];`

### AP-N002: Mutating Input Data

**Category**: Node Type / Data Integrity
**Severity**: High

NEVER modify items returned by `this.getInputData()`. They are shared references. Always create new objects.

- **Wrong**: `items[i].json.newField = 'value';`
- **Correct**: `returnData.push({ json: { ...items[i].json, newField: 'value' } });`

### AP-N003: Missing pairedItem in Output

**Category**: Node Type / Item Linking
**Severity**: Medium

ALWAYS include `pairedItem: { item: i }` on output items to maintain item linking across nodes.

- **Wrong**: `returnData.push({ json: result });`
- **Correct**: `returnData.push({ json: result, pairedItem: { item: i } });`

### AP-N004: Trigger Node with Inputs

**Category**: Node Type / Trigger
**Severity**: High

NEVER define `inputs` on a trigger node. Trigger nodes are workflow entry points and MUST have `inputs: []`.

- **Wrong**: `inputs: [NodeConnectionTypes.Main]` on a trigger
- **Correct**: `inputs: []`

### AP-N005: Missing Manual Trigger Function

**Category**: Node Type / Trigger
**Severity**: High

ALWAYS provide a `manualTriggerFunction` in the `ITriggerResponse` for manual execution mode. Without it, clicking "Test" hangs indefinitely.

- **Wrong**: Only registering event listeners, no manual path
- **Correct**: Check `this.getMode() === 'manual'` and return `{ manualTriggerFunction }`

### AP-N006: Processing Only First Item

**Category**: Node Type / Data Processing
**Severity**: High

NEVER process only `items[0]` when the node should handle all items. ALWAYS iterate the full input array.

- **Wrong**: `const value = this.getNodeParameter('field', 0);` (only index 0)
- **Correct**: `for (let i = 0; i < items.length; i++) { getNodeParameter('field', i); }`

### AP-N007: package.json Points to TypeScript Files

**Category**: Node Type / Registration
**Severity**: Critical

NEVER list `.ts` files in `package.json`'s `n8n.nodes` or `n8n.credentials`. n8n loads compiled JavaScript.

- **Wrong**: `"nodes": ["nodes/MyNode.node.ts"]`
- **Correct**: `"nodes": ["dist/nodes/MyNode.node.js"]`

### AP-N008: Wrong Error Class

**Category**: Node Type / Error Handling
**Severity**: Medium

ALWAYS use `NodeApiError` for external service failures and `NodeOperationError` for internal/validation errors. Using the wrong class produces misleading error messages.

- **Wrong**: `throw new NodeApiError(...)` for a validation error
- **Correct**: `throw new NodeOperationError(this.getNode(), 'Invalid input', { itemIndex: i })`

---

## Deployment Anti-Patterns

### AP-D001: Auto-Generated Encryption Key

**Category**: Deployment / Security
**Severity**: Critical

NEVER rely on the auto-generated encryption key. If the container is rebuilt or the `.n8n` directory is lost, all encrypted credentials become permanently inaccessible.

- **Wrong**: No `N8N_ENCRYPTION_KEY` in environment
- **Correct**: Explicitly set and securely back up `N8N_ENCRYPTION_KEY`

### AP-D002: SQLite in Production

**Category**: Deployment / Database
**Severity**: High

NEVER use SQLite for production n8n deployments. It does not support concurrent access, queue mode, or multi-instance scaling.

- **Wrong**: `DB_TYPE=sqlite` (or default) in production
- **Correct**: `DB_TYPE=postgresdb` with PostgreSQL 13+

### AP-D003: SQLite with Queue Mode

**Category**: Deployment / Queue
**Severity**: Critical

NEVER use SQLite when `EXECUTIONS_MODE=queue`. Queue mode requires PostgreSQL for concurrent worker access.

- **Wrong**: `EXECUTIONS_MODE=queue` without `DB_TYPE=postgresdb`
- **Correct**: PostgreSQL + Redis + queue mode

### AP-D004: Ephemeral Container Storage

**Category**: Deployment / Persistence
**Severity**: Critical

NEVER run n8n without a persistent volume for `/home/node/.n8n`. Container restarts destroy the database, encryption key, and all configuration.

- **Wrong**: No volume mount for `/home/node/.n8n`
- **Correct**: `n8n_data:/home/node/.n8n` with named Docker volume

### AP-D005: HTTP in Production

**Category**: Deployment / Security
**Severity**: Critical

NEVER run n8n over HTTP in production. Credentials and session tokens are transmitted in cleartext.

- **Wrong**: `N8N_PROTOCOL=http` or no HTTPS termination
- **Correct**: TLS via reverse proxy (Traefik/nginx/Caddy) or `N8N_SSL_KEY`/`N8N_SSL_CERT`

### AP-D006: Missing WEBHOOK_URL

**Category**: Deployment / Webhooks
**Severity**: High

NEVER omit `WEBHOOK_URL` in production. Without it, webhook URLs may resolve to internal addresses unreachable from outside.

- **Wrong**: No `WEBHOOK_URL` set
- **Correct**: `WEBHOOK_URL=https://n8n.example.com/`

### AP-D007: Different n8n Versions in Queue Mode

**Category**: Deployment / Queue
**Severity**: High

NEVER run different n8n versions across main and worker instances. Serialization formats may differ between versions.

- **Wrong**: Main on v1.80.0, workers on v1.75.0
- **Correct**: All instances use identical Docker image tag

### AP-D008: Missing Task Runners

**Category**: Deployment / Security
**Severity**: High

NEVER run without task runners enabled in production. Without them, Code node JavaScript executes in the main n8n process without sandboxing.

- **Wrong**: No `N8N_RUNNERS_ENABLED` or set to `false`
- **Correct**: `N8N_RUNNERS_ENABLED=true`

### AP-D009: No Execution Pruning

**Category**: Deployment / Maintenance
**Severity**: Medium

NEVER run production n8n without execution data pruning. The database grows unbounded and eventually degrades performance.

- **Wrong**: `EXECUTIONS_DATA_PRUNE=false` or not configured
- **Correct**: `EXECUTIONS_DATA_PRUNE=true` with `EXECUTIONS_DATA_MAX_AGE` and `EXECUTIONS_DATA_PRUNE_MAX_COUNT`

### AP-D010: Missing Timezone Configuration

**Category**: Deployment / Configuration
**Severity**: Medium

NEVER omit timezone configuration. Schedule triggers and date expressions use the system default, which may differ between environments.

- **Wrong**: No `GENERIC_TIMEZONE` or `TZ` set
- **Correct**: Both `GENERIC_TIMEZONE` and `TZ` set to the same value (e.g., `Europe/Amsterdam`)

---

## Workflow Anti-Patterns

### AP-W001: No Error Workflow Configured

**Category**: Workflow / Error Handling
**Severity**: Critical

NEVER activate a production workflow without `settings.errorWorkflow`. Failed executions produce no alerts.

- **Wrong**: Empty or missing `errorWorkflow` setting
- **Correct**: Set to a valid Error Trigger workflow ID

### AP-W002: Duplicate Node Names

**Category**: Workflow / JSON Structure
**Severity**: Critical

NEVER allow two nodes in a workflow to have the same `name`. Connections reference nodes by name — duplicates make wiring ambiguous.

- **Wrong**: Two nodes both named "HTTP Request"
- **Correct**: Descriptive unique names like "Fetch Users", "Fetch Orders"

### AP-W003: Orphan Nodes

**Category**: Workflow / Topology
**Severity**: Medium

NEVER leave non-trigger nodes disconnected from the execution path. They consume resources but produce no output.

- **Wrong**: Nodes with no incoming or outgoing connections
- **Correct**: Connect or remove unused nodes

### AP-W004: Using Static Data During Testing

**Category**: Workflow / Development
**Severity**: Medium

NEVER rely on `$getWorkflowStaticData()` during manual test executions. Static data is only persisted for active workflows triggered automatically.

- **Wrong**: Testing static data logic with "Execute Workflow" button
- **Correct**: Activate workflow and trigger via webhook/schedule to test static data

### AP-W005: Large Static Data Storage

**Category**: Workflow / Performance
**Severity**: Medium

NEVER store large amounts of data in workflow static data. It is loaded into memory on every execution and stored in the database.

- **Wrong**: Storing thousands of processed IDs in static data
- **Correct**: Use an external database for large state tracking

### AP-W006: Unauthenticated Production Webhooks

**Category**: Workflow / Security
**Severity**: High

NEVER expose production webhook endpoints without authentication. Any internet user can trigger the workflow.

- **Wrong**: Webhook node with authentication set to "None"
- **Correct**: Use Basic Auth, Header Auth, or JWT authentication on production webhooks
