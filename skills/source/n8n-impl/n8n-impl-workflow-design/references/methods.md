# Core Workflow Design Nodes — Methods Reference

## 1. Execute Workflow Node

The Execute Workflow node calls another workflow as a sub-workflow, passing data in and receiving data back.

### Node Type
- **Type**: `n8n-nodes-base.executeWorkflow`
- **Category**: Core / Flow
- **Inputs**: 1 main
- **Outputs**: 1 main

### Key Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `source` | options | `database` (by ID) or `parameter` (inline JSON) |
| `workflowId` | workflowSelector | The workflow to execute (when source = database) |
| `workflowJson` | json | Inline workflow JSON (when source = parameter) |
| `mode` | options | `once` (run sub-workflow once with all items) or `each` (run per item) |
| `waitForSubWorkflow` | boolean | Wait for completion or fire-and-forget |

### Workflow Settings for Sub-Workflows

Configure these in the SUB-WORKFLOW's settings (not the caller):

```typescript
interface IWorkflowSettings {
    callerPolicy?: 'any' | 'none' | 'workflowsFromAList' | 'workflowsFromSameOwner';
    callerIds?: string; // Comma-separated workflow IDs
}
```

**callerPolicy values:**

| Policy | Description |
|--------|-------------|
| `workflowsFromSameOwner` | DEFAULT — only workflows from the same owner can call |
| `workflowsFromAList` | Only IDs in `callerIds` can call |
| `any` | Any workflow can call this sub-workflow |
| `none` | This workflow CANNOT be called as a sub-workflow |

**Environment variable default:**
`N8N_WORKFLOW_CALLER_POLICY_DEFAULT_OPTION` = `workflowsFromSameOwner`

### Data Flow

1. Caller sends input items to Execute Workflow node
2. Sub-workflow receives items at its trigger node
3. Sub-workflow processes and outputs items from its last node
4. Execute Workflow node receives the sub-workflow output as its own output

### Mode Behavior

- **once**: All input items are sent as a single array to the sub-workflow. Sub-workflow runs once.
- **each**: The sub-workflow runs once per input item. Results are collected into a single output array.

---

## 2. IF Node

The IF node evaluates conditions and routes items to true (output 0) or false (output 1).

### Node Type
- **Type**: `n8n-nodes-base.if`
- **Category**: Core / Flow
- **Inputs**: 1 main
- **Outputs**: 2 main (true, false)
- **Version**: 2

### Condition Structure

```json
{
    "conditions": {
        "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict"
        },
        "conditions": [
            {
                "id": "uuid",
                "leftValue": "={{ $json.status }}",
                "rightValue": "active",
                "operator": {
                    "type": "string",
                    "operation": "equals"
                }
            }
        ],
        "combinator": "and"
    }
}
```

### Available Operators

**String**: `equals`, `notEquals`, `contains`, `notContains`, `startsWith`, `notStartsWith`, `endsWith`, `notEndsWith`, `regex`, `notRegex`, `isEmpty`, `isNotEmpty`

**Number**: `equals`, `notEquals`, `gt`, `gte`, `lt`, `lte`, `between`, `notBetween`

**Boolean**: `true`, `false`, `equals`, `notEquals`

**Date/Time**: `after`, `before`, `equals`

**Array**: `contains`, `notContains`, `lengthEquals`, `lengthGt`, `lengthLt`, `isEmpty`, `isNotEmpty`

### Combinators

- `and` — ALL conditions must be true
- `or` — ANY condition must be true

---

## 3. Switch Node

The Switch node routes items to multiple outputs based on rules or expression values.

### Node Type
- **Type**: `n8n-nodes-base.switch`
- **Category**: Core / Flow
- **Inputs**: 1 main
- **Outputs**: Dynamic (based on rules/values)

### Routing Modes

**Rules mode**: Define condition rules per output. Each rule has the same condition format as the IF node. Items matching rule 0 go to output 0, rule 1 to output 1, etc.

**Expression mode**: Route based on an expression's value. Each output is associated with a specific value. Items are routed to the output whose value matches the expression result.

### Fallback Output

When no rule matches, items go to the **fallback output** (the last output). ALWAYS define a fallback to prevent silent data loss.

---

## 4. Merge Node

The Merge node combines data from two or more input branches.

### Node Type
- **Type**: `n8n-nodes-base.merge`
- **Category**: Core / Flow
- **Inputs**: 2+ main
- **Outputs**: 1 main

### Modes

#### Append
Concatenates all items from all inputs into a single output array.
- Input 1: `[A, B]`, Input 2: `[C, D]` → Output: `[A, B, C, D]`

#### Combine > Merge By Fields
Joins items from two inputs based on matching field values (SQL JOIN).

| Join Type | Behavior |
|-----------|----------|
| Inner Join | Only items with matches in BOTH inputs |
| Left Join | All items from Input 1 + matching from Input 2 |
| Right Join | All items from Input 2 + matching from Input 1 |
| Outer Join | All items from both inputs, matched where possible |

**Parameters:**
- `fieldsToMatchOn`: Field name(s) to match (e.g., `id`, `email`)
- `joinMode`: `keepMatches`, `keepNonMatches`, `enrichInput1`, `enrichInput2`
- `outputDataFrom`: `both`, `input1`, `input2`
- `multipleMatches.includeAllMatches`: Include all matches or first only

