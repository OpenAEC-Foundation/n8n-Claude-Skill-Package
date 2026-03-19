# n8n Workflow Automation Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-19

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-01 | **Merged** `n8n-core-data-flow` into `n8n-core-architecture` | Item-based data flow IS the core architecture. Splitting forces loading two skills for one concept. Architecture skill covers execution model + data flow + node types. |
| D-02 | **Merged** `n8n-impl-backup-restore` into `n8n-impl-deployment` | CLI export/import is thin (3 commands). Backup is a deployment operations concern. Combined skill covers the full ops lifecycle. |
| D-03 | **Added** `n8n-syntax-extension-methods` | Research revealed 80+ n8n-specific extension methods (string, array, number, object, DateTime). Too substantial for the expressions skill — needs its own reference tables. |
| D-04 | **Renamed** `n8n-impl-workflow-patterns` to `n8n-impl-workflow-design` | Better reflects scope: sub-workflows, error workflows, branching/merge patterns, loops. "Design" signals architectural patterns, not just code snippets. |
| D-05 | **Reordered** batches to respect dependency chains | `syntax-expressions` moved earlier (batch 2) because code-node and workflow-design depend on expression knowledge. Core skills first, then syntax, then impl. |
| D-06 | **Confirmed** `n8n-syntax-ai-nodes` as standalone | Research confirmed 6 agent types, 3 chain types, 12+ model providers, 8 memory backends, 11 vector stores. Substantial enough for its own skill. |

**Result**: 22 raw skills → **21 definitive skills** (2 merges, 1 addition).

---

## Definitive Skill Inventory (21 skills)

### n8n-core/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `n8n-core-architecture` | Execution model; item-based data flow; INodeExecutionData structure; binary data; paired items; node types (trigger/regular); process modes (main/worker/queue); workflow settings; WorkflowExecuteMode | `INodeExecutionData`, `IBinaryData`, `IPairedItemData`, `IWorkflowSettings`, `WorkflowExecuteMode`, `ProxyInput` | Vooronderzoek §1-§1.6 | L | None |
| `n8n-core-api` | REST API endpoints; API key auth (X-N8N-API-KEY); cursor-based pagination; workflow CRUD; execution management; credential operations; user/tag/variable/project endpoints; API configuration | All `/api/v1/` endpoints, `X-N8N-API-KEY`, cursor pagination | Vooronderzoek §14 | L | None |

### n8n-syntax/ (8 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `n8n-syntax-workflow-json` | Workflow JSON structure; INode; IConnections (3-level nesting); NodeConnectionTypes (main + 12 AI types); node configuration objects; workflow settings; connection wiring format | `IWorkflowBase`, `INode`, `IConnections`, `NodeConnectionType` | Vooronderzoek §5 | L | core-architecture |
| `n8n-syntax-expressions` | Expression syntax ($json, $input, $node, $workflow, $execution, $env, $vars); JMESPath ($jmespath); paired items; static workflow data ($getWorkflowStaticData); utility functions ($ifEmpty); context availability per node type | All `$` variables, `$jmespath()`, `$getWorkflowStaticData()`, `$ifEmpty()` | Vooronderzoek §7, §9 | L | core-architecture |
| `n8n-syntax-extension-methods` | n8n-specific extension methods: 17 string methods, 21 array methods, 13 number methods, 12 object methods, 3 boolean methods, 25+ DateTime/Luxon methods, 7 BinaryFile properties | `.extractEmail()`, `.hash()`, `.toDateTime()`, `.pluck()`, `.compact()`, `.format()`, Luxon methods | Vooronderzoek §8 | M | syntax-expressions |
| `n8n-syntax-node-types` | INodeType interface; INodeTypeDescription; INodeProperties (22 types); displayOptions; execute() method; IExecuteFunctions; Node base class; versioned nodes; declarative nodes; methods (loadOptions, listSearch, credentialTest) | `INodeType`, `INodeTypeDescription`, `INodeProperties`, `IExecuteFunctions`, `IDisplayOptions` | Vooronderzoek §2, §3 | L | core-architecture |
| `n8n-syntax-credentials` | ICredentialType interface; credential properties; authenticate property (4 injection methods); OAuth2 configuration; API key auth; basic auth; credential testing (ICredentialTestRequest); genericCredentialRequest | `ICredentialType`, `IAuthenticate`, `ICredentialTestRequest` | Vooronderzoek §11 | L | syntax-node-types |
| `n8n-syntax-code-node` | Code node JavaScript/Python; runOnceForAllItems vs runOnceForEachItem; available variables ($input, $json, items); binary data in code; built-in modules; $() node access; Python _ prefix convention; restrictions (no fs, no http, no $secrets) | Code node API, `$input`, `$json`, `items`, Python `_` prefix | Vooronderzoek §10 | M | syntax-expressions |
| `n8n-syntax-trigger-nodes` | Webhook triggers; cron/schedule triggers; polling triggers; ITriggerFunctions; IWebhookFunctions; IPollFunctions; trigger node lifecycle; webhookMethods (checkExists/create/delete); WebhookResponseMode | `ITriggerFunctions`, `IWebhookFunctions`, `IPollFunctions`, `ITriggerResponse`, `WebhookSetupMethodNames` | Vooronderzoek §4 | M | syntax-node-types |
| `n8n-syntax-ai-nodes` | AI/LLM cluster node types; agent nodes (6 types); chain nodes; tool nodes; memory backends (8 types); vector stores (11 types); output parsers; AI sub-node connections; langchain integration; RAG patterns; human-in-the-loop | Cluster node API, AI sub-node connections | Vooronderzoek §13 | M | syntax-node-types |

