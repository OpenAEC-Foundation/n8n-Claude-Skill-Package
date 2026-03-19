# n8n Expression Methods — Complete Reference

## Current Node Input Variables

### $json

- **Returns**: `Object`
- **Signature**: `$json` (property access, no arguments)
- **Description**: Shorthand for `$input.item.json`. Returns the JSON data of the item currently being processed. This is the most commonly used variable in n8n expressions.
- **Example**: `{{ $json.email }}`, `{{ $json.address.city }}`

### $binary

- **Returns**: `Object`
- **Signature**: `$binary` (property access, no arguments)
- **Description**: Shorthand for `$input.item.binary`. Returns the binary data of the current item. Each binary property contains `mimeType`, `fileExtension`, `fileName`, `fileSize`, `fileType`, and `data`.
- **Example**: `{{ $binary.data.fileName }}`, `{{ $binary.attachment.mimeType }}`

### $input.item

- **Returns**: `Item` (object with `.json` and `.binary` properties)
- **Signature**: `$input.item`
- **Description**: The complete input item currently being processed, including both JSON and binary data.

### $input.all()

- **Returns**: `Array<Item>`
- **Signature**: `$input.all(branchIndex?: number, runIndex?: number)`
- **Description**: All input items from the current node. Optional parameters select specific branch (for nodes with multiple inputs) and run index.
- **Example**: `{{ $input.all().length }}` — count of all input items

### $input.first()

- **Returns**: `Item`
- **Signature**: `$input.first(branchIndex?: number, runIndex?: number)`
- **Description**: First input item. Useful for accessing header/config data.

### $input.last()

- **Returns**: `Item`
- **Signature**: `$input.last(branchIndex?: number, runIndex?: number)`
- **Description**: Last input item.

### $input.params

- **Returns**: `Object`
- **Signature**: `$input.params`
- **Description**: Node configuration settings and operation parameters for the current node.

---

## Other Node Output Variables

### $("node-name")

Returns a reference object for accessing another node's output data.

#### .all()

- **Returns**: `Array<Item>`
- **Signature**: `$("node-name").all(branchIndex?: number, runIndex?: number)`
- **Description**: All output items from the named node.
- **Example**: `{{ $("HTTP Request").all().length }}`

#### .first()

- **Returns**: `Item`
- **Signature**: `$("node-name").first(branchIndex?: number, runIndex?: number)`
- **Description**: First output item from the named node.
- **Example**: `{{ $("Webhook").first().json.body.userId }}`

#### .last()

- **Returns**: `Item`
- **Signature**: `$("node-name").last(branchIndex?: number, runIndex?: number)`
- **Description**: Last output item from the named node.

#### .item

- **Returns**: `Item`
- **Signature**: `$("node-name").item`
- **Description**: The linked/paired item — the item in the specified node that was used to produce the current item. Uses n8n's automatic item linking.
- **Example**: `{{ $("Get User").item.json.name }}`

#### .itemMatching()

- **Returns**: `Item`
- **Signature**: `$("node-name").itemMatching(currentNodeInputIndex: number)`
- **Description**: Traces back from input item at the given index to the matching item in the named node. Preferred method in the Code node where `.item` may not resolve correctly.
- **Example**: `$("Webhook").itemMatching(i).json.originalData`

#### .params

- **Returns**: `Object`
- **Signature**: `$("node-name").params`
- **Description**: Query settings/parameters configured on the named node.

#### .context

- **Returns**: `Object`
- **Signature**: `$("node-name").context`
- **Description**: Context object for Loop Over Items nodes. Only available when referencing a Loop Over Items node.

#### .isExecuted

- **Returns**: `Boolean`
- **Signature**: `$("node-name").isExecuted`
- **Description**: Returns `true` if the named node has executed in the current workflow run, `false` otherwise.

---

## Workflow & Execution Metadata

### $workflow

| Property | Returns | Description |
|----------|---------|-------------|
| `$workflow.id` | String | The workflow ID (also visible in the workflow URL) |
| `$workflow.name` | String | The workflow name as shown in the editor |
| `$workflow.active` | Boolean | Whether the workflow is currently active |

### $execution