#### Combine > Merge By Position
Pairs items by their index position.
- Input 1: `[A, B]`, Input 2: `[X, Y]` → Output: `[A+X, B+Y]`

#### Combine > Multiplex
Creates all possible combinations (Cartesian product).
- Input 1: `[A, B]`, Input 2: `[X, Y]` → Output: `[A+X, A+Y, B+X, B+Y]`

#### Choose Branch
Waits for all inputs to complete, then outputs data from a selected input only.
- Use when parallel branches exist but only one result is needed.

#### SQL Query
Write SQL queries against the input data.
- Reference inputs as `input1`, `input2`, etc.
- Example: `SELECT * FROM input1 LEFT JOIN input2 ON input1.id = input2.userId`

---

## 5. Loop Over Items Node

The Loop Over Items node (formerly "Split In Batches") processes items in batches.

### Node Type
- **Type**: `n8n-nodes-base.splitInBatches`
- **Category**: Core / Flow
- **Inputs**: 1 main
- **Outputs**: 2 main (Loop body, Done)

### Key Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `batchSize` | number | 10 | Items per batch |
| `options.reset` | boolean | false | Reset the node (start over) |

### Output Behavior

- **Output 0 ("Loop")**: Emits the current batch of items. Connect to your processing nodes.
- **Output 1 ("Done")**: Emits ALL processed items after the last batch completes.

### Wiring Pattern

```
Loop Over Items ──[Loop]──→ Process Node(s) ──→ Loop Over Items
                 ──[Done]──→ Next workflow step
```

The processing chain MUST connect back to the Loop Over Items node input to continue iteration.

---

## 6. Wait Node

The Wait node pauses workflow execution and resumes based on a configured trigger.

### Node Type
- **Type**: `n8n-nodes-base.wait`
- **Category**: Core / Flow
- **Inputs**: 1 main
- **Outputs**: 1 main

### Resume Methods

#### Time-Based
- **After time interval**: Resume after N seconds/minutes/hours/days
- **At specified time**: Resume at a specific date and time

#### Webhook Resume
- Generates a unique URL: `$execution.resumeUrl`
- Execution pauses until an HTTP request hits the URL
- Request data becomes the Wait node's output
- Webhook path: `<base>/webhook-waiting/<path>`

#### Form Resume
- Generates a form URL: `$execution.resumeFormUrl`
- Displays a form to the user
- Execution resumes on form submission
- Form data becomes the Wait node's output

### Webhook Resume Authentication

The Wait node webhook supports the same authentication methods as the Webhook node:
- None
- Basic Auth
- Header Auth
- JWT Auth

---

## 7. Error Trigger Node

The Error Trigger node starts an error workflow when another workflow fails.

### Node Type
- **Type**: `n8n-nodes-base.errorTrigger`
- **Category**: Core / Trigger
- **Inputs**: None (trigger node)
- **Outputs**: 1 main

### Error Data Received

The Error Trigger node outputs items containing:

```json
{
    "execution": {
        "id": "execution-id",
        "url": "https://n8n.example.com/execution/123",
        "retryOf": "previous-execution-id",
        "error": {
            "message": "Error message",
            "name": "Error type"
        },
        "lastNodeExecuted": "Node Name",
        "mode": "trigger"
    },
    "workflow": {
        "id": "workflow-id",
        "name": "Workflow Name"
    }
}
```

### Setup

1. Create a new workflow with Error Trigger as the first node
2. Add notification/recovery nodes (Slack, email, database log, etc.)
3. In the MAIN workflow's Settings, set "Error Workflow" to this workflow

---

## 8. Stop And Error Node

The Stop And Error node deliberately terminates execution with an error, triggering the configured error workflow.

### Node Type
- **Type**: `n8n-nodes-base.stopAndError`
- **Category**: Core / Flow
- **Inputs**: 1 main
- **Outputs**: None (terminates execution)

### Key Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `errorType` | options | `errorMessage` (custom text) or `errorObject` (from previous node) |
| `errorMessage` | string | Custom error message (when errorType = errorMessage) |

### Usage

Place after an IF or Switch node to fail the workflow when validation conditions are not met:

```
IF (data valid?) ──[false]──→ Stop And Error ──→ triggers Error Workflow
                 ──[true]──→ Continue processing
```

---

## 9. Node-Level Error Settings

Every node in n8n has these error-related settings in its Settings tab:

### continueOnFail
- **Type**: boolean
- **Default**: false
- When enabled, the workflow continues even if this node fails
- Error data is passed to the next node in `$json.error`

### retryOnFail
- **Type**: boolean
- **Default**: false
- When enabled, the node retries on failure

### maxTries
- **Type**: number
- **Default**: 3
- Maximum number of retry attempts (only when retryOnFail is true)

### waitBetweenTries
- **Type**: number (ms)
- **Default**: 1000
- Delay between retry attempts in milliseconds

### onError
- **Type**: `'stopWorkflow' | 'continueRegularOutput' | 'continueErrorOutput'`
- **Default**: `stopWorkflow`
- Controls error routing behavior

### executeOnce
- **Type**: boolean
- **Default**: false
- When enabled, the node executes only for the first input item

### alwaysOutputData
- **Type**: boolean
- **Default**: false
- When enabled, outputs an empty item if the node produces no output