### n8n-impl/ (6 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `n8n-impl-custom-nodes` | Custom node development workflow; n8n-nodes-starter template; community node packaging; node directory structure; package.json n8n field; programmatic vs declarative style; testing; publishing to npm | `n8n-nodes-starter`, `n8n.nodes`, `n8n.credentials`, packaging conventions | Vooronderzoek §6 | L | syntax-node-types, syntax-credentials |
| `n8n-impl-deployment` | Docker/Docker Compose setup; environment variables (100+); queue mode with Redis/BullMQ; worker processes; webhook processor; PostgreSQL/SQLite config; scaling strategies; volume mounts; CLI commands (export/import/execute); backup strategies; updating | Docker Compose, `EXECUTIONS_MODE`, `QUEUE_BULL_*`, `DB_*`, `n8n export/import` | Vooronderzoek §15, §16, §19, §20 | L | core-architecture |
| `n8n-impl-workflow-design` | Common workflow patterns; sub-workflow execution (Execute Workflow node); error workflows; retry logic; branching (IF/Switch); merge patterns (Merge node); loop patterns (Loop Over Items); wait/webhook resume; scheduling patterns | Execute Workflow node, Merge node, IF/Switch nodes, Loop Over Items, Wait node | Vooronderzoek §18 | M | syntax-expressions, core-architecture |
| `n8n-impl-integrations` | HTTP Request node patterns; OAuth2 credential flows; pagination handling; API integration best practices; generic credential usage; response handling; binary downloads; batch requests | HTTP Request node, OAuth2, generic credentials, pagination | Vooronderzoek §17 (webhook), §11 (credentials) | M | syntax-credentials |
| `n8n-impl-webhooks` | Webhook node setup; test vs production URLs; 6 HTTP methods; 4 response modes; 4 auth methods; Respond to Webhook node (8 response types); dynamic path parameters; binary data; CORS; IP whitelist; 16MB payload limit | Webhook node, Respond to Webhook node, response modes | Vooronderzoek §17 | M | syntax-trigger-nodes |
| `n8n-impl-security` | Environment variable security; credential encryption (N8N_ENCRYPTION_KEY); execution data pruning; CORS configuration; reverse proxy setup; SSL; task runners; external secrets; audit endpoint | `N8N_ENCRYPTION_KEY`, `N8N_RUNNERS_ENABLED`, security env vars | Vooronderzoek §19 | M | impl-deployment |

### n8n-errors/ (3 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `n8n-errors-execution` | Node execution failures; timeout handling; continueOnFail; error trigger node; error workflows; retry on fail (per-node); Stop And Error node; execution data inspection; 4 error handling patterns | `continueOnFail`, Error Trigger, Error Workflow, Stop And Error, retry config | Vooronderzoek §18 | M | impl-workflow-design |
| `n8n-errors-expressions` | Expression evaluation errors; undefined $json references; type mismatches; missing paired items; $itemIndex not available in Code node; $secrets not in Code node; JMESPath parameter order confusion | Expression debugging, common mistakes | Vooronderzoek §7, §9, §10 | M | syntax-expressions |
| `n8n-errors-connection` | API connection failures; retry strategies; rate limiting; credential errors; timeout configuration; SSL errors; webhook URL issues (test vs production); queue mode connection issues | HTTP errors, credential failures, webhook URL patterns | Vooronderzoek §17, §19 | M | impl-integrations |

### n8n-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `n8n-agents-review` | Workflow validation checklist; connection verification; credential check; expression validation; error handling audit; anti-pattern detection; node configuration check; JSON structure validation | All validation rules from all skills | All vooronderzoek anti-patterns | M | ALL syntax + impl skills |
| `n8n-agents-project-scaffolder` | Generate complete n8n setup: Docker Compose with PostgreSQL; workflow JSON templates; custom node scaffold; credential configuration; environment file; queue mode setup; project structure | All scaffolding patterns | Vooronderzoek §6, §15, §16 | L | ALL core + syntax skills |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-architecture`, `core-api`, `syntax-workflow-json` | 3 | None | Foundation, no deps |
| 2 | `syntax-expressions`, `syntax-node-types`, `syntax-credentials` | 3 | Batch 1 | Core syntax patterns |
| 3 | `syntax-extension-methods`, `syntax-code-node`, `syntax-trigger-nodes` | 3 | Batch 1-2 | Extended syntax + triggers |
| 4 | `syntax-ai-nodes`, `impl-custom-nodes`, `impl-deployment` | 3 | Batch 1-3 | AI nodes + first impl skills |
| 5 | `impl-workflow-design`, `impl-integrations`, `impl-webhooks` | 3 | Batch 1-3 | Core implementation patterns |
| 6 | `impl-security`, `errors-execution`, `errors-expressions` | 3 | Batch 4-5 | Security + first error skills |
| 7 | `errors-connection`, `agents-review`, `agents-project-scaffolder` | 3 | ALL above | Final batch |

**Total**: 21 skills across 7 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package
RESEARCH_FILE = C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\docs\research\vooronderzoek-n8n.md
RESEARCH_FRAGMENTS = C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\docs\research\fragments\
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: n8n-core-architecture

```
## Task: Create the n8n-core-architecture skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-core\n8n-core-architecture\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (INodeExecutionData, IBinaryData, IPairedItemData, IWorkflowSettings, WorkflowExecuteMode)
3. references/examples.md (item processing, binary data handling, paired items, workflow settings)
4. references/anti-patterns.md (data flow mistakes, item handling errors)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: n8n-core-architecture
description: "Guides n8n v1.x architecture including the item-based execution model, INodeExecutionData structure, binary data handling, paired items, node types (trigger/regular), process modes (main/worker/queue), and workflow settings. Activates when building n8n workflows, creating custom nodes, understanding data flow, or debugging execution issues."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Execution model: item-based processing, INodeExecutionData (json, binary, pairedItem, error)
- Data flow: items through nodes, execute() return type INodeExecutionData[][], multiple outputs
- Binary data: IBinaryData structure, binary key data, file types
- Paired items: IPairedItemData, item linking between nodes
- Execution modes: WorkflowExecuteMode (manual, webhook, trigger, retry, cli, error, integrated, queue, chat)
- Process modes: main process, worker processes, queue mode overview
- Workflow settings: IWorkflowSettings (timezone, errorWorkflow, executionTimeout, executionOrder)
- Expression data proxy: $json, $input, $binary, $execution, $workflow, $node, $now, $today

