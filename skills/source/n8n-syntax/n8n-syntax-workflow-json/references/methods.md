# Workflow JSON Interface Reference

## IWorkflowBase

The complete workflow structure as stored and exchanged via the n8n API.

```typescript
export interface IWorkflowBase {
    id: string;                          // Unique workflow identifier
    name: string;                        // Human-readable name
    description?: string | null;         // Optional description
    active: boolean;                     // Whether triggers are listening
    isArchived: boolean;                 // Whether workflow is archived
    createdAt: Date;                     // Creation timestamp
    startedAt?: Date;                    // Last execution start
    updatedAt: Date;                     // Last modification timestamp
    nodes: INode[];                      // Array of all nodes
    connections: IConnections;           // Connection map between nodes
    settings?: IWorkflowSettings;        // Workflow-level configuration
    staticData?: IDataObject;            // Persistent data across executions
    pinData?: IPinData;                  // Pinned test data
    versionId?: string;                  // Version identifier
    activeVersionId: string | null;      // Active version reference
    activeVersion?: IWorkflowHistory | null;
    versionCounter?: number;             // Incremented on each save
    meta?: WorkflowFEMeta;              // Frontend metadata (template ID, etc.)
}
```

### Key Rules

- `active` controls whether trigger nodes listen for events. Set to `false` during development.
- `staticData` persists between executions. Polling triggers use this to track the last poll timestamp.
- `pinData` stores test data keyed by node name. Pinned nodes skip execution and return pinned data instead.
- `nodes` array order does NOT determine execution order -- connections determine the flow.

---

## INode

Represents a single node instance within a workflow.

```typescript
export interface INode {
    id: string;                         // UUID format (e.g., "a1b2c3d4-...")
    name: string;                       // Display name -- MUST be unique in workflow
    type: string;                       // Node type identifier
    typeVersion: number;                // Node version (determines available features)
    position: [number, number];         // [x, y] canvas coordinates
    parameters: INodeParameters;        // Node configuration values
    credentials?: INodeCredentials;     // Credential references
    disabled?: boolean;                 // Skip this node during execution
    notes?: string;                     // User-visible notes
    notesInFlow?: boolean;              // Display notes on canvas
    retryOnFail?: boolean;              // Auto-retry on error
    maxTries?: number;                  // Maximum retry attempts (default: 3)
    waitBetweenTries?: number;          // Milliseconds between retries (default: 1000)
    alwaysOutputData?: boolean;         // Output empty item when no data produced
    executeOnce?: boolean;              // Process only the first input item
    onError?: OnError;                  // Error handling strategy
    continueOnFail?: boolean;           // Continue workflow even if this node errors
    webhookId?: string;                 // Webhook identifier (webhook nodes only)
}

export type OnError = 'continueErrorOutput' | 'continueRegularOutput' | 'stopWorkflow';
```

### Node Type Format

Node types follow the pattern `{package}.{nodeName}`:

| Package | Example | Description |
|---------|---------|-------------|
| `n8n-nodes-base` | `n8n-nodes-base.httpRequest` | Built-in nodes |
| `n8n-nodes-base` | `n8n-nodes-base.scheduleTrigger` | Built-in triggers |
| `@n8n/n8n-nodes-langchain` | `@n8n/n8n-nodes-langchain.agent` | AI/LangChain nodes |
| `n8n-nodes-{name}` | `n8n-nodes-slack.slack` | Community nodes |

### INodeParameters

Parameters are a nested key-value structure. Values can be strings, numbers, booleans, objects, or arrays. Expression values use the format `={{ expression }}`.

```typescript
export type INodeParameters = Record<string, NodeParameterValueType>;

// Example parameters for HTTP Request node
{
    "method": "POST",
    "url": "https://api.example.com/data",
    "authentication": "genericCredentialType",
    "sendBody": true,
    "bodyParameters": {
        "parameters": [
            {"name": "key", "value": "={{ $json.id }}"}
        ]
    }
}
```

### INodeCredentials

```typescript
export interface INodeCredentials {
    [credentialType: string]: INodeCredentialDescription;
}

// In workflow JSON, credentials reference stored credentials by name and ID:
{
    "credentials": {
        "httpBasicAuth": {
            "id": "cred_123",
            "name": "My API Credentials"
        }
    }
}
```

---

## IConnections (3-Level Nesting)

The connection map defines how data flows between nodes. ALWAYS use this exact 3-level structure.

