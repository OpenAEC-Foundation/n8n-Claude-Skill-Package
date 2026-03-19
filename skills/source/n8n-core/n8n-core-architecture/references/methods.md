# n8n-core-architecture — Methods Reference

> Type signatures for core n8n v1.x architecture interfaces.

---

## INodeExecutionData

The fundamental data unit flowing between nodes.

```typescript
export interface INodeExecutionData {
    json: IDataObject;                                        // REQUIRED - main data payload
    binary?: IBinaryKeyData;                                  // Optional file/binary data
    error?: NodeApiError | NodeOperationError;                // Error info if item errored
    pairedItem?: IPairedItemData | IPairedItemData[] | number; // Item linking
    metadata?: {
        subExecution: RelatedExecution;                       // Sub-workflow execution reference
    };
    evaluationData?: Record<string, GenericValue>;
    redaction?: INodeExecutionRedactionInfo;                  // Data redaction marker
    sendMessage?: ChatNodeMessage;                            // Chat message to send
}

// IDataObject is a recursive key-value type:
export interface IDataObject {
    [key: string]: GenericValue;
}
type GenericValue =
    | string | number | boolean | null | undefined
    | IDataObject | IDataObject[]
    | GenericValue[];
```

---

## IBinaryData

Represents a single binary file attached to an item.

```typescript
export interface IBinaryData {
    data: string;                     // Base64 encoded binary data (or reference ID in filesystem mode)
    mimeType: string;                 // MIME type (e.g., "image/png", "application/pdf")
    fileType?: BinaryFileType;        // 'text' | 'json' | 'image' | 'audio' | 'video' | 'pdf' | 'html'
    fileName?: string;               // Original file name
    directory?: string;               // Directory path (if applicable)
    fileExtension?: string;           // File extension without dot
    fileSize?: string;                // Human-readable file size
    bytes?: number;                   // File size in bytes
    id?: string;                      // Reference ID for filesystem-stored binary
}

export type BinaryFileType = 'text' | 'json' | 'image' | 'audio' | 'video' | 'pdf' | 'html';
```

### IBinaryKeyData

Binary data on an item is keyed by property name:

```typescript
export interface IBinaryKeyData {
    [key: string]: IBinaryData;       // e.g., { "data": {...}, "attachment": {...} }
}
```

---

## IPairedItemData

Links an output item back to its source input item for data lineage tracking.

```typescript
export interface IPairedItemData {
    item: number;                     // Index of the source item in the input array
    input?: number;                   // Input index (default 0, for nodes with multiple inputs)
    sourceOverwrite?: ISourceData;    // Override source node reference
}

export interface ISourceData {
    previousNode: string;             // Source node name
    previousNodeOutput?: number;      // Source node output index
    previousNodeRun?: number;         // Source node run index
}
```

---

## IWorkflowSettings

Workflow-level configuration applied to all executions.

```typescript
export interface IWorkflowSettings {
    timezone?: 'DEFAULT' | string;                            // Timezone (e.g., "Europe/Amsterdam")
    errorWorkflow?: 'DEFAULT' | string;                       // Workflow ID to run on error
    callerIds?: string;                                       // Allowed caller workflow IDs (comma-separated)
    callerPolicy?: WorkflowSettings.CallerPolicy;             // Sub-workflow access policy
    saveDataErrorExecution?: WorkflowSettings.SaveDataExecution;   // Save policy for errors
    saveDataSuccessExecution?: WorkflowSettings.SaveDataExecution; // Save policy for success
    saveManualExecutions?: 'DEFAULT' | boolean;               // Save manual executions
    saveExecutionProgress?: 'DEFAULT' | boolean;              // Save progress during execution
    executionTimeout?: number;                                // Timeout in seconds
    executionOrder?: 'v0' | 'v1';                            // Node execution order algorithm
    binaryMode?: WorkflowSettingsBinaryMode;                  // Binary data storage mode
    timeSavedPerExecution?: number;                           // Estimated time saved per execution
    timeSavedMode?: 'fixed' | 'dynamic';                     // Time saved calculation mode
    availableInMCP?: boolean;                                 // MCP protocol availability
    credentialResolverId?: string;                            // Credential resolver ID
    redactionPolicy?: WorkflowSettings.RedactionPolicy;      // 'none' | 'all' | 'non-manual'
}
```

### CallerPolicy Values

| Policy | Description |
|--------|-------------|
| `'workflowsFromSameOwner'` | Only workflows from the same owner can call this sub-workflow |
| `'workflowsFromAList'` | Only workflows listed in `callerIds` can call this sub-workflow |
| `'any'` | Any workflow can call this sub-workflow |
| `'none'` | No workflow can call this sub-workflow |

### SaveDataExecution Values

| Value | Description |
|-------|-------------|
| `'DEFAULT'` | Use instance-level setting |
| `'all'` | Save all execution data |
| `'none'` | Do not save execution data |

---

## WorkflowExecuteMode

Union type defining how a workflow execution was triggered.

```typescript
type WorkflowExecuteModeValues =
    | 'cli'            // Command-line execution
    | 'error'          // Error workflow triggered
    | 'integrated'     // Sub-workflow called from another workflow
    | 'internal'       // Internal system execution
    | 'manual'         // User clicked "Execute Workflow" in editor
    | 'retry'          // Retrying a failed execution
    | 'trigger'        // Trigger node activated (schedule, poll)
    | 'webhook'        // Webhook received
    | 'evaluation'     // Evaluation/test execution
    | 'chat';          // Chat-based trigger
```