### Research Sections to Read
From research fragments:
- research-core-architecture.md: Section 1 (Execution Model) — complete
- research-core-architecture.md: Section 2.1 (INodeType Interface) — type hierarchy only
- research-expressions-credentials.md: Section 1.2 (Built-in Variables) — overview only

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be TypeScript for node development, JSON for workflow definitions
- Include version annotations (n8n v1.x)
- Include Critical Warnings section with NEVER rules
```

#### Prompt: n8n-core-api

```
## Task: Create the n8n-core-api skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-core\n8n-core-api\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete endpoint reference with methods, paths, scopes)
3. references/examples.md (API call examples with curl, authentication, pagination)
4. references/anti-patterns.md (API usage mistakes)

### YAML Frontmatter
---
name: n8n-core-api
description: "Guides n8n v1.x REST API including all endpoints (workflows, executions, credentials, users, tags, variables, projects), API key authentication via X-N8N-API-KEY header, cursor-based pagination, and API configuration. Activates when integrating with n8n programmatically, managing workflows via API, or building n8n admin tools."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Authentication: API key via X-N8N-API-KEY header, API key creation, scopes (enterprise)
- Pagination: cursor-based, limit (max 250), nextCursor
- Workflow endpoints: CRUD, activate/deactivate, transfer, tags, versions
- Execution endpoints: list, get, delete, retry, stop, tags
- Credential endpoints: CRUD, transfer, credential type schema
- User endpoints: list, get, role management
- Tag/Variable/Project endpoints
- API configuration env vars (N8N_PUBLIC_API_DISABLED, N8N_PUBLIC_API_ENDPOINT)
- API playground (Scalar UI) at /api/v1/docs

### Research Sections to Read
- research-deployment-api.md: Section 1 (REST API) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete endpoint reference table in references/methods.md
- Include curl examples for each major resource type
- Include pagination example with cursor handling
```

#### Prompt: n8n-syntax-workflow-json

```
## Task: Create the n8n-syntax-workflow-json skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-workflow-json\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (IWorkflowBase, INode, IConnections, NodeConnectionType)
3. references/examples.md (complete workflow JSON, minimal workflow, multi-output workflow)
4. references/anti-patterns.md (JSON structure mistakes, connection format errors)

### YAML Frontmatter
---
name: n8n-syntax-workflow-json
description: "Guides n8n v1.x workflow JSON structure including IWorkflowBase format, INode configuration, IConnections 3-level nesting (node name → connection type → output index → targets), NodeConnectionTypes (main + 12 AI types), node parameter formats, and workflow settings. Activates when creating workflow JSON, importing/exporting workflows, building workflow templates, or debugging connection issues."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- IWorkflowBase: id, name, nodes[], connections, active, settings, staticData, tags
- INode: id, name, type, typeVersion, position, parameters, credentials, disabled, notes, webhookId
- IConnections: 3-level nesting format (sourceName → connectionType → outputIndex → [{node, type, index}])
- NodeConnectionType: "main" + 12 AI types (ai_agent, ai_chain, ai_document, ai_embedding, ai_languageModel, ai_memory, ai_outputParser, ai_retriever, ai_textSplitter, ai_tool, ai_vectorStore, ai_file)
- Workflow settings object
- Complete JSON examples with proper connection wiring

### Research Sections to Read
- research-core-architecture.md: Section 5 (Workflow JSON Structure) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- MUST include complete connection format with all 3 nesting levels
- MUST include working workflow JSON example that validates
- Include NodeConnectionType enum reference
- CRITICAL: Connection format is the #1 source of errors — document thoroughly
```

---

### Batch 2

#### Prompt: n8n-syntax-expressions

```
## Task: Create the n8n-syntax-expressions skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-expressions\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md ($json, $input, $node, $workflow, $execution, $env, $vars, all variable signatures)
3. references/examples.md (expression patterns, JMESPath, static data, IIFE patterns)
4. references/anti-patterns.md (expression mistakes, context confusion)

### YAML Frontmatter
---
name: n8n-syntax-expressions
description: "Guides n8n v1.x expression system including all built-in variables ($json, $input, $node, $workflow, $execution, $env, $vars, $now, $today), JMESPath queries ($jmespath), paired items, static workflow data ($getWorkflowStaticData), utility functions ($ifEmpty), and expression context rules. Activates when writing n8n expressions, accessing node data, using JMESPath, or debugging expression errors."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Expression syntax: {{ expression }} double-curly-brace format
- Current node input: $json, $binary, $input (item, all, first, last, params)
- Other node output: $("node-name").all/first/last/item/itemMatching/params/context/isExecuted
- Workflow/execution metadata: $workflow (id, name, active), $execution (id, mode, resumeUrl, customData)
- Environment: $env, $vars, $secrets (NOT in Code node)
- Date/time: $now, $today (Luxon DateTime)
- Context variables: $prevNode, $runIndex, $itemIndex, $parameter, $nodeVersion
- Utility: $ifEmpty(), $jmespath(object, searchString), $getWorkflowStaticData(type)
- HTTP context: $response (body, headers, statusCode) — HTTP Request node only
- Paired items concept and item linking
- IIFE pattern for multi-statement expressions

### Research Sections to Read
- research-expressions-credentials.md: Section 1 (Expression System) — complete
- research-expressions-credentials.md: Section 3 (JMESPath) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include variable reference table with return types and descriptions
- Include JMESPath parameter order warning ($jmespath(object, search) NOT search, object)
- Include context availability matrix (which vars available where)
- $itemIndex NOT available in Code node — Critical Warning
```

#### Prompt: n8n-syntax-node-types

```
## Task: Create the n8n-syntax-node-types skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-node-types\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (INodeType, INodeTypeDescription, INodeProperties, IDisplayOptions, IExecuteFunctions)
3. references/examples.md (complete node implementation, property definitions, displayOptions patterns)
4. references/anti-patterns.md (node type mistakes, property configuration errors)

