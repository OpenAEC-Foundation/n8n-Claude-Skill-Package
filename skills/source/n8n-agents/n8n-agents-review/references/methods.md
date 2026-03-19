# n8n Review Validation Methods

> Complete checklist organized by area. Each check specifies what to verify, the expected state, and the common failure mode.

---

## 1. Workflow JSON Validation

### 1.1 Root Structure

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 1.1.1 | `id` field exists | Non-empty string | Missing on manually created JSON |
| 1.1.2 | `name` field exists | Non-empty string | Empty string |
| 1.1.3 | `active` field exists | Boolean value | String `"true"` instead of `true` |
| 1.1.4 | `nodes` field exists | Array (can be empty) | Missing entirely |
| 1.1.5 | `connections` field exists | Object (can be empty) | Missing entirely |
| 1.1.6 | `settings.executionOrder` | `"v1"` for new workflows | Using deprecated `"v0"` |

### 1.2 Node Objects

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 1.2.1 | `id` is unique UUID | No duplicates across `nodes[]` | Copy-pasted nodes with same ID |
| 1.2.2 | `name` is unique | No duplicates across `nodes[]` | Duplicate names break connections |
| 1.2.3 | `type` is valid format | `n8n-nodes-base.<name>` or `n8n-nodes-<pkg>.<name>` | Typo in type name |
| 1.2.4 | `typeVersion` is number | Positive number matching valid version | Version 0 or negative |
| 1.2.5 | `position` is `[x, y]` | Array of exactly 2 numbers | Missing or wrong array length |
| 1.2.6 | `parameters` exists | Object (can be empty) | `null` or missing |
| 1.2.7 | `credentials` format | Object with credential type keys | Array instead of object |

### 1.3 Workflow Settings

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 1.3.1 | `errorWorkflow` set | Valid workflow ID string | Empty or missing in production |
| 1.3.2 | `timezone` set | Valid IANA timezone or `"DEFAULT"` | Invalid timezone string |
| 1.3.3 | `executionTimeout` | Positive number (seconds) | Zero or negative |
| 1.3.4 | `callerPolicy` | Valid policy for sub-workflows | Missing when used as sub-workflow |

---

## 2. Connection Wiring

### 2.1 Structure Validation

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 2.1.1 | Source node exists | Every key in `connections` matches a node name | Renamed node, stale connection key |
| 2.1.2 | Destination node exists | Every `IConnection.node` matches a node name | Deleted node still referenced |
| 2.1.3 | Connection type valid | `"main"` or valid `NodeConnectionType` | Typo or wrong AI connection type |
| 2.1.4 | Destination index valid | `index` within target node input count | Index exceeds available inputs |
| 2.1.5 | No self-connections | Source node differs from destination | Node connected to itself |

### 2.2 Topology Validation

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 2.2.1 | Trigger has no inputs | Nodes with `group: ['trigger']` have no incoming connections | Wiring into trigger node |
| 2.2.2 | Exactly one trigger | Production workflow has one trigger node | Zero or multiple triggers |
| 2.2.3 | No unreachable nodes | All non-trigger nodes reachable from trigger | Orphaned node cluster |
| 2.2.4 | IF outputs correct | IF node `connections.main` has 2 arrays | Missing true or false branch |
| 2.2.5 | Switch outputs correct | Switch node outputs match defined cases | Fewer arrays than cases |

### 2.3 AI Connection Validation

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 2.3.1 | AI connection types | `ai_languageModel`, `ai_memory`, `ai_tool`, etc. | Using `"main"` for AI sub-nodes |
| 2.3.2 | Agent has model | AI Agent node has `ai_languageModel` connection | Missing language model |
| 2.3.3 | Embedding with vector store | Vector store has embedding model connected | Vector store without embeddings |

---

## 3. Expression Validation

### 3.1 Variable Reference Checks

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 3.1.1 | `$json` references valid | Fields exist in upstream node output | Referencing non-existent field |
| 3.1.2 | `$("<NodeName>")` valid | Node name exists in workflow | Typo in node name |
| 3.1.3 | `$input.all()` context | Used in expression, not Code node raw | Calling without parentheses |
| 3.1.4 | `$itemIndex` context | NEVER in Code node | Used in Code node (undefined) |
| 3.1.5 | `$secrets` context | NEVER in Code node | Used in Code node (undefined) |
| 3.1.6 | `$env` access | Returns configured env vars | Assuming env var exists |
| 3.1.7 | `$vars` access | Returns string values | Expecting non-string type |

### 3.2 JMESPath Validation

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 3.2.1 | Parameter order | `$jmespath(object, searchString)` | Reversed: search string first |
| 3.2.2 | Python variant | `_jmespath(object, searchString)` | Using `$jmespath` in Python |
| 3.2.3 | Valid JMESPath syntax | Proper projections, filters | Invalid filter expression |

### 3.3 DateTime Validation

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 3.3.1 | Use Luxon for dates | `$now`, `$today`, `DateTime.fromISO()` | Using `new Date()` |
| 3.3.2 | Format tokens correct | Luxon tokens (e.g., `yyyy-MM-dd`) | Moment.js tokens (`YYYY-MM-DD`) |
| 3.3.3 | Timezone handling | Explicit timezone via `.setZone()` | Assuming UTC |