---

## IExecuteFunctions (Key Methods)

Available inside the `execute()` method of programmatic nodes via `this`.

### Data Access

```typescript
// Get all input items for the specified input index
getInputData(inputIndex?: number, connectionType?: NodeConnectionType): INodeExecutionData[];

// Get a node parameter value for a specific item
getNodeParameter(parameterName: string, itemIndex: number, fallbackValue?: any): NodeParameterValueType;

// Get credentials by name
getCredentials(type: string, itemIndex?: number): Promise<ICredentialDataDecryptedObject>;

// Get the current execution mode
getMode(): WorkflowExecuteMode;
```

### Binary Helpers

```typescript
// Assert binary data exists and return it (throws if missing)
this.helpers.assertBinaryData(itemIndex: number, propertyName: string | IBinaryData): IBinaryData;

// Get binary data as a Buffer
this.helpers.getBinaryDataBuffer(itemIndex: number, propertyName: string | IBinaryData): Promise<Buffer>;

// Detect encoding of binary text data
this.helpers.detectBinaryEncoding(buffer: Buffer): string;

// Prepare binary data from a Buffer
this.helpers.prepareBinaryData(buffer: Buffer, fileName?: string, mimeType?: string): Promise<IBinaryData>;
```

### Execution Control

```typescript
// Check if user enabled "Continue On Fail" for this node
continueOnFail(): boolean;

// Execute a sub-workflow
executeWorkflow(
    workflowInfo: IExecuteWorkflowInfo,
    inputData?: INodeExecutionData[],
    parentCallbackManager?: CallbackManager,
    options?: {
        doNotWaitToFinish?: boolean;
        parentExecution?: RelatedExecution;
        executionMode?: WorkflowExecuteMode;
    },
): Promise<ExecuteWorkflowData>;

// Put execution into wait state
putExecutionToWait(waitTill: Date): Promise<void>;

// Send a message to the UI
sendMessageToUI(message: any): void;
```

### Paired Item Helper

```typescript
// Construct execution metadata with paired item information
this.helpers.constructExecutionMetaData(
    inputData: INodeExecutionData[],
    options: { itemData: IPairedItemData | IPairedItemData[] }
): NodeExecutionWithMetadata[];
```

### HTTP Request Helper

```typescript
// Make an HTTP request (recommended over external libraries)
this.helpers.httpRequest(options: IHttpRequestOptions): Promise<any>;

// Make a request with full options (including pagination)
this.helpers.httpRequestWithAuthentication(
    credentialType: string,
    requestOptions: IHttpRequestOptions,
    additionalCredentialOptions?: IAdditionalCredentialOptions,
): Promise<any>;
```

---

## Expression Data Proxy (IWorkflowDataProxyData)

Variables available in n8n expressions during execution.

```typescript
export interface IWorkflowDataProxyData {
    $binary: INodeExecutionData['binary'];           // Current item binary data
    $data: any;                                       // Alias for $json
    $env: any;                                        // Environment variables
    $evaluateExpression: (expression: string, itemIndex?: number) => NodeParameterValueType;
    $item: (itemIndex: number, runIndex?: number) => IWorkflowDataProxyData;
    $items: (nodeName?: string, outputIndex?: number, runIndex?: number) => INodeExecutionData[];
    $json: INodeExecutionData['json'];                // Current item's JSON data
    $node: any;                                       // Access other nodes' data
    $parameter: INodeParameters;                      // Current node's parameters
    $position: number;                                // Current item index
    $workflow: any;                                    // Workflow metadata
    $: any;                                           // Shorthand for $node
    $input: ProxyInput;                               // Current node's input
    $thisItem: any;                                   // Current item reference
    $thisRunIndex: number;                            // Current run index
    $thisItemIndex: number;                           // Current item index
    $now: any;                                        // Current DateTime (Luxon)
    $today: any;                                      // Today's date (Luxon)
}

export interface ProxyInput {
    all: () => INodeExecutionData[];                  // All input items
    context: any;                                     // Execution context
    first: () => INodeExecutionData | undefined;      // First input item
    item: INodeExecutionData | undefined;             // Current input item
    last: () => INodeExecutionData | undefined;       // Last input item
    params?: INodeParameters;                         // Input parameters
}
```

---

## NodeConnectionType Values

```typescript
export const NodeConnectionTypes = {
    AiAgent: 'ai_agent',
    AiChain: 'ai_chain',
    AiDocument: 'ai_document',
    AiEmbedding: 'ai_embedding',
    AiLanguageModel: 'ai_languageModel',
    AiMemory: 'ai_memory',
    AiOutputParser: 'ai_outputParser',
    AiRetriever: 'ai_retriever',
    AiReranker: 'ai_reranker',
    AiTextSplitter: 'ai_textSplitter',
    AiTool: 'ai_tool',
    AiVectorStore: 'ai_vectorStore',
    Main: 'main',
} as const;
```

The `'main'` connection type is used for standard data flow. AI connection types are used for AI agent sub-nodes (language models, tools, memory, etc.).
