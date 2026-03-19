# Workflow JSON Anti-Patterns

## Connection Format Errors

### WRONG: Using Node ID Instead of Name as Connection Key

```json
{
    "connections": {
        "uuid-trigger-001": {
            "main": [
                [{ "node": "uuid-set-001", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

**Why it fails**: Connections are keyed by node **name**, and target `node` fields reference node **name**. Using IDs produces a workflow where no connections are resolved.

```json
{
    "connections": {
        "Schedule Trigger": {
            "main": [
                [{ "node": "HTTP Request", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

---

### WRONG: Missing `type` or `index` in Connection Target

```json
{
    "connections": {
        "Trigger": {
            "main": [
                [{ "node": "Next Node" }]
            ]
        }
    }
}
```

**Why it fails**: Every `IConnection` object MUST have all three fields: `node`, `type`, and `index`. Missing fields cause runtime errors or undefined behavior.

```json
{
    "connections": {
        "Trigger": {
            "main": [
                [{ "node": "Next Node", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

---

### WRONG: Flat Connection Structure (Missing Nesting Levels)

```json
{
    "connections": {
        "Trigger": [
            { "node": "Next Node", "type": "main", "index": 0 }
        ]
    }
}
```

**Why it fails**: Missing the connection type level (`"main"`) and the output index level (array of arrays). Connections ALWAYS require exactly 3 nesting levels.

```json
{
    "connections": {
        "Trigger": {
            "main": [
                [
                    { "node": "Next Node", "type": "main", "index": 0 }
                ]
            ]
        }
    }
}
```

---

### WRONG: Two-Level Nesting (Missing Output Index Array)

```json
{
    "connections": {
        "Trigger": {
            "main": [
                { "node": "Next Node", "type": "main", "index": 0 }
            ]
        }
    }
}
```

**Why it fails**: The third level (output index) MUST be an array of arrays. Each inner array represents one output. Without it, n8n cannot determine which output the connection belongs to.

```json
{
    "connections": {
        "Trigger": {
            "main": [
                [
                    { "node": "Next Node", "type": "main", "index": 0 }
                ]
            ]
        }
    }
}
```

---

### WRONG: Using `"main"` for AI Sub-Node Connections

```json
{
    "connections": {
        "OpenAI Chat Model": {
            "main": [
                [{ "node": "AI Agent", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

**Why it fails**: AI sub-nodes (language models, tools, memory, etc.) MUST use their specific `ai_*` connection type. Using `"main"` means the AI agent never receives the model/tool/memory.

```json
{
    "connections": {
        "OpenAI Chat Model": {
            "ai_languageModel": [
                [{ "node": "AI Agent", "type": "ai_languageModel", "index": 0 }]
            ]
        }
    }
}
```

---

### WRONG: Mismatched Source and Target Connection Types

```json
{
    "connections": {
        "Calculator Tool": {
            "ai_tool": [
                [{ "node": "AI Agent", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

**Why it fails**: The source connection type (`ai_tool`) does not match the target connection type (`main`). The `type` field in the target MUST match the connection type declared by the target node's input. For AI sub-nodes, source and target types ALWAYS match.

```json
{
    "connections": {
        "Calculator Tool": {
            "ai_tool": [
                [{ "node": "AI Agent", "type": "ai_tool", "index": 0 }]
            ]
        }
    }
}
```

---

## Node Definition Errors

### WRONG: Duplicate Node Names

```json
{
    "nodes": [
        { "id": "uuid-001", "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest", ... },
        { "id": "uuid-002", "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest", ... }
    ]
}
```

**Why it fails**: Node names MUST be unique within a workflow. Connections reference nodes by name, so duplicates make connections ambiguous. n8n appends numbers to auto-generated duplicates (e.g., "HTTP Request1").

```json
{
    "nodes": [
        { "id": "uuid-001", "name": "Fetch Users", "type": "n8n-nodes-base.httpRequest", ... },
        { "id": "uuid-002", "name": "Fetch Orders", "type": "n8n-nodes-base.httpRequest", ... }
    ]
}
```

---

### WRONG: Missing Required Node Fields

```json
{
    "nodes": [
        {
            "name": "My Node",
            "type": "n8n-nodes-base.set"
        }
    ]
}
```

**Why it fails**: Every node MUST have `id`, `name`, `type`, `typeVersion`, `position`, and `parameters`. Missing fields cause import failures or runtime errors.

```json
{
    "nodes": [
        {
            "id": "uuid-set-001",
            "name": "My Node",
            "type": "n8n-nodes-base.set",
            "typeVersion": 3.4,
            "position": [450, 300],
            "parameters": {}
        }
    ]
}
```

---

### WRONG: Position as Object Instead of Tuple

```json
{
    "position": { "x": 250, "y": 300 }
}
```

**Why it fails**: Position MUST be a 2-element array `[x, y]`, not an object with named keys.

```json
{
    "position": [250, 300]
}
```

---

### WRONG: Referencing Non-Existent Node in Connections

```json
{
    "nodes": [
        { "id": "uuid-001", "name": "Trigger", ... }
    ],
    "connections": {
        "Trigger": {
            "main": [
                [{ "node": "Processing Node", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

**Why it fails**: "Processing Node" does not exist in the `nodes` array. ALWAYS verify that every node name referenced in connections exists as a node in the workflow.

---

## Settings Errors

### WRONG: Using Execution Order v0

```json
{
    "settings": {
        "executionOrder": "v0"
    }
}
```

**Why it is problematic**: The `v0` execution order algorithm has known edge cases with complex branching and parallel execution. ALWAYS use `"v1"` for new workflows.

```json
{
    "settings": {
        "executionOrder": "v1"
    }
}
```

---

### WRONG: Setting `active: true` Without a Trigger Node

```json
{
    "active": true,
    "nodes": [
        {
            "name": "Set",
            "type": "n8n-nodes-base.set",
            ...
        }
    ]
}
```

**Why it fails**: A workflow set to `active: true` expects a trigger node to start executions. Without a trigger, the workflow activates but never executes. ALWAYS include a trigger node (schedule, webhook, polling, or event-based) when setting `active: true`.

---

## Expression Errors

### WRONG: Expression Without Wrapper Syntax

```json
{
    "parameters": {
        "value": "$json.name"
    }
}
```

**Why it fails**: Expressions MUST be wrapped in `={{ }}` syntax. Without the wrapper, the string is treated as a literal value.

```json
{
    "parameters": {
        "value": "={{ $json.name }}"
    }
}
```

---

### WRONG: Using `$json` in Node With Multiple Inputs

When a node has multiple inputs (like Merge), `$json` refers to the first input only. Use `$input` with explicit references:

```json
{
    "parameters": {
        "value": "={{ $('Source Node Name').item.json.field }}"
    }
}
```

ALWAYS use `$('Node Name')` syntax when referencing data from a specific upstream node in complex workflows.