### YAML Frontmatter
---
name: n8n-syntax-node-types
description: "Guides n8n v1.x node type system including INodeType interface, INodeTypeDescription, INodeProperties (22 types), displayOptions with rich conditions, execute() method with IExecuteFunctions, Node base class alternative, versioned nodes, declarative routing, and methods (loadOptions, listSearch, credentialTest, resourceMapping). Activates when creating custom n8n nodes, defining node properties, implementing execute methods, or building declarative nodes."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- INodeType interface: description, execute, trigger, webhook, poll, supplyData, methods, webhookMethods
- INodeTypeDescription: displayName, name, icon, group, version, description, defaults, inputs, outputs, credentials, properties
- INodeProperties: 22 property types (boolean, string, number, options, multiOptions, collection, fixedCollection, json, filter, resourceMapper, assignmentCollection, etc.)
- IDisplayOptions: show/hide conditions with operators
- INodePropertyTypeOptions: all type-specific options
- execute() method: IExecuteFunctions, getInputData, getNodeParameter, return type
- Node base class: context parameter vs this binding
- Versioned nodes: IVersionedNodeType, nodeVersions
- Declarative nodes: routing property
- methods: loadOptions, listSearch, credentialTest, resourceMapping, actionHandler

### Research Sections to Read
- research-core-architecture.md: Section 2 (INodeType Interface) — complete
- research-core-architecture.md: Section 3 (Node Properties) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete INodeType interface in references/
- Include property type reference table (all 22 types)
- Include displayOptions example with conditions
- execute() return type INodeExecutionData[][] is CRITICAL
```

#### Prompt: n8n-syntax-credentials

```
## Task: Create the n8n-syntax-credentials skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-credentials\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (ICredentialType, IAuthenticate, ICredentialTestRequest, property types)
3. references/examples.md (API key credential, OAuth2 credential, basic auth, credential test)
4. references/anti-patterns.md (credential configuration mistakes)

### YAML Frontmatter
---
name: n8n-syntax-credentials
description: "Guides n8n v1.x credential system including ICredentialType interface, credential properties, authenticate property with 4 injection methods (query, header, body, basic auth), OAuth2 configuration, API key patterns, credential testing via ICredentialTestRequest, and genericCredentialRequest. Activates when creating custom credentials, implementing authentication, configuring OAuth2, or testing credential connections."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- ICredentialType interface: name, displayName, documentationUrl, properties, authenticate, test
- Credential properties: 12 UI element types
- authenticate property: type 'generic' with 4 injection methods (qs, headers, body, basicAuth)
- OAuth2 configuration: token URLs, scopes, grant type
- API key authentication patterns
- Basic auth patterns
- Credential testing: ICredentialTestRequest
- genericCredentialRequest pattern
- Credential registration in package.json (n8n.credentials)

### Research Sections to Read
- research-expressions-credentials.md: Section 5 (Credential System) — complete
- research-core-architecture.md: Section 6 (Custom Node Development) — credential registration

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete ICredentialType interface
- Include examples for all 4 authenticate injection methods
- Include OAuth2 credential template
- Include credential test pattern
```

---

### Batch 3

#### Prompt: n8n-syntax-extension-methods

```
## Task: Create the n8n-syntax-extension-methods skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-extension-methods\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (ALL extension methods: string, array, number, object, boolean, DateTime, BinaryFile)
3. references/examples.md (practical extension method usage patterns)
4. references/anti-patterns.md (common method misuse)

### YAML Frontmatter
---
name: n8n-syntax-extension-methods
description: "Guides n8n v1.x expression extension methods including 17 string methods (.extractEmail, .hash, .toDateTime, .urlEncode), 21 array methods (.pluck, .removeDuplicates, .chunk, .smartJoin), 13 number methods, 12 object methods (.compact, .keepFieldsContaining), 3 boolean methods, 25+ DateTime/Luxon methods (.plus, .minus, .format, .diffTo), and 7 BinaryFile properties. Activates when transforming data in expressions, formatting dates, manipulating arrays/objects, or working with binary file metadata."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- String methods: extractEmail, extractUrl, extractDomain, extractUrlPath, isEmail, isUrl, isEmpty, isNotEmpty, removeMarkdown, toDateTime, toBoolean, toSentenceCase, toSnakeCase, toTitleCase, urlEncode, urlDecode, base64Encode, base64Decode, hash, quote, replaceSpecialChars
- Array methods: append, average, chunk, compact, difference, intersection, isEmpty, first, last, max, min, pluck, randomItem, removeDuplicates, renameKeys, smartJoin, sum, toJsonString, union, unique
- Number methods: abs, ceil, floor, format, isEmpty, isEven, isInteger, isOdd, round, toBoolean, toDateTime
- Object methods: compact, hasField, isEmpty, isNotEmpty, keepFieldsContaining, keys, merge, removeField, removeFieldsContaining, toJsonString, urlEncode, values
- Boolean methods: isEmpty, toNumber, toString
- DateTime: property accessors (day, hour, month, weekday, etc.), transformation (plus, minus, set, startOf, endOf, setZone), comparison (equals, hasSame, isBetween, diffTo, diffToNow), formatting (format, toISO, toRelative, toMillis)
- BinaryFile properties: directory, fileExtension, fileName, fileSize, fileType, id, mimeType

### Research Sections to Read
- research-expressions-credentials.md: Section 2 (Expression Extension Methods) — ALL subsections

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- SKILL.md should contain decision trees and quick reference, NOT all method tables
- ALL method tables MUST go in references/methods.md
- Include practical examples combining multiple methods
- DateTime Luxon format tokens reference is CRITICAL
```

#### Prompt: n8n-syntax-code-node

```
## Task: Create the n8n-syntax-code-node skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-code-node\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (available variables, mode differences, built-in modules)
3. references/examples.md (JavaScript and Python patterns, item manipulation, binary data)
4. references/anti-patterns.md (code node mistakes, variable unavailability)

### YAML Frontmatter
---
name: n8n-syntax-code-node
description: "Guides n8n v1.x Code node including JavaScript and Python modes, runOnceForAllItems vs runOnceForEachItem execution, available variables ($input, $json, items), binary data handling, built-in modules, $() node access, Python _ prefix convention, and restrictions. Activates when writing Code node logic, transforming data programmatically, or debugging Code node errors."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Two modes: runOnceForAllItems (default, process all items) vs runOnceForEachItem (per-item)
- JavaScript: available variables, return format, item manipulation
- Python: _ prefix convention (_json, _input, _jmespath), native support (v1.111.0+)
- Variables: $input, $json, items, $(), $execution, $workflow, $now, $today, $env, $vars
- Restrictions: no filesystem access, no HTTP requests, no $itemIndex, no $secrets, no $parameter
- Binary data: accessing, creating, transforming
- Built-in modules: available in JavaScript/Python
- Return format: must return items array [{json: {...}}]
- Error handling in Code node