---

## 4. Credential Validation

### 4.1 ICredentialType Completeness

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 4.1.1 | `name` defined | Unique string identifier | Missing or duplicating another credential |
| 4.1.2 | `displayName` defined | Human-readable string | Missing |
| 4.1.3 | `properties` defined | Non-empty `INodeProperties[]` | Empty array |
| 4.1.4 | Password fields use `typeOptions` | `{ password: true }` on sensitive fields | Plain text display for secrets |
| 4.1.5 | `documentationUrl` set | Valid URL | Missing documentation reference |

### 4.2 Authentication Method

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 4.2.1 | `authenticate` exists | Object with `type` and `properties` | Missing entirely |
| 4.2.2 | `type` is `'generic'` | Exactly `'generic'` | Wrong type value |
| 4.2.3 | Credential expressions | Use `={{$credentials.fieldName}}` | Missing `=` prefix or wrong path |
| 4.2.4 | Injection target valid | `headers`, `qs`, `body`, or `auth` | Wrong property name |

### 4.3 Credential Testing

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 4.3.1 | `test` property exists | `ICredentialTestRequest` object | Missing test endpoint |
| 4.3.2 | Test URL valid | Lightweight GET endpoint | Heavy or destructive endpoint |
| 4.3.3 | Test uses credentials | Inherits authenticate config | Hardcoded auth in test |

### 4.4 Registration

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 4.4.1 | package.json `n8n.credentials` | Points to compiled `.js` file in `dist/` | Pointing to `.ts` source |
| 4.4.2 | Credential name matches | Node's `credentials[].name` matches `ICredentialType.name` | Name mismatch |

---

## 5. Node Type Validation

### 5.1 INodeTypeDescription

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 5.1.1 | `displayName` defined | Non-empty string | Missing |
| 5.1.2 | `name` defined | Camel-case string | Spaces or special characters |
| 5.1.3 | `group` defined | Array of valid group types | Empty array |
| 5.1.4 | `version` defined | Number or number array | Missing or zero |
| 5.1.5 | `inputs` defined | Array of `NodeConnectionType` or expression | Missing |
| 5.1.6 | `outputs` defined | Array of `NodeConnectionType` or expression | Missing |
| 5.1.7 | `defaults.name` defined | Default display name | Missing defaults |
| 5.1.8 | `properties` defined | Array of `INodeProperties[]` | Missing |
| 5.1.9 | `description` defined | Short description string | Missing |

### 5.2 Execute Method

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 5.2.1 | Return type correct | `Promise<INodeExecutionData[][]>` | Single array instead of double |
| 5.2.2 | Items have `json` key | Every item: `{ json: { ... } }` | Missing `json` wrapper |
| 5.2.3 | Item processing loop | Iterates `getInputData()` items | Processing only first item |
| 5.2.4 | Parameter access indexed | `getNodeParameter('name', i)` with item index | Missing index parameter |
| 5.2.5 | Input data not mutated | Clone before modifying | Mutating `getInputData()` directly |

### 5.3 Trigger Nodes

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 5.3.1 | `inputs` is empty | `inputs: []` | Having input connections |
| 5.3.2 | `group` includes trigger | `['trigger']` or `['trigger', 'schedule']` | Missing trigger group |
| 5.3.3 | Returns `ITriggerResponse` | Object with optional `closeFunction` | Wrong return type |
| 5.3.4 | Manual mode handled | `manualTriggerFunction` provided | Trigger hangs in manual test |
| 5.3.5 | `emit()` data format | `this.emit([[items]])` — double-wrapped | Single-wrapped array |

### 5.4 Property Definitions

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 5.4.1 | `type` is valid | One of `NodePropertyTypes` union values | Invalid type string |
| 5.4.2 | `default` matches type | Default value compatible with declared type | String default for number type |
| 5.4.3 | `options` for dropdowns | Present when `type` is `'options'` or `'multiOptions'` | Missing options array |
| 5.4.4 | `displayOptions` valid | Referenced properties exist | Referencing non-existent property |
| 5.4.5 | `noDataExpression` on selectors | `true` for resource/operation fields | Allowing expressions on static selectors |

---

## 6. Error Handling

### 6.1 Workflow-Level

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 6.1.1 | Error workflow set | `settings.errorWorkflow` points to valid workflow | Missing in production |
| 6.1.2 | Error workflow exists | Referenced workflow ID is accessible | Deleted or inaccessible workflow |
| 6.1.3 | Error trigger present | Error workflow starts with Error Trigger node | Missing trigger node |