| Property | Returns | Description |
|----------|---------|-------------|
| `$execution.id` | String | Unique ID of the current execution |
| `$execution.mode` | String | `"test"` (manual), `"production"` (trigger), `"evaluation"` (workflow tests) |
| `$execution.resumeUrl` | String | Webhook URL to resume a workflow waiting at a Wait node |
| `$execution.resumeFormUrl` | String | URL for accessing a Wait node form |
| `$execution.customData` | CustomData | Custom execution data store (see below) |

### $execution.customData Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `.set()` | `set(key: string, value: string)` | void | Set a single key-value pair |
| `.setAll()` | `setAll(data: Record<string, string>)` | void | Set multiple pairs at once |
| `.get()` | `get(key: string)` | string | Get value for a specific key |
| `.getAll()` | `getAll()` | Record\<string, string\> | Get all custom data |

---

## Environment & Configuration

### $env

- **Returns**: `Object`
- **Signature**: `$env` (property access)
- **Description**: n8n instance configuration environment variables. Access via `$env.VARIABLE_NAME`.

### $vars

- **Returns**: `Object`
- **Signature**: `$vars` (property access)
- **Description**: Variables defined in the active n8n environment. All values are strings. Access via `$vars.variableName`.

### $secrets

- **Returns**: `Object`
- **Signature**: `$secrets` (property access)
- **Description**: External secrets provider values. **NOT available in the Code node.** Access via `$secrets.provider.secretName`.

---

## Date & Time

### $now

- **Returns**: `DateTime` (Luxon)
- **Signature**: `$now`
- **Description**: Current moment, respects the workflow's configured timezone. Equivalent to `DateTime.now()`.
- **Key methods**: `.format()`, `.toISO()`, `.plus()`, `.minus()`, `.toRelative()`, `.year`, `.month`, `.day`, `.hour`

### $today

- **Returns**: `DateTime` (Luxon)
- **Signature**: `$today`
- **Description**: Midnight at the start of the current day, respects the instance timezone.

---

## Node Execution Context

### $prevNode

| Property | Returns | Description |
|----------|---------|-------------|
| `$prevNode.name` | String | Name of the node that provided the current input |
| `$prevNode.outputIndex` | Number | Output connector index the input came from |
| `$prevNode.runIndex` | Number | Run index of the previous node that generated this input |

### $runIndex

- **Returns**: `Number`
- **Description**: How many times n8n has executed the current node in this workflow run (zero-based).

### $itemIndex

- **Returns**: `Number`
- **Description**: Position of the item currently being processed in the input list. **NOT available in the Code node.**

### $parameter

- **Returns**: `Object`
- **Description**: Configuration settings of the current node. Access individual settings via `$parameter.fieldName`.

### $nodeVersion

- **Returns**: `Number`
- **Description**: Version number of the current node type.

### $pageCount

- **Returns**: `Number`
- **Description**: Number of result pages fetched so far. **Only available in the HTTP Request node** when pagination is enabled.

---

## Utility Functions

### $ifEmpty()

- **Returns**: `any`
- **Signature**: `$ifEmpty(value: any, fallback: any)`
- **Description**: Returns `value` if it is non-empty; otherwise returns `fallback`. Empty values: `""`, `[]`, `{}`, `null`, `undefined`. Note: `0` and `false` are NOT considered empty.

### $jmespath()

- **Returns**: `any`
- **Signature**: `$jmespath(object: any, searchString: string)`
- **Description**: Performs a JMESPath query on a JSON object. **CRITICAL**: Parameter order is `(object, searchString)` — this differs from the JMESPath specification's `search(searchString, object)`.
- **Python variant**: `_jmespath(object, searchString)` (Code node only)

### $getWorkflowStaticData()

- **Returns**: `Object`
- **Signature**: `$getWorkflowStaticData(type: "global" | "node")`
- **Description**: Returns persistent static data that survives across workflow executions. `"global"` is accessible by every node; `"node"` is scoped to the current node. Modifications are saved automatically on successful execution. **NOT available during manual test executions.**

---

## HTTP Response Variables (HTTP Request Node Only)

### $response

| Property | Returns | Description |
|----------|---------|-------------|
| `$response.body` | Object | Body of the response from the last HTTP call |
| `$response.headers` | Object | Headers returned by the last HTTP call |
| `$response.statusCode` | Number | HTTP status code |
| `$response.statusMessage` | String | Optional status message |