### Research Sections to Read
- research-expressions-credentials.md: Section 4 (Code Node) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include mode decision tree (when to use each mode)
- Include JavaScript AND Python examples
- CRITICAL: $itemIndex NOT available in Code node
- CRITICAL: $secrets NOT available in Code node
- CRITICAL: Return format must be [{json: {...}}]
```

#### Prompt: n8n-syntax-trigger-nodes

```
## Task: Create the n8n-syntax-trigger-nodes skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-trigger-nodes\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (ITriggerFunctions, IWebhookFunctions, IPollFunctions, ITriggerResponse, webhookMethods)
3. references/examples.md (schedule trigger, webhook trigger, polling trigger)
4. references/anti-patterns.md (trigger implementation mistakes)

### YAML Frontmatter
---
name: n8n-syntax-trigger-nodes
description: "Guides n8n v1.x trigger node development including three trigger patterns (event/timer, polling, webhook), ITriggerFunctions, IWebhookFunctions, IPollFunctions interfaces, ITriggerResponse with manualTriggerFunction and closeFunction, webhook lifecycle methods (checkExists/create/delete), and WebhookResponseMode types. Activates when creating custom trigger nodes, implementing webhook handlers, building polling triggers, or understanding trigger node lifecycle."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Three trigger patterns: event/timer triggers, polling triggers, webhook triggers
- ITriggerFunctions interface: emit(), getNodeParameter(), helpers
- ITriggerResponse: closeFunction (cleanup), manualTriggerFunction (test execution)
- IPollFunctions interface: poll cycle, deduplication
- IWebhookFunctions interface: getRequestObject(), getResponseObject(), getBodyData()
- Webhook lifecycle: webhookMethods (checkExists, create, delete)
- WebhookResponseMode: onReceived, lastNode, responseNode, default handling
- Trigger node description: group includes 'trigger', webhook paths
- Event emitting pattern: this.emit([[items]])

### Research Sections to Read
- research-core-architecture.md: Section 4 (Trigger Nodes) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include decision tree: event vs polling vs webhook trigger
- Include complete trigger lifecycle diagram
- Include working examples for all three trigger patterns
```

---

### Batch 4

#### Prompt: n8n-syntax-ai-nodes

```
## Task: Create the n8n-syntax-ai-nodes skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-syntax\n8n-syntax-ai-nodes\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (AI node types, cluster node architecture, sub-node connections)
3. references/examples.md (agent workflow, RAG workflow, tool usage)
4. references/anti-patterns.md (AI workflow mistakes)

### YAML Frontmatter
---
name: n8n-syntax-ai-nodes
description: "Guides n8n v1.x AI/LLM cluster node system including agent nodes (6 types), chain nodes, tool nodes, memory backends (8 types), vector stores (11 types), output parsers, text splitters, retrievers, AI sub-node connections (12 NodeConnectionTypes), langchain integration, RAG patterns, and human-in-the-loop. Activates when building AI workflows, connecting LLM models, implementing RAG, or using AI agents in n8n."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x (v1.19.4+ for AI features)."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Cluster node architecture: root nodes with sub-node connectors
- Agent types: Conversational Agent, OpenAI Functions Agent, Plan and Execute, ReAct, SQL Agent, Tools Agent
- Chain types: Basic LLM Chain, Summarization Chain, Retrieval QA Chain
- AI sub-node connection types: ai_agent, ai_chain, ai_document, ai_embedding, ai_languageModel, ai_memory, ai_outputParser, ai_retriever, ai_textSplitter, ai_tool, ai_vectorStore, ai_file
- Chat model providers: OpenAI, Anthropic, Azure OpenAI, Google, Ollama, etc.
- Memory backends: Buffer, Window Buffer, Token Buffer, Summary, PostgresChat, Redis, Xata, Zep
- Vector stores: Pinecone, Qdrant, Supabase, PGVector, Chroma, Weaviate, In-Memory, etc.
- Output parsers: Structured, Auto-fixing, Item List
- RAG patterns: document loading, splitting, embedding, retrieval
- Human-in-the-loop patterns

### Research Sections to Read
- research-expressions-credentials.md: Section 6 (AI/LLM Nodes) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include AI node type reference table
- Include sub-node connection type reference
- Include RAG workflow pattern
- Include decision tree: which agent type to use
```

#### Prompt: n8n-impl-custom-nodes

```
## Task: Create the n8n-impl-custom-nodes skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-impl\n8n-impl-custom-nodes\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (project structure, package.json config, node registration)
3. references/examples.md (complete custom node, declarative node, credential, test setup)
4. references/anti-patterns.md (packaging and development mistakes)

### YAML Frontmatter
---
name: n8n-impl-custom-nodes
description: "Guides n8n v1.x custom node development workflow including n8n-nodes-starter template, community node packaging, project structure, package.json n8n field configuration, programmatic vs declarative node styles, credential registration, testing with WorkflowTestData, and publishing to npm. Activates when creating custom n8n nodes, building community node packages, or publishing nodes to the n8n community."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- n8n-nodes-starter template project setup
- Project structure: nodes/, credentials/, package.json, tsconfig.json
- package.json n8n field: nodes array, credentials array, registration format
- Programmatic node style (execute method, this.getInputData)
- Declarative node style (routing property, request definitions)
- Credential creation and registration
- Development workflow: npm link, testing, debugging
- Testing with WorkflowTestData
- Publishing to npm: naming convention (n8n-nodes-*), community nodes
- Icon handling: SVG or PNG