```typescript
// Level 1: Keyed by SOURCE node name (string)
export interface IConnections {
    [sourceNodeName: string]: INodeConnections;
}

// Level 2: Keyed by connection type (string)
export interface INodeConnections {
    [connectionType: string]: NodeInputConnections;
}

// Level 3: Indexed by output index (array position)
// Each element is an array of target connections, or null for unused outputs
export type NodeInputConnections = Array<IConnection[] | null>;

// Individual connection target
export interface IConnection {
    node: string;                       // Destination node NAME (not ID)
    type: NodeConnectionType;           // Connection type at the destination
    index: number;                      // Destination input index
}
```

### Connection Reading Guide

To read: "Source node **X** sends data from connection type **Y**, output index **Z**, to targets **[...]**"

```json
{
    "Schedule Trigger": {           // Source node name
        "main": [                   // Connection type
            [                       // Output index 0
                {
                    "node": "HTTP Request",  // Target node name
                    "type": "main",          // Target input type
                    "index": 0               // Target input index
                }
            ]
        ]
    }
}
```

### Null Entries in Output Array

When a multi-output node only connects some outputs, use `null` for unconnected positions:

```json
{
    "Switch": {
        "main": [
            [{"node": "Handler A", "type": "main", "index": 0}],
            null,
            [{"node": "Handler C", "type": "main", "index": 0}]
        ]
    }
}
```

This means: output 0 connects to "Handler A", output 1 is unconnected, output 2 connects to "Handler C".

---

## NodeConnectionType

All valid connection types as defined in the `NodeConnectionTypes` constant:

```typescript
export const NodeConnectionTypes = {
    Main: 'main',                    // Standard data flow between nodes
    AiAgent: 'ai_agent',            // AI agent sub-node
    AiChain: 'ai_chain',            // AI chain sub-node
    AiDocument: 'ai_document',      // Document loader for AI
    AiEmbedding: 'ai_embedding',    // Embedding model for AI
    AiLanguageModel: 'ai_languageModel',  // LLM for AI nodes
    AiMemory: 'ai_memory',          // Memory store for AI
    AiOutputParser: 'ai_outputParser',    // Output parser for AI
    AiRetriever: 'ai_retriever',    // Retriever for RAG
    AiReranker: 'ai_reranker',      // Reranker for search results
    AiTextSplitter: 'ai_textSplitter',   // Text splitter for documents
    AiTool: 'ai_tool',              // Tool for AI agents
    AiVectorStore: 'ai_vectorStore', // Vector store for embeddings
} as const;

export type NodeConnectionType = (typeof NodeConnectionTypes)[keyof typeof NodeConnectionTypes];
```

### AI Connection Type Usage

AI nodes declare their inputs and outputs using specific connection types:

```typescript
// AI Agent node declares typed inputs
inputs: [
    NodeConnectionTypes.Main,
    { type: NodeConnectionTypes.AiLanguageModel, required: true },
    { type: NodeConnectionTypes.AiMemory },
    { type: NodeConnectionTypes.AiTool },
    { type: NodeConnectionTypes.AiOutputParser },
]
```

The connection type in the connections map MUST match the input type declared by the target node.

---

## IWorkflowSettings

```typescript
export interface IWorkflowSettings {
    timezone?: 'DEFAULT' | string;
    errorWorkflow?: 'DEFAULT' | string;
    callerIds?: string;
    callerPolicy?: WorkflowSettings.CallerPolicy;
    saveDataErrorExecution?: WorkflowSettings.SaveDataExecution;
    saveDataSuccessExecution?: WorkflowSettings.SaveDataExecution;
    saveManualExecutions?: 'DEFAULT' | boolean;
    saveExecutionProgress?: 'DEFAULT' | boolean;
    executionTimeout?: number;
    executionOrder?: 'v0' | 'v1';
    binaryMode?: WorkflowSettingsBinaryMode;
    availableInMCP?: boolean;
    redactionPolicy?: WorkflowSettings.RedactionPolicy;
}
```

### CallerPolicy Values

| Value | Meaning |
|-------|---------|
| `workflowsFromSameOwner` | Only workflows owned by the same user can call this |
| `workflowsFromAList` | Only workflows listed in `callerIds` can call this |
| `any` | Any workflow can call this as a sub-workflow |
| `none` | Cannot be called as a sub-workflow |

### SaveDataExecution Values

| Value | Meaning |
|-------|---------|
| `all` | Save all execution data |
| `none` | Do not save execution data |
| `DEFAULT` | Use instance-level setting |

### RedactionPolicy Values

| Value | Meaning |
|-------|---------|
| `none` | No redaction |
| `all` | Redact all execution data |
| `non-manual` | Redact only production execution data |