### 6.2 Node-Level

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 6.2.1 | continueOnFail pattern | Check + error output + continue | Missing error data in output |
| 6.2.2 | pairedItem in errors | `pairedItem: { item: i }` on error items | Missing item linking |
| 6.2.3 | Retry on HTTP nodes | `retryOnFail: true` on API calls | No retry on transient failures |
| 6.2.4 | Retry config reasonable | `maxTries <= 5`, `waitBetweenTries >= 1000` | Aggressive retry (no backoff) |
| 6.2.5 | Error type correct | `NodeApiError` for API, `NodeOperationError` for logic | Wrong error class |
| 6.2.6 | itemIndex in errors | `{ itemIndex: i }` in error options | Missing index for batch errors |

---

## 7. Deployment Validation

### 7.1 Core Requirements

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 7.1.1 | Encryption key explicit | `N8N_ENCRYPTION_KEY` set as env var | Auto-generated (lost on redeploy) |
| 7.1.2 | Production mode | `NODE_ENV=production` | Missing or `development` |
| 7.1.3 | HTTPS configured | `N8N_PROTOCOL=https` with TLS | HTTP in production |
| 7.1.4 | Webhook URL set | `WEBHOOK_URL` = public HTTPS URL | Localhost or missing |
| 7.1.5 | Data volume mounted | `/home/node/.n8n` on persistent volume | Ephemeral container storage |
| 7.1.6 | Settings permissions | `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true` | Readable by others |
| 7.1.7 | Task runners enabled | `N8N_RUNNERS_ENABLED=true` | Code runs unsandboxed |

### 7.2 Queue Mode

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 7.2.1 | PostgreSQL database | `DB_TYPE=postgresdb` (NOT sqlite) | SQLite with queue mode |
| 7.2.2 | Queue mode enabled | `EXECUTIONS_MODE=queue` | Missing on workers |
| 7.2.3 | Redis configured | `QUEUE_BULL_REDIS_HOST` and port set | Workers cannot find broker |
| 7.2.4 | Shared encryption key | Same `N8N_ENCRYPTION_KEY` on all instances | Credential decryption fails |
| 7.2.5 | S3 binary storage | Binary data mode set to `s3` | Binary data not shared |
| 7.2.6 | Same n8n version | All instances run identical version | Incompatible serialization |

### 7.3 Docker Specific

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 7.3.1 | Named volume used | `n8n_data:/home/node/.n8n` | Bind mount with wrong permissions |
| 7.3.2 | Timezone configured | `GENERIC_TIMEZONE` and `TZ` set | Schedule triggers use wrong time |
| 7.3.3 | Restart policy | `restart: always` or `unless-stopped` | Container stays down after crash |
| 7.3.4 | Image source | `docker.n8n.io/n8nio/n8n` | Third-party image |

---

## 8. Code Node Validation

### 8.1 JavaScript

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 8.1.1 | Return format (all) | `return [{json: {...}}, ...]` | Missing `json` wrapper |
| 8.1.2 | Return format (each) | `return {json: {...}}` | Returning raw object |
| 8.1.3 | No `$itemIndex` | Variable not referenced | ReferenceError at runtime |
| 8.1.4 | No `$secrets` | Variable not referenced | Undefined value |
| 8.1.5 | No HTTP requests | No fetch/axios/http calls | Use HTTP Request node |
| 8.1.6 | No fs access | No `require('fs')` or file operations | Use Read/Write Files nodes |
| 8.1.7 | Binary via helper | `this.helpers.getBinaryDataBuffer()` | Direct `items[0].binary.data.data` |
| 8.1.8 | Luxon for dates | `DateTime.fromISO()`, `$now` | `new Date()` |

### 8.2 Python

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 8.2.1 | Bracket notation | `item["json"]["field"]` | Dot notation `item.json.field` |
| 8.2.2 | `_` prefix variables | `_items`, `_json`, `_execution` | Using `$` prefix |
| 8.2.3 | No external imports (Cloud) | Only standard library on Cloud | Third-party imports fail |
| 8.2.4 | `len()` for length | `len(items)` | `items.length` (JS syntax) |
| 8.2.5 | No `getBinaryDataBuffer` | Not supported in Python | Runtime error |

---

## 9. Security Validation

| # | Check | Expected State | Common Failure |
|---|-------|---------------|----------------|
| 9.1 | No hardcoded API keys | Credentials via credential system | Keys in parameter values |
| 9.2 | No secrets in Code node | Use `$env` or credential references | Inline passwords |
| 9.3 | Encryption key backed up | Key stored outside container | Lost with container rebuild |
| 9.4 | Task runners enabled | `N8N_RUNNERS_ENABLED=true` | Code executes in main process |
| 9.5 | File access restricted | `N8N_RESTRICT_FILE_ACCESS_TO` set | Unrestricted filesystem |
| 9.6 | Env access controlled | `N8N_BLOCK_ENV_ACCESS_IN_NODE` if needed | Secrets leaked via `$env` |
| 9.7 | Webhook auth enabled | Authentication on production webhooks | Public webhooks without auth |
| 9.8 | HTTPS enforced | TLS termination configured | Credentials in plaintext |
| 9.9 | Secure cookies | `N8N_SECURE_COOKIE=true` | Session hijacking risk |
| 9.10 | Execution pruning | `EXECUTIONS_DATA_PRUNE=true` | Database grows unbounded |