### Research Sections to Read
- research-core-architecture.md: Section 6 (Custom Node Development) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include step-by-step development workflow
- Include complete project structure diagram
- Include package.json template
- Include both programmatic and declarative examples
```

#### Prompt: n8n-impl-deployment

```
## Task: Create the n8n-impl-deployment skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-impl\n8n-impl-deployment\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (environment variable reference — 100+ vars organized by category)
3. references/examples.md (Docker Compose configs, queue mode setup, CLI commands)
4. references/anti-patterns.md (deployment mistakes, security misconfigurations)

### YAML Frontmatter
---
name: n8n-impl-deployment
description: "Guides n8n v1.x deployment including Docker/Docker Compose setup, 100+ environment variables, queue mode with Redis/BullMQ, worker processes, webhook processor, PostgreSQL/SQLite configuration, volume mounts, Traefik reverse proxy, scaling strategies, CLI commands (export/import/execute), backup strategies, and updating procedures. Activates when deploying n8n, configuring Docker, setting up queue mode, managing environment variables, or performing backup/restore operations."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Docker deployment: basic run, PostgreSQL, Docker Compose with Traefik
- Volume mounts: /home/node/.n8n, /files, /letsencrypt
- Environment variables: 100+ vars across deployment, database, execution, credentials, endpoints, security, SSL, nodes, workflows, proxy, binary data, task runners
- Queue mode: EXECUTIONS_MODE=queue, Redis config, worker commands, webhook processor
- Database: PostgreSQL (DB_TYPE=postgresdb), SQLite (default), connection vars
- Scaling: main + workers + Redis architecture, concurrency, multi-main setup
- CLI commands: n8n execute, n8n export:workflow, n8n import:workflow, n8n export:credentials
- Backup strategies: CLI export, database dump, volume backup
- Configuration methods: env vars, Docker Compose, _FILE suffix for secrets
- Updating: docker pull, container replacement

### Research Sections to Read
- research-deployment-api.md: Section 2 (Docker Deployment) — complete
- research-deployment-api.md: Section 3 (Queue Mode & Scaling) — complete
- research-deployment-api.md: Section 6 (Environment Variables) — complete
- research-deployment-api.md: Section 7 (CLI & Backup) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include Docker Compose production template
- Include environment variable reference table (in references/)
- Include queue mode architecture diagram
- CRITICAL: N8N_ENCRYPTION_KEY must be shared across all instances
- CRITICAL: PostgreSQL required for queue mode (NOT SQLite)
```

---

### Batch 5

#### Prompt: n8n-impl-workflow-design

```
## Task: Create the n8n-impl-workflow-design skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-impl\n8n-impl-workflow-design\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (core workflow nodes: Execute Workflow, Merge, IF, Switch, Loop Over Items, Wait, Error Trigger, Stop And Error)
3. references/examples.md (workflow patterns: sub-workflow, error handling, branching, loops, scheduling)
4. references/anti-patterns.md (workflow design mistakes)

### YAML Frontmatter
---
name: n8n-impl-workflow-design
description: "Guides n8n v1.x workflow design patterns including sub-workflow execution (Execute Workflow node), error workflows and Error Trigger node, retry logic, branching (IF/Switch nodes), merge patterns (Merge node modes), loop patterns (Loop Over Items), wait/webhook resume, scheduling patterns, and Stop And Error node. Activates when designing complex workflows, implementing error handling, building sub-workflow architectures, or creating branching logic."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Sub-workflows: Execute Workflow node, callerPolicy, parameter passing
- Error handling: Error Trigger node, error workflows, continueOnFail, retry on fail, Stop And Error
- Branching: IF node (conditions), Switch node (routing), multiple outputs
- Merge patterns: Merge node modes (append, combine, choose branch, SQL query)
- Loop patterns: Loop Over Items (batch processing), split in batches
- Wait/Resume: Wait node, webhook resume ($execution.resumeUrl), form resume
- Scheduling: Schedule Trigger, Cron expressions
- Workflow settings: errorWorkflow, executionTimeout, callerIds

### Research Sections to Read
- research-deployment-api.md: Section 5 (Error Handling) — complete
- research-core-architecture.md: Section 1.5 (Workflow Settings)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include decision tree: which pattern for which use case
- Include error handling workflow diagram
- Include sub-workflow parameter passing pattern
```

#### Prompt: n8n-impl-integrations

```
## Task: Create the n8n-impl-integrations skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-impl\n8n-impl-integrations\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (HTTP Request node options, OAuth2 flow, pagination options, generic credentials)
3. references/examples.md (REST API integration, OAuth2 flow, paginated fetching, batch requests)
4. references/anti-patterns.md (integration mistakes)

### YAML Frontmatter
---
name: n8n-impl-integrations
description: "Guides n8n v1.x API integration patterns including HTTP Request node configuration, OAuth2 credential flows, pagination handling, generic credential usage for custom APIs, response handling, binary downloads, batch request patterns, and API error handling. Activates when integrating external APIs, configuring OAuth2, handling paginated responses, or building custom API workflows."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- HTTP Request node: methods, URL, authentication, headers, query parameters, body types
- Authentication options: predefined credentials, generic credentials, header/query auth
- OAuth2 flow: credential configuration, callback URL, token refresh
- Pagination: automatic pagination options, cursor-based, offset-based, manual
- Response handling: JSON, text, binary, file downloads
- Batch requests: Send and Wait, batching strategies
- Error handling: retry on fail, error output, status code handling
- Generic credential request pattern for custom APIs

### Research Sections to Read
- research-expressions-credentials.md: Section 5 (Credential System)
- research-deployment-api.md: Section 4 (Webhooks) — response handling patterns

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include HTTP Request node configuration reference
- Include OAuth2 setup workflow (step by step)
- Include pagination pattern examples
```

#### Prompt: n8n-impl-webhooks

```
## Task: Create the n8n-impl-webhooks skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-impl\n8n-impl-webhooks\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (Webhook node options, Respond to Webhook options, response modes, auth methods)
3. references/examples.md (basic webhook, authenticated webhook, file upload, custom response)
4. references/anti-patterns.md (webhook configuration mistakes)

### YAML Frontmatter
---
name: n8n-impl-webhooks
description: "Guides n8n v1.x webhook implementation including Webhook node configuration (6 HTTP methods, test vs production URLs, 4 response modes, 4 auth methods), Respond to Webhook node (8 response types including JWT and streaming), dynamic path parameters, binary data handling, CORS configuration, IP whitelist, and 16MB payload limit. Activates when setting up webhooks, configuring webhook authentication, handling webhook responses, or debugging webhook URL issues."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Webhook node: path, HTTP methods (GET, POST, PUT, PATCH, DELETE, HEAD)
- Test vs production URLs: test URL format, production URL format, WEBHOOK_URL env var
- Response modes: immediately, when last node finishes, using Respond to Webhook node, default
- Authentication: none, basic auth, header auth, JWT auth
- Respond to Webhook node: 8 response types (text, JSON, binary, JWT, redirect, noData, streaming, form submitted)
- Dynamic path parameters: :param syntax
- Binary data via webhooks: file uploads, multipart handling
- CORS configuration
- IP whitelist
- 16MB payload limit
- Webhook lifecycle: registration, test mode, production activation

### Research Sections to Read
- research-deployment-api.md: Section 4 (Webhooks) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- CRITICAL: Test vs production URL difference — document thoroughly
- Include response mode decision tree
- Include authentication configuration examples
```

---

### Batch 6

#### Prompt: n8n-impl-security

```
## Task: Create the n8n-impl-security skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-impl\n8n-impl-security\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (security environment variables, encryption settings, audit API)
3. references/examples.md (security hardening, reverse proxy config, SSL setup)
4. references/anti-patterns.md (security mistakes)

### YAML Frontmatter
---
name: n8n-impl-security
description: "Guides n8n v1.x security implementation including credential encryption (N8N_ENCRYPTION_KEY), execution data pruning, CORS configuration, reverse proxy setup, SSL/TLS, task runners (N8N_RUNNERS_ENABLED), external secrets, security audit endpoint, Docker Secrets (_FILE suffix), and environment variable security. Activates when hardening n8n security, configuring encryption, setting up reverse proxy, or performing security audits."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Credential encryption: N8N_ENCRYPTION_KEY, sharing across instances
- Task runners: N8N_RUNNERS_ENABLED=true (sandboxed code execution)
- Execution data pruning: EXECUTIONS_DATA_PRUNE, EXECUTIONS_DATA_MAX_AGE
- Docker Secrets: _FILE suffix pattern for sensitive values
- External secrets: integration with secret providers
- CORS: allowed origins configuration
- Reverse proxy: Traefik, Nginx patterns
- SSL/TLS: certificate configuration
- Security audit: POST /audit endpoint
- Environment variable security: which vars contain sensitive data
- Security hardening checklist

### Research Sections to Read
- research-deployment-api.md: Section 6 (Environment Variables & Security)
- research-deployment-api.md: Section 2.6 (Configuration Methods)

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include security hardening checklist
- CRITICAL: N8N_ENCRYPTION_KEY MUST be backed up — losing it means losing all credentials
- Include _FILE suffix pattern for Docker Secrets
```

#### Prompt: n8n-errors-execution

```
## Task: Create the n8n-errors-execution skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-errors\n8n-errors-execution\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (error handling nodes, continueOnFail API, retry configuration)
3. references/examples.md (error workflow setup, continueOnFail patterns, retry patterns)
4. references/anti-patterns.md (error handling mistakes)

### YAML Frontmatter
---
name: n8n-errors-execution
description: "Diagnoses and resolves n8n v1.x execution errors including node failures, timeout handling, continueOnFail pattern, Error Trigger node, error workflows, retry on fail configuration, Stop And Error node, execution data inspection, and the 4 error handling patterns. Activates when debugging workflow failures, implementing error recovery, configuring retry logic, or setting up error notification workflows."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Error workflows: setup in workflow settings, Error Trigger node receives error data
- continueOnFail: per-node setting, error output vs success output
- Retry on fail: per-node configuration (max retries, wait between)
- Stop And Error: explicitly stop workflow with error message
- Execution timeout: workflow-level and instance-level settings
- 4 error handling patterns: error workflow, continueOnFail, retry, stop and error
- Execution data inspection: execution list, data viewer
- Error trigger data structure: what data is available in error workflow
- Common execution errors and their causes

### Research Sections to Read
- research-deployment-api.md: Section 5 (Error Handling) — complete

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic patterns: Symptom → Cause → Fix
- Include error handling decision tree
- Include complete error workflow setup guide
```

#### Prompt: n8n-errors-expressions

```
## Task: Create the n8n-errors-expressions skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-errors\n8n-errors-expressions\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (expression error types, variable availability matrix)
3. references/examples.md (common expression errors with fixes)
4. references/anti-patterns.md (expression anti-patterns)

### YAML Frontmatter
---
name: n8n-errors-expressions
description: "Diagnoses and resolves n8n v1.x expression errors including undefined $json references, type mismatches, missing paired items, $itemIndex unavailability in Code node, $secrets restriction in Code node, JMESPath parameter order confusion, empty expression results, and context-dependent variable availability. Activates when debugging expression evaluation failures, fixing undefined references, or troubleshooting data access issues."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Undefined $json references: accessing fields that don't exist
- Type mismatches: string vs number, null handling
- Paired item errors: missing item links, broken pairing
- $itemIndex not in Code node: use items index instead
- $secrets not in Code node: use $env or other methods
- JMESPath parameter order: $jmespath(object, search) NOT search(search, object)
- Empty results: expression returns undefined/null
- Context-dependent variables: which are available where
- Static data pitfalls: not available during testing
- Expression debugging: preview in editor, test values
- $response only in HTTP Request node context

### Research Sections to Read
- research-expressions-credentials.md: Section 1 (Expression System) — variable availability
- research-expressions-credentials.md: Section 3 (JMESPath) — parameter order
- research-expressions-credentials.md: Section 4 (Code Node) — restrictions

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table: Symptom → Cause → Fix
- Include variable availability matrix
- Include common error messages with explanations
```

---

### Batch 7

#### Prompt: n8n-errors-connection

```
## Task: Create the n8n-errors-connection skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-errors\n8n-errors-connection\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (connection error types, retry configuration, timeout settings)
3. references/examples.md (connection error scenarios with fixes)
4. references/anti-patterns.md (connection handling mistakes)

### YAML Frontmatter
---
name: n8n-errors-connection
description: "Diagnoses and resolves n8n v1.x connection errors including API failures, credential errors, timeout configuration, SSL/TLS issues, webhook URL problems (test vs production), rate limiting, queue mode connection issues (Redis), database connection failures, and retry strategies. Activates when troubleshooting API connection failures, credential authentication errors, timeout issues, or webhook delivery problems."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- API connection failures: HTTP errors, DNS resolution, network timeouts
- Credential errors: invalid credentials, expired tokens, wrong credential type
- Timeout configuration: node-level, workflow-level, instance-level
- SSL/TLS errors: self-signed certificates, certificate chain issues
- Webhook URL issues: test URL only works in editor, production URL requires active workflow
- Rate limiting: API rate limits, backoff strategies
- Queue mode: Redis connection failures, worker connection issues
- Database: PostgreSQL connection errors, SQLite locking
- Retry strategies: exponential backoff, max retries, delay configuration
- WEBHOOK_URL environment variable misconfiguration

### Research Sections to Read
- research-deployment-api.md: Section 3 (Queue Mode) — connection requirements
- research-deployment-api.md: Section 4 (Webhooks) — URL patterns
- research-deployment-api.md: Section 5 (Error Handling) — retry configuration

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Format as diagnostic table: Symptom → Cause → Fix
- CRITICAL: Test vs production webhook URL confusion
- Include retry configuration reference
```

#### Prompt: n8n-agents-review

```
## Task: Create the n8n-agents-review skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-agents\n8n-agents-review\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (complete validation checklist organized by area)
3. references/examples.md (review scenarios: good code, bad code, fixes)
4. references/anti-patterns.md (all anti-patterns consolidated)

### YAML Frontmatter
---
name: n8n-agents-review
description: "Validates generated n8n v1.x code and workflows for correctness by checking workflow JSON structure, node configuration, connection wiring, expression syntax, credential setup, error handling patterns, deployment configuration, and known anti-patterns. Activates when reviewing n8n workflows, validating workflow JSON, checking custom node code, or auditing n8n deployment configuration."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Workflow JSON validation: IConnections format, node configuration, required fields
- Connection wiring: 3-level nesting correct, connection types match node types
- Expression validation: correct variable references, context-appropriate variables
- Credential validation: proper ICredentialType, authenticate method, test method
- Node type validation: INodeType completeness, execute return type, property types
- Error handling: error workflow configured, continueOnFail where appropriate
- Deployment: encryption key set, queue mode requirements met, volume mounts correct
- Code node: correct return format, no restricted variables
- Anti-pattern detection: all known anti-patterns from all skills
- Security: credentials not hardcoded, encryption configured

### Research Sections to Read
All research fragments — anti-patterns and critical warnings from every section

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Structure as a checklist runnable against any n8n project
- Group checks by area (JSON, nodes, expressions, credentials, deployment, security)
- Each check: what to verify, expected state, common failure
```

#### Prompt: n8n-agents-project-scaffolder

```
## Task: Create the n8n-agents-project-scaffolder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\n8n-Claude-Skill-Package\skills\source\n8n-agents\n8n-agents-project-scaffolder\

### Files to Create
1. SKILL.md (<500 lines)
2. references/methods.md (scaffolding templates, Docker Compose template, custom node template)
3. references/examples.md (complete project scaffolds: deployment, custom node package, workflow template)
4. references/anti-patterns.md (scaffolding mistakes)

### YAML Frontmatter
---
name: n8n-agents-project-scaffolder
description: "Generates complete n8n v1.x project structures including Docker Compose with PostgreSQL configuration, environment file templates, custom node package scaffolds (n8n-nodes-starter), workflow JSON templates, credential configurations, queue mode setup, and backup scripts. Activates when creating new n8n deployments, scaffolding custom node packages, or generating workflow templates."
license: MIT
compatibility: "Designed for Claude Code. Requires n8n v1.x."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT)
- Deployment scaffold: Docker Compose (PostgreSQL + n8n + Traefik), .env template, volume setup
- Queue mode scaffold: Docker Compose with Redis + workers, scaling configuration
- Custom node scaffold: n8n-nodes-starter based, package.json, tsconfig.json, node template, credential template
- Workflow templates: basic webhook workflow, error handling workflow, sub-workflow pattern, scheduled workflow
- Environment template: all essential variables with documentation
- Decision tree: which scaffold based on project requirements
- Security defaults: encryption key, task runners, credential encryption

### Research Sections to Read
All research fragments — deployment patterns, custom node patterns, workflow patterns

### Quality Rules
- English only, <500 lines SKILL.md, ALWAYS/NEVER language
- Include complete file templates for each scaffold type
- Include decision tree: deployment type selection
- Generated code MUST follow all patterns from other skills
- All Docker Compose MUST include proper volume mounts
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── n8n-core/
│   ├── n8n-core-architecture/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── methods.md
│   │       ├── examples.md
│   │       └── anti-patterns.md
│   └── n8n-core-api/
│       ├── SKILL.md
│       └── references/
├── n8n-syntax/
│   ├── n8n-syntax-workflow-json/
│   ├── n8n-syntax-expressions/
│   ├── n8n-syntax-extension-methods/
│   ├── n8n-syntax-node-types/
│   ├── n8n-syntax-credentials/
│   ├── n8n-syntax-code-node/
│   ├── n8n-syntax-trigger-nodes/
│   └── n8n-syntax-ai-nodes/
├── n8n-impl/
│   ├── n8n-impl-custom-nodes/
│   ├── n8n-impl-deployment/
│   ├── n8n-impl-workflow-design/
│   ├── n8n-impl-integrations/
│   ├── n8n-impl-webhooks/
│   └── n8n-impl-security/
├── n8n-errors/
│   ├── n8n-errors-execution/
│   ├── n8n-errors-expressions/
│   └── n8n-errors-connection/
└── n8n-agents/
    ├── n8n-agents-review/
    └── n8n-agents-project-scaffolder/
```
