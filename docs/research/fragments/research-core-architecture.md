# n8n Core Architecture Research

> **Source**: n8n v1.x official documentation + GitHub source code (`n8n-io/n8n` repository)
> **Date**: 2026-03-19
> **Scope**: Execution model, INodeType interface, node properties, trigger nodes, workflow JSON, custom node development

---

## 1. Execution Model

### 1.1 Execution Modes (WorkflowExecuteMode)

n8n defines execution modes as a union type in `execution-context.ts`:

```typescript
type WorkflowExecuteModeValues =
  | 'cli'        // Command-line execution
  | 'error'      // Error workflow triggered
  | 'integrated' // Sub-workflow called from another workflow
  | 'internal'   // Internal system execution
  | 'manual'     // User clicked "Execute Workflow" in editor
  | 'retry'      // Retrying a failed execution
  | 'trigger'    // Trigger node activated (schedule, poll)
  | 'webhook'    // Webhook received
  | 'evaluation' // Evaluation/test execution
  | 'chat';      // Chat-based trigger
```

### 1.2 Data Flow: Items Through Nodes

n8n processes data as **items**. Each node receives an array of items, processes them, and outputs an array of items.

- **Input**: `INodeExecutionData[]` — an array of items
- **Output**: `INodeExecutionData[][]` — a 2D array where the first dimension is the output index (for nodes with multiple outputs like IF/Switch), and the second dimension is the array of items for that output
- **Each item** has a `json` property (the main data) and optional `binary` property (file data)

```typescript
// The execute() method receives items via this.getInputData()
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData(); // INodeExecutionData[]

    const returnData: INodeExecutionData[] = [];
    for (let i = 0; i < items.length; i++) {
        const item = items[i];
        // item.json contains the data
        // item.binary contains file data (optional)
        returnData.push({ json: { processed: true, ...item.json } });
    }

    return [returnData]; // Single output: [[item1, item2, ...]]
}
```

### 1.3 INodeExecutionData Structure

```typescript
export interface INodeExecutionData {
    json: IDataObject;                    // REQUIRED - the main data payload
    binary?: IBinaryKeyData;              // Optional file/binary data
    error?: NodeApiError | NodeOperationError;  // Error info if item errored
    pairedItem?: IPairedItemData | IPairedItemData[] | number;  // Item linking
    metadata?: {
        subExecution: RelatedExecution;   // Sub-workflow execution reference
    };
    evaluationData?: Record<string, GenericValue>;
    redaction?: INodeExecutionRedactionInfo;  // Data redaction marker
    sendMessage?: ChatNodeMessage;        // Chat message to send
}

export interface IBinaryKeyData {
    [key: string]: IBinaryData;  // Keyed by property name (e.g., "data", "attachment")
}

export interface IBinaryData {
    data: string;          // Base64 encoded binary data (or reference ID)
    mimeType: string;      // MIME type
    fileType?: BinaryFileType;  // 'text' | 'json' | 'image' | 'audio' | 'video' | 'pdf' | 'html'
    fileName?: string;
    directory?: string;
    fileExtension?: string;
    fileSize?: string;
    bytes?: number;
    id?: string;           // Reference ID for filesystem-stored binary
}

export interface IPairedItemData {
    item: number;          // Index of the source item
    input?: number;        // Input index (default 0)
    sourceOverwrite?: ISourceData;
}
```

### 1.4 Execution Data Proxy (Expression Variables)

During execution, nodes have access to a rich set of variables through the data proxy:

```typescript
export interface IWorkflowDataProxyData {
    $binary: INodeExecutionData['binary'];  // Current item binary data
    $data: any;                              // Alias for $json
    $env: any;                               // Environment variables
    $evaluateExpression: (expression: string, itemIndex?: number) => NodeParameterValueType;
    $item: (itemIndex: number, runIndex?: number) => IWorkflowDataProxyData;
    $items: (nodeName?: string, outputIndex?: number, runIndex?: number) => INodeExecutionData[];
    $json: INodeExecutionData['json'];       // Current item's JSON data
    $node: any;                              // Access other nodes' data
    $parameter: INodeParameters;             // Current node's parameters
    $position: number;                       // Current item index
    $workflow: any;                           // Workflow metadata
    $: any;                                  // Shorthand for $node
    $input: ProxyInput;                      // Current node's input
    $thisItem: any;                          // Current item reference
    $thisRunIndex: number;                   // Current run index
    $thisItemIndex: number;                  // Current item index
    $now: any;                               // Current DateTime (Luxon)
    $today: any;                             // Today's date (Luxon)
}

export interface ProxyInput {
    all: () => INodeExecutionData[];                // All input items
    context: any;                                    // Execution context
    first: () => INodeExecutionData | undefined;     // First input item
    item: INodeExecutionData | undefined;            // Current input item
    last: () => INodeExecutionData | undefined;      // Last input item
    params?: INodeParameters;                        // Input parameters
}
```

**$execution context** available in expressions:

```typescript
$execution?: {
    id: string;                    // Execution ID
    mode: 'test' | 'production';   // Whether manual or production
    resumeUrl: string;             // URL to resume waiting executions
    resumeFormUrl: string;         // URL for form-based resume
    customData?: {
        set(key: string, value: string): void;
        setAll(obj: Record<string, string>): void;
        get(key: string): string;
        getAll(): Record<string, string>;
    };
};
```

### 1.5 Workflow Settings

```typescript
export interface IWorkflowSettings {
    timezone?: 'DEFAULT' | string;
    errorWorkflow?: 'DEFAULT' | string;              // Workflow to run on error
    callerIds?: string;                              // Allowed caller workflow IDs
    callerPolicy?: WorkflowSettings.CallerPolicy;   // Sub-workflow access policy
    saveDataErrorExecution?: WorkflowSettings.SaveDataExecution;
    saveDataSuccessExecution?: WorkflowSettings.SaveDataExecution;
    saveManualExecutions?: 'DEFAULT' | boolean;
    saveExecutionProgress?: 'DEFAULT' | boolean;
    executionTimeout?: number;                       // Timeout in seconds
    executionOrder?: 'v0' | 'v1';                   // Node execution order algorithm
    binaryMode?: WorkflowSettingsBinaryMode;
    timeSavedPerExecution?: number;
    timeSavedMode?: 'fixed' | 'dynamic';
    availableInMCP?: boolean;                        // MCP protocol availability
    credentialResolverId?: string;
    redactionPolicy?: WorkflowSettings.RedactionPolicy;  // 'none' | 'all' | 'non-manual'
}
```

### 1.6 Queue Mode

n8n supports **queue mode** for horizontal scaling:
- Uses **BullMQ** (Redis-backed) for job queue management
- **Main process**: Handles webhooks, triggers, and UI; enqueues execution jobs
- **Worker processes**: Dequeue and execute workflows
- Configuration via environment variables:
  - `EXECUTIONS_MODE=queue`
  - `QUEUE_BULL_REDIS_HOST`, `QUEUE_BULL_REDIS_PORT`, etc.
  - `QUEUE_WORKER_CONCURRENCY` — number of concurrent executions per worker

---

## 2. INodeType Interface

### 2.1 Complete INodeType Interface

```typescript
export interface INodeType {
    // REQUIRED: Node description (metadata, properties, credentials)
    description: INodeTypeDescription;

    // Main execution method for regular (non-trigger) nodes
    execute?(this: IExecuteFunctions, response?: EngineResponse): Promise<NodeOutput>;

    // Called when node receives a chat message
    onMessage?(context: IExecuteFunctions, data: INodeExecutionData): Promise<NodeOutput>;

    // For polling trigger nodes
    poll?(this: IPollFunctions): Promise<INodeExecutionData[][] | null>;

    // For event-based trigger nodes
    trigger?(this: ITriggerFunctions): Promise<ITriggerResponse | undefined>;

    // For webhook-based nodes
    webhook?(this: IWebhookFunctions): Promise<IWebhookResponseData>;

    // For AI sub-nodes that supply data to AI agent nodes
    supplyData?(this: ISupplyDataFunctions, itemIndex: number): Promise<SupplyData>;

    // Dynamic methods (load options, credential test, resource mapping)
    methods?: {
        loadOptions?: {
            [key: string]: (this: ILoadOptionsFunctions) => Promise<INodePropertyOptions[]>;
        };
        listSearch?: {
            [key: string]: (
                this: ILoadOptionsFunctions,
                filter?: string,
                paginationToken?: string,
            ) => Promise<INodeListSearchResult>;
        };
        credentialTest?: {
            [functionName: string]: ICredentialTestFunction;
        };
        resourceMapping?: {
            [functionName: string]: (this: ILoadOptionsFunctions) => Promise<ResourceMapperFields>;
        };
        localResourceMapping?: {
            [functionName: string]: (this: ILocalLoadOptionsFunctions) => Promise<ResourceMapperFields>;
        };
        actionHandler?: {
            [functionName: string]: (
                this: ILoadOptionsFunctions,
                payload: IDataObject | string | undefined,
            ) => Promise<NodeParameterValueType>;
        };
    };

    // Webhook lifecycle methods (for nodes that register external webhooks)
    webhookMethods?: {
        [name in WebhookType]?: {
            [method in WebhookSetupMethodNames]: (this: IHookFunctions) => Promise<boolean>;
        };
    };

    // Custom operations for declarative nodes
    customOperations?: {
        [resource: string]: {
            [operation: string]: (this: IExecuteFunctions) => Promise<NodeOutput>;
        };
    };
}

// Return type of execute()
export type NodeOutput =
    | INodeExecutionData[][]           // Standard: array of outputs, each with array of items
    | NodeExecutionWithMetadata[][]    // With metadata
    | EngineRequest                    // Engine action request
    | null;

// WebhookType and setup methods
export type WebhookType = 'default' | 'setup';
export type WebhookSetupMethodNames = 'checkExists' | 'create' | 'delete';
```

### 2.2 New Node Base Class (Alternative)

n8n also provides an abstract `Node` class as a newer API:

```typescript
export abstract class Node {
    abstract description: INodeTypeDescription;
    execute?(context: IExecuteFunctions, response?: EngineResponse): Promise<INodeExecutionData[][] | EngineRequest>;
    webhook?(context: IWebhookFunctions): Promise<IWebhookResponseData>;
    poll?(context: IPollFunctions): Promise<INodeExecutionData[][] | null>;
}
```

Key difference: The `Node` class passes `context` as a parameter instead of using `this` binding.

### 2.3 Versioned Nodes

```typescript
export interface IVersionedNodeType {
    nodeVersions: {
        [key: number]: INodeType;    // Map version number to node implementation
    };
    currentVersion: number;          // Latest version
    description: INodeTypeBaseDescription;  // Shared base description
    getNodeType: (version?: number) => INodeType;
}
```

### 2.4 IExecuteFunctions (Available in execute())

```typescript
export type IExecuteFunctions = ExecuteFunctions.GetNodeParameterFn &
    BaseExecutionFunctions & {
        // Sub-workflow execution
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

        // Data access
        getInputData(inputIndex?: number, connectionType?: NodeConnectionType): INodeExecutionData[];
        getInputConnectionData(connectionType: AINodeConnectionType, itemIndex: number, inputIndex?: number): Promise<unknown>;
        getNodeInputs(): INodeInputConfiguration[];
        getNodeOutputs(): INodeOutputConfiguration[];
        getExecutionDataById(executionId: string): Promise<IRunExecutionData | undefined>;

        // Execution control
        putExecutionToWait(waitTill: Date): Promise<void>;
        sendMessageToUI(message: any): void;
        sendResponse(response: IExecuteResponsePromiseData): void;

        // AI / Tool support
        isToolExecution(): boolean;

        // Execution data tracking
        addInputData(connectionType: NodeConnectionType, data: INodeExecutionData[][] | ExecutionError, runIndex?: number): { index: number };
        addOutputData(connectionType: NodeConnectionType, currentNodeRunIndex: number, data: INodeExecutionData[][] | ExecutionError, metadata?: ITaskMetadata, sourceNodeRunIndex?: number): void;

        // Hints
        addExecutionHints(...hints: NodeExecutionHint[]): void;

        // Node helpers
        nodeHelpers: NodeHelperFunctions;
        helpers: RequestHelperFunctions &
            BaseHelperFunctions &
            BinaryHelperFunctions &
            DeduplicationHelperFunctions &
            FileSystemHelperFunctions &
            SSHTunnelFunctions &
            DataTableProxyFunctions & {
                normalizeItems(items: INodeExecutionData | INodeExecutionData[]): INodeExecutionData[];
                constructExecutionMetaData(inputData: INodeExecutionData[], options: { itemData: IPairedItemData | IPairedItemData[] }): NodeExecutionWithMetadata[];
                assertBinaryData(itemIndex: number, parameterData: string | IBinaryData): IBinaryData;
                getBinaryDataBuffer(itemIndex: number, parameterData: string | IBinaryData): Promise<Buffer>;
                detectBinaryEncoding(buffer: Buffer): string;
                copyInputItems(items: INodeExecutionData[], properties: string[]): IDataObject[];
            };

        // Job runner
        startJob<T = unknown, E = unknown>(jobType: string, settings: unknown, itemIndex: number): Promise<Result<T, E>>;
        getRunnerStatus(taskType: string): { available: true } | { available: false; reason?: string };
    };
```

---

## 3. Node Properties (INodeProperties)

### 3.1 Complete INodeProperties Interface

```typescript
export interface INodeProperties {
    displayName: string;                    // Label shown in UI
    name: string;                           // Internal property name (used in code)
    type: NodePropertyTypes;                // Property type (see below)
    typeOptions?: INodePropertyTypeOptions; // Type-specific options
    default: NodeParameterValueType;        // Default value
    description?: string;                   // Help text shown below property
    hint?: string;                          // Additional hint text
    builderHint?: IParameterBuilderHint;    // Hint for workflow SDK
    disabledOptions?: IDisplayOptions;      // Conditions to disable
    displayOptions?: IDisplayOptions;       // Conditions to show/hide
    options?: Array<INodePropertyOptions | INodeProperties | INodePropertyCollection>;
    placeholder?: string;                   // Placeholder text
    isNodeSetting?: boolean;                // Whether this is a node setting
    noDataExpression?: boolean;             // Disable expression support
    required?: boolean;                     // Whether required
    routing?: INodePropertyRouting;         // Declarative API routing
    credentialTypes?: Array<'extends:oAuth2Api' | 'extends:oAuth1Api' | 'has:authenticate' | 'has:genericAuth'>;
    extractValue?: INodePropertyValueExtractor;
    modes?: INodePropertyMode[];            // For resourceLocator
    requiresDataPath?: 'single' | 'multiple';
    doNotInherit?: boolean;
    validateType?: FieldType;               // Expected type for validation
    ignoreValidationDuringExecution?: boolean;
    allowArbitraryValues?: boolean;         // Skip options validation
    resolvableField?: boolean;              // For credential resolution
}
```

### 3.2 All Property Types (NodePropertyTypes)

```typescript
export type NodePropertyTypes =
    | 'boolean'              // Toggle switch
    | 'button'               // Action button
    | 'collection'           // Group of optional fields
    | 'color'                // Color picker
    | 'dateTime'             // Date/time picker
    | 'fixedCollection'      // Group of fields with fixed structure
    | 'hidden'               // Hidden value
    | 'icon'                 // Icon selector
    | 'json'                 // JSON editor
    | 'callout'              // Info/warning callout
    | 'notice'               // Notice/info text
    | 'multiOptions'         // Multi-select dropdown
    | 'number'               // Number input
    | 'options'              // Single-select dropdown
    | 'string'               // Text input
    | 'credentialsSelect'    // Credential selector
    | 'resourceLocator'      // Resource locator (ID, URL, list)
    | 'curlImport'           // cURL import
    | 'resourceMapper'       // Resource field mapper
    | 'filter'               // Filter/condition builder
    | 'assignmentCollection' // Field assignment collection
    | 'credentials'          // Credentials property
    | 'workflowSelector';   // Workflow selector
```

### 3.3 INodePropertyTypeOptions (Type-Specific Options)

```typescript
export interface INodePropertyTypeOptions {
    // Button
    buttonConfig?: {
        action?: string | NodePropertyAction;
        label?: string;
        hasInputField?: boolean;
        inputFieldMaxLength?: number;
    };

    // Notice
    containerClass?: string;

    // JSON
    alwaysOpenEditWindow?: boolean;

    // String (code editor)
    codeAutocomplete?: 'function' | 'functionItem';
    editor?: 'codeNodeEditor' | 'jsEditor' | 'htmlEditor' | 'sqlEditor' | 'cssEditor';
    editorIsReadOnly?: boolean;
    sqlDialect?: SQLDialect;

    // Options (dynamic loading)
    loadOptionsDependsOn?: string[];
    loadOptionsMethod?: string;
    loadOptions?: ILoadOptions;

    // Number
    maxValue?: number;
    minValue?: number;
    numberPrecision?: number;

    // All types
    multipleValues?: boolean;
    multipleValueButtonText?: string;

    // fixedCollection
    fixedCollection?: {
        itemTitle?: string;
        layout?: 'inline';
    };
    minRequiredFields?: number;
    maxAllowedFields?: number;
    hideOptionalFields?: boolean;
    addOptionalFieldButtonText?: string;
    showEvenWhenOptional?: boolean;

    // String
    password?: boolean;
    rows?: number;                   // Textarea rows

    // Color
    showAlpha?: boolean;

    // Sortable (with multipleValues)
    sortable?: boolean;

    // Hidden (credentials)
    expirable?: boolean;

    // DateTime
    dateOnly?: boolean;

    // Resource mapper
    resourceMapper?: ResourceMapperTypeOptions;

    // Filter
    filter?: FilterTypeOptions;

    // Assignment
    assignment?: AssignmentTypeOptions;

    // Callout
    calloutAction?: CalloutAction;

    // Binary data indicator
    binaryDataProperty?: boolean;

    [key: string]: any;  // Extensible
}
```

### 3.4 IDisplayOptions (Conditional Show/Hide)

```typescript
export interface IDisplayOptions {
    hide?: {
        [key: string]: Array<NodeParameterValue | DisplayCondition> | undefined;
    };
    show?: {
        '@version'?: Array<number | DisplayCondition>;   // Show for specific node versions
        '@feature'?: Array<string | DisplayCondition>;   // Show for specific features
        '@tool'?: boolean[];                             // Show when used as AI tool
        [key: string]: Array<NodeParameterValue | DisplayCondition> | undefined;
    };
    hideOnCloud?: boolean;  // Hide on n8n Cloud
}

// Display conditions support rich comparisons
export type DisplayCondition =
    | { _cnd: { eq: NodeParameterValue } }
    | { _cnd: { not: NodeParameterValue } }
    | { _cnd: { gte: number | string } }
    | { _cnd: { lte: number | string } }
    | { _cnd: { gt: number | string } }
    | { _cnd: { lt: number | string } }
    | { _cnd: { between: { from: number | string; to: number | string } } }
    | { _cnd: { startsWith: string } }
    | { _cnd: { endsWith: string } }
    | { _cnd: { includes: string } }
    | { _cnd: { regex: string } }
    | { _cnd: { exists: true } };
```

**Example: Conditional display based on parameter values**

```typescript
{
    displayName: 'Limit',
    name: 'limit',
    type: 'number',
    default: 50,
    typeOptions: { minValue: 1 },
    displayOptions: {
        show: {
            operation: ['getAll'],        // Show only when operation is 'getAll'
            returnAll: [false],           // AND returnAll is false
        },
    },
    description: 'Max number of results to return',
}
```

### 3.5 INodePropertyOptions (Dropdown Options)

```typescript
export interface INodePropertyOptions {
    name: string;                        // Display label
    value: string | number | boolean;    // Internal value
    action?: string;                     // Action description for UI
    description?: string;                // Tooltip/description
    builderHint?: IParameterBuilderHint;
    routing?: INodePropertyRouting;      // Declarative routing for this option
    outputConnectionType?: NodeConnectionType;
    inputSchema?: any;
    displayOptions?: IDisplayOptions;
}
```

### 3.6 INodePropertyCollection (Fixed Collection Values)

```typescript
export interface INodePropertyCollection {
    displayName: string;
    name: string;
    values: INodeProperties[];           // Sub-properties in this collection
    builderHint?: IParameterBuilderHint;
}
```

### 3.7 Property Routing (Declarative Nodes)

```typescript
export interface INodePropertyRouting {
    operations?: IN8nRequestOperations;
    output?: INodeRequestOutput;
    request?: DeclarativeRestApiSettings.HttpRequestOptions;
    send?: INodeRequestSend;
}

export interface INodeRequestSend {
    preSend?: PreSendAction[];
    paginate?: boolean | string;
    property?: string;
    propertyInDotNotation?: boolean;
    type?: 'body' | 'query';
    value?: string;
}

export interface INodeRequestOutput {
    maxResults?: number | string;
    postReceive?: PostReceiveAction[];
}
```

---

## 4. Trigger Nodes

### 4.1 Types of Trigger Nodes

n8n has three distinct trigger patterns:

1. **Event/Timer Triggers** — Use `trigger()` method with `ITriggerFunctions`
2. **Polling Triggers** — Use `poll()` method with `IPollFunctions`
3. **Webhook Triggers** — Use `webhook()` method with `IWebhookFunctions`

### 4.2 ITriggerFunctions Interface

```typescript
export interface ITriggerFunctions
    extends FunctionsBaseWithRequiredKeys<'getMode' | 'getActivationMode'> {

    // Emit data to start workflow execution
    emit(
        data: INodeExecutionData[][],
        responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>,
        donePromise?: IDeferredPromise<IRun>,
    ): void;

    // Report non-fatal execution error (workflow stays active)
    saveFailedExecution(error: ExecutionError): void;

    // Report fatal error (deactivates workflow, queues for reactivation with backoff)
    emitError(error: Error, responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>): void;

    // Parameter access
    getNodeParameter(
        parameterName: string,
        fallbackValue?: any,
        options?: IGetNodeParameterOptions,
    ): NodeParameterValueType | object;

    // Helpers
    helpers: RequestHelperFunctions &
        BaseHelperFunctions &
        BinaryHelperFunctions &
        SSHTunnelFunctions &
        SchedulingFunctions;
}
```

### 4.3 ITriggerResponse

```typescript
export interface ITriggerResponse {
    closeFunction?: CloseFunction;          // Cleanup function (stop listeners, etc.)
    manualTriggerFunction?: () => Promise<void>;  // Function to simulate trigger in manual mode
    manualTriggerResponse?: Promise<INodeExecutionData[][]>;  // Promise that resolves with emitted data
}
```

### 4.4 Trigger Node Pattern (Schedule Trigger Example)

```typescript
export class ScheduleTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Schedule Trigger',
        name: 'scheduleTrigger',
        icon: 'fa:clock',
        group: ['trigger', 'schedule'],
        version: [1, 1.1, 1.2, 1.3],
        description: 'Triggers the workflow on a given schedule',
        eventTriggerDescription: '',
        activationMessage: 'Your schedule trigger will now trigger executions on the schedule you have defined.',
        defaults: { name: 'Schedule Trigger' },
        inputs: [],                          // Trigger nodes have NO inputs
        outputs: [NodeConnectionTypes.Main], // One main output
        properties: [ /* ... */ ],
    };

    async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
        const executeTrigger = () => {
            const resultData = { timestamp: new Date().toISOString() };
            this.emit([this.helpers.returnJsonArray([resultData])]);
        };

        if (this.getMode() !== 'manual') {
            // Production mode: register cron job
            this.helpers.registerCron(cron, () => executeTrigger());
            return {};  // Empty response - cleanup handled by cron system
        } else {
            // Manual mode: provide function to simulate trigger
            const manualTriggerFunction = async () => {
                executeTrigger();
            };
            return { manualTriggerFunction };
        }
    }
}
```

### 4.5 IWebhookFunctions Interface

```typescript
export interface IWebhookFunctions extends FunctionsBaseWithRequiredKeys<'getMode'> {
    getBodyData(): IDataObject;
    getHeaderData(): IncomingHttpHeaders;
    getInputConnectionData(connectionType: AINodeConnectionType, itemIndex: number, inputIndex?: number): Promise<unknown>;
    getNodeParameter(parameterName: string, fallbackValue?: any, options?: IGetNodeParameterOptions): NodeParameterValueType | object;
    getNodeWebhookUrl: (name: WebhookType) => string | undefined;
    evaluateExpression(expression: string, itemIndex?: number): NodeParameterValueType;
    getParamsData(): object;
    getQueryData(): object;
    getRequestObject(): express.Request;
    getResponseObject(): express.Response;
    getWebhookName(): string;
    validateCookieAuth(cookieValue: string): Promise<void>;
    nodeHelpers: NodeHelperFunctions;
    helpers: RequestHelperFunctions & BaseHelperFunctions & BinaryHelperFunctions;
}
```

### 4.6 IWebhookResponseData

```typescript
export interface IWebhookResponseData {
    workflowData?: INodeExecutionData[][];   // Data to pass to workflow
    webhookResponse?: any;                    // Response to send to webhook caller
    noWebhookResponse?: boolean;             // If true, response was already sent
}

export type WebhookResponseData = 'allEntries' | 'firstEntryJson' | 'firstEntryBinary' | 'noData';

export type WebhookResponseMode =
    | 'onReceived'     // Respond immediately after webhook node executes
    | 'lastNode'       // Respond after last node finishes
    | 'responseNode'   // Respond from a "Respond to Webhook" node
    | 'formPage'       // Form-based response
    | 'hostedChat'     // Chat-based response
    | 'streaming';     // Streaming response
```

### 4.7 Webhook Node Pattern

```typescript
export class Webhook extends Node {
    description: INodeTypeDescription = {
        displayName: 'Webhook',
        name: 'webhook',
        group: ['trigger'],
        version: [1, 1.1, 2, 2.1],
        inputs: [],
        outputs: `={{(${configuredOutputs})($parameter)}}`,  // Dynamic outputs
        supportsCORS: true,
        triggerPanel: { /* ... */ },
        webhooks: [defaultWebhookDescription],  // Webhook configuration
        sensitiveOutputFields: ['headers.authorization', 'headers.cookie'],
        properties: [ /* ... */ ],
    };

    async webhook(context: IWebhookFunctions): Promise<IWebhookResponseData> {
        const req = context.getRequestObject();
        const resp = context.getResponseObject();
        const body = context.getBodyData();
        const headers = context.getHeaderData();
        const query = context.getQueryData();

        return {
            workflowData: [[{ json: body }]],
        };
    }
}
```

### 4.8 Webhook Description (Registration)

```typescript
export interface IWebhookDescription {
    httpMethod: IHttpRequestMethods | string;
    isFullPath?: boolean;
    name: WebhookType;                    // 'default' or 'setup'
    path: string;
    responseBinaryPropertyName?: string;
    responseContentType?: string;
    responsePropertyName?: string;
    responseMode?: WebhookResponseMode | string;
    responseData?: WebhookResponseData | string;
    restartWebhook?: boolean;
    nodeType?: 'webhook' | 'form' | 'mcp';
    ndvHideUrl?: string | boolean;
    ndvHideMethod?: string | boolean;
}

// Example
const defaultWebhookDescription: IWebhookDescription = {
    name: 'default',
    httpMethod: '={{$parameter["httpMethod"] || "GET"}}',
    isFullPath: true,
    responseMode: '={{$parameter["responseMode"]}}',
    path: '={{$parameter["path"]}}',
};
```

### 4.9 Webhook Lifecycle Methods (webhookMethods)

For nodes that register external webhooks (not n8n's own URL), use `webhookMethods`:

```typescript
webhookMethods: {
    default: {
        // Check if webhook already exists on the external service
        async checkExists(this: IHookFunctions): Promise<boolean> {
            const webhookUrl = this.getNodeWebhookUrl('default');
            // Check external service...
            return false;
        },
        // Create webhook on the external service
        async create(this: IHookFunctions): Promise<boolean> {
            const webhookUrl = this.getNodeWebhookUrl('default');
            // Register webhook on external service...
            return true;
        },
        // Delete webhook from the external service
        async delete(this: IHookFunctions): Promise<boolean> {
            // Remove webhook from external service...
            return true;
        },
    },
},
```

### 4.10 IPollFunctions Interface

```typescript
export interface IPollFunctions
    extends FunctionsBaseWithRequiredKeys<'getMode' | 'getActivationMode'> {
    __emit(data: INodeExecutionData[][], responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>, donePromise?: IDeferredPromise<IRun>): void;
    __emitError(error: Error, responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>): void;
    getNodeParameter(parameterName: string, fallbackValue?: any, options?: IGetNodeParameterOptions): NodeParameterValueType | object;
    helpers: RequestHelperFunctions & BaseHelperFunctions & BinaryHelperFunctions & SchedulingFunctions;
}
```

---

## 5. Workflow JSON Structure

### 5.1 IWorkflowBase (Complete Workflow Structure)

```typescript
export interface IWorkflowBase {
    id: string;
    name: string;
    description?: string | null;
    active: boolean;                    // Whether workflow is active (listening for triggers)
    isArchived: boolean;
    createdAt: Date;
    startedAt?: Date;
    updatedAt: Date;
    nodes: INode[];                     // Array of all nodes
    connections: IConnections;          // Connection map
    settings?: IWorkflowSettings;       // Workflow-level settings
    staticData?: IDataObject;           // Persistent data (e.g., polling state)
    pinData?: IPinData;                 // Pinned test data
    versionId?: string;
    activeVersionId: string | null;
    activeVersion?: IWorkflowHistory | null;
    versionCounter?: number;
    meta?: WorkflowFEMeta;             // Frontend metadata (template ID, etc.)
}
```

### 5.2 INode (Node in Workflow JSON)

```typescript
export interface INode {
    id: string;                         // Unique node ID (UUID)
    name: string;                       // Display name (must be unique in workflow)
    typeVersion: number;                // Node version
    type: string;                       // Node type (e.g., 'n8n-nodes-base.httpRequest')
    position: [number, number];         // [x, y] canvas position
    disabled?: boolean;                 // Whether node is disabled
    notes?: string;                     // User notes
    notesInFlow?: boolean;              // Show notes in flow
    retryOnFail?: boolean;              // Retry on error
    maxTries?: number;                  // Max retry attempts
    waitBetweenTries?: number;          // ms between retries
    alwaysOutputData?: boolean;         // Output empty item if no data
    executeOnce?: boolean;              // Execute only for first item
    onError?: OnError;                  // Error behavior
    continueOnFail?: boolean;           // Continue workflow on error
    parameters: INodeParameters;        // Node parameter values
    credentials?: INodeCredentials;     // Credential references
    webhookId?: string;                 // Webhook ID for webhook nodes
}

export type OnError = 'continueErrorOutput' | 'continueRegularOutput' | 'stopWorkflow';
```

### 5.3 IConnections (Connection Map)

```typescript
// Top level: keyed by SOURCE node name
export interface IConnections {
    [key: string]: INodeConnections;
}

// Second level: keyed by connection type (usually "main")
export interface INodeConnections {
    [key: string]: NodeInputConnections;
}

// Third level: array indexed by output index
// Each element is an array of connections from that output
export type NodeInputConnections = Array<IConnection[] | null>;

// Individual connection
export interface IConnection {
    node: string;                       // Destination node name
    type: NodeConnectionType;           // Connection type (usually "main")
    index: number;                      // Destination input index
}
```

### 5.4 Complete Workflow JSON Example

```json
{
    "id": "wf_abc123",
    "name": "My Workflow",
    "active": true,
    "isArchived": false,
    "nodes": [
        {
            "id": "node_1",
            "name": "Schedule Trigger",
            "type": "n8n-nodes-base.scheduleTrigger",
            "typeVersion": 1.2,
            "position": [250, 300],
            "parameters": {
                "rule": {
                    "interval": [{ "field": "hours", "hoursInterval": 1 }]
                }
            }
        },
        {
            "id": "node_2",
            "name": "HTTP Request",
            "type": "n8n-nodes-base.httpRequest",
            "typeVersion": 4.2,
            "position": [450, 300],
            "parameters": {
                "method": "GET",
                "url": "https://api.example.com/data",
                "authentication": "none"
            }
        },
        {
            "id": "node_3",
            "name": "IF",
            "type": "n8n-nodes-base.if",
            "typeVersion": 2,
            "position": [650, 300],
            "parameters": {
                "conditions": {
                    "options": { "caseSensitive": true },
                    "conditions": [
                        {
                            "leftValue": "={{ $json.status }}",
                            "rightValue": "active",
                            "operator": { "type": "string", "operation": "equals" }
                        }
                    ],
                    "combinator": "and"
                }
            }
        }
    ],
    "connections": {
        "Schedule Trigger": {
            "main": [
                [
                    { "node": "HTTP Request", "type": "main", "index": 0 }
                ]
            ]
        },
        "HTTP Request": {
            "main": [
                [
                    { "node": "IF", "type": "main", "index": 0 }
                ]
            ]
        },
        "IF": {
            "main": [
                [
                    { "node": "True Branch Node", "type": "main", "index": 0 }
                ],
                [
                    { "node": "False Branch Node", "type": "main", "index": 0 }
                ]
            ]
        }
    },
    "settings": {
        "executionOrder": "v1",
        "timezone": "Europe/Amsterdam",
        "saveDataErrorExecution": "all",
        "saveDataSuccessExecution": "all"
    },
    "pinData": {},
    "staticData": null,
    "meta": {}
}
```

### 5.5 NodeConnectionType Values

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

---

## 6. Custom Node Development

### 6.1 Two Building Approaches

| Approach | When to Use | execute() | routing |
|----------|------------|-----------|---------|
| **Programmatic** | Complex logic, data transformation, multiple APIs | YES - write custom code | No |
| **Declarative** | REST API wrappers, CRUD operations | NO - use routing config | YES - define in properties |

### 6.2 Community Node Project Structure

```
n8n-nodes-<your-name>/
├── package.json              # Node registration and metadata
├── tsconfig.json             # TypeScript configuration
├── credentials/
│   └── MyApi.credentials.ts  # Credential definitions
├── nodes/
│   └── MyNode/
│       ├── MyNode.node.ts    # Main node file
│       ├── MyNode.node.json  # Codex metadata (optional)
│       └── resources/        # Resource/operation descriptions (declarative)
│           └── user/
│               ├── index.ts
│               ├── create.ts
│               └── getAll.ts
├── icons/
│   └── my-icon.svg           # Node icon
└── dist/                     # Compiled output (generated)
```

### 6.3 package.json Configuration

```json
{
    "name": "n8n-nodes-myservice",
    "version": "0.1.0",
    "description": "n8n nodes for MyService",
    "license": "MIT",
    "keywords": ["n8n-community-node-package"],
    "n8n": {
        "n8nNodesApiVersion": 1,
        "strict": true,
        "credentials": [
            "dist/credentials/MyApi.credentials.js"
        ],
        "nodes": [
            "dist/nodes/MyNode/MyNode.node.js"
        ]
    },
    "scripts": {
        "build": "n8n-node build",
        "dev": "n8n-node dev",
        "lint": "n8n-node lint",
        "lint:fix": "n8n-node lint --fix",
        "release": "n8n-node release",
        "prepublishOnly": "n8n-node prerelease"
    },
    "files": ["dist"],
    "devDependencies": {
        "@n8n/node-cli": "*",
        "typescript": "5.9.2"
    },
    "peerDependencies": {
        "n8n-workflow": "*"
    }
}
```

**Critical**: The `n8n.nodes` and `n8n.credentials` arrays MUST point to the compiled `.js` files in `dist/`.

### 6.4 Programmatic Node Example (Complete)

```typescript
import type {
    IExecuteFunctions,
    INodeExecutionData,
    INodeType,
    INodeTypeDescription,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class MyNode implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'My Node',
        name: 'myNode',
        icon: 'file:myIcon.svg',
        group: ['transform'],
        version: 1,
        description: 'Does something useful',
        defaults: {
            name: 'My Node',
        },
        inputs: [NodeConnectionTypes.Main],
        outputs: [NodeConnectionTypes.Main],
        credentials: [
            {
                name: 'myApi',
                required: true,
            },
        ],
        properties: [
            {
                displayName: 'Resource',
                name: 'resource',
                type: 'options',
                noDataExpression: true,
                options: [
                    { name: 'User', value: 'user' },
                    { name: 'Post', value: 'post' },
                ],
                default: 'user',
            },
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: {
                    show: { resource: ['user'] },
                },
                options: [
                    { name: 'Get', value: 'get', action: 'Get a user' },
                    { name: 'Get Many', value: 'getAll', action: 'Get many users' },
                ],
                default: 'get',
            },
            {
                displayName: 'User ID',
                name: 'userId',
                type: 'string',
                required: true,
                default: '',
                displayOptions: {
                    show: { resource: ['user'], operation: ['get'] },
                },
                description: 'The ID of the user to retrieve',
            },
            {
                displayName: 'Limit',
                name: 'limit',
                type: 'number',
                default: 50,
                typeOptions: { minValue: 1, maxValue: 100 },
                displayOptions: {
                    show: { resource: ['user'], operation: ['getAll'] },
                },
                description: 'Max number of results to return',
            },
        ],
    };

    async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
        const items = this.getInputData();
        const returnData: INodeExecutionData[] = [];
        const resource = this.getNodeParameter('resource', 0) as string;
        const operation = this.getNodeParameter('operation', 0) as string;

        const credentials = await this.getCredentials('myApi');

        for (let i = 0; i < items.length; i++) {
            try {
                if (resource === 'user') {
                    if (operation === 'get') {
                        const userId = this.getNodeParameter('userId', i) as string;
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `https://api.example.com/users/${userId}`,
                            headers: {
                                Authorization: `Bearer ${credentials.apiKey}`,
                            },
                        });
                        returnData.push({ json: response as IDataObject });
                    }
                    if (operation === 'getAll') {
                        const limit = this.getNodeParameter('limit', i) as number;
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: 'https://api.example.com/users',
                            qs: { limit },
                            headers: {
                                Authorization: `Bearer ${credentials.apiKey}`,
                            },
                        });
                        const users = (response as IDataObject[]);
                        for (const user of users) {
                            returnData.push({ json: user });
                        }
                    }
                }
            } catch (error) {
                if (this.continueOnFail()) {
                    returnData.push({ json: { error: (error as Error).message }, pairedItem: { item: i } });
                    continue;
                }
                throw error;
            }
        }

        return [returnData];
    }
}
```

### 6.5 Declarative Node Example (Complete)

```typescript
import type { INodeType, INodeTypeDescription } from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class GithubIssues implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'GitHub Issues',
        name: 'githubIssues',
        icon: { light: 'file:github.svg', dark: 'file:github.dark.svg' },
        group: ['input'],
        version: 1,
        subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
        description: 'Consume issues from the GitHub API',
        defaults: { name: 'GitHub Issues' },
        usableAsTool: true,
        inputs: [NodeConnectionTypes.Main],
        outputs: [NodeConnectionTypes.Main],
        credentials: [
            {
                name: 'githubApi',
                required: true,
            },
        ],
        requestDefaults: {
            baseURL: 'https://api.github.com',
            headers: {
                Accept: 'application/json',
                'Content-Type': 'application/json',
            },
        },
        properties: [
            {
                displayName: 'Resource',
                name: 'resource',
                type: 'options',
                noDataExpression: true,
                options: [
                    { name: 'Issue', value: 'issue' },
                ],
                default: 'issue',
            },
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: { show: { resource: ['issue'] } },
                options: [
                    {
                        name: 'Get Many',
                        value: 'getAll',
                        action: 'Get issues in a repository',
                        routing: {
                            request: {
                                method: 'GET',
                                url: '=/repos/{{$parameter.owner}}/{{$parameter.repo}}/issues',
                            },
                        },
                    },
                    {
                        name: 'Create',
                        value: 'create',
                        action: 'Create an issue',
                        routing: {
                            request: {
                                method: 'POST',
                                url: '=/repos/{{$parameter.owner}}/{{$parameter.repo}}/issues',
                            },
                        },
                    },
                ],
                default: 'getAll',
            },
            // Parameters with routing.send for request body
            {
                displayName: 'Title',
                name: 'title',
                type: 'string',
                default: '',
                required: true,
                displayOptions: { show: { operation: ['create'] } },
                routing: {
                    send: {
                        type: 'body',
                        property: 'title',
                    },
                },
            },
        ],
    };

    // Declarative nodes: NO execute() method needed
    // n8n handles HTTP requests automatically based on routing config

    methods = {
        listSearch: {
            // Dynamic option loading methods
        },
    };
}
```

### 6.6 Credential Definition

```typescript
import type { ICredentialType, INodeProperties, IAuthenticate } from 'n8n-workflow';

export class MyApi implements ICredentialType {
    name = 'myApi';
    displayName = 'My API';
    documentationUrl = 'https://docs.example.com/api';
    properties: INodeProperties[] = [
        {
            displayName: 'API Key',
            name: 'apiKey',
            type: 'string',
            typeOptions: { password: true },
            default: '',
        },
        {
            displayName: 'Base URL',
            name: 'baseUrl',
            type: 'string',
            default: 'https://api.example.com',
        },
    ];

    authenticate: IAuthenticate = {
        type: 'generic',
        properties: {
            headers: {
                Authorization: '=Bearer {{$credentials.apiKey}}',
            },
        },
    };
}
```

### 6.7 INodeTypeDescription (Complete)

```typescript
export interface INodeTypeDescription extends INodeTypeBaseDescription {
    version: number | number[];              // Supported versions
    defaults: NodeDefaults;                  // Default values (name, color)
    eventTriggerDescription?: string;        // Description for trigger panel
    activationMessage?: string;              // Message when workflow activated
    inputs: Array<NodeConnectionType | INodeInputConfiguration> | ExpressionString;
    requiredInputs?: string | number[] | number;
    inputNames?: string[];
    outputs: Array<NodeConnectionType | INodeOutputConfiguration> | ExpressionString;
    outputNames?: string[];
    properties: INodeProperties[];           // Node parameters
    credentials?: INodeCredentialDescription[];
    maxNodes?: number;                       // Max instances in workflow
    polling?: true;                          // Marks as polling trigger
    supportsCORS?: true;                     // Webhook CORS support
    requestDefaults?: DeclarativeRestApiSettings.HttpRequestOptions;  // Declarative
    requestOperations?: IN8nRequestOperations;
    hooks?: {
        activate?: INodeHookDescription[];
        deactivate?: INodeHookDescription[];
    };
    webhooks?: IWebhookDescription[];        // Webhook registrations
    triggerPanel?: TriggerPanelDefinition | boolean;
    hints?: NodeHint[];                      // UI hints
    skipNameGeneration?: boolean;
    features?: NodeFeaturesDefinition;
    sensitiveOutputFields?: string[];        // Fields to always redact
}

export interface INodeTypeBaseDescription {
    displayName: string;                     // Human-readable name
    name: string;                            // Internal name (e.g., 'myNode')
    icon?: Icon;                             // Icon (FontAwesome or file)
    iconColor?: ThemeIconColor;              // Icon color
    iconUrl?: Themed<string>;                // URL-based icon
    badgeIconUrl?: Themed<string>;
    group: NodeGroupType[];                  // ['input'], ['output'], ['transform'], ['trigger']
    description: string;                     // Short description
    documentationUrl?: string;
    subtitle?: string;                       // Dynamic subtitle expression
    defaultVersion?: number;                 // Default version to use
    codex?: CodexData;                       // AI/search metadata
    parameterPane?: 'wide';
    hidden?: true;                           // Hide from node creator
    usableAsTool?: true | UsableAsToolDescription;  // AI agent tool support
    schemaPath?: string;
}

export interface INodeCredentialDescription {
    name: string;                            // Credential type name
    required?: boolean;
    displayName?: string;
    disabledOptions?: ICredentialsDisplayOptions;
    displayOptions?: ICredentialsDisplayOptions;  // Show/hide based on parameters
    testedBy?: ICredentialTestRequest | string;
}

export type NodeDefaults = Partial<{
    color: string;   // DEPRECATED - use iconColor instead
    name: string;    // Default node name
}>;

export type NodeGroupType = 'input' | 'output' | 'transform' | 'trigger' | 'schedule';
```

### 6.8 Development Workflow

1. **Scaffold**: Use `n8n-nodes-starter` template from GitHub
2. **Develop**: `npm run dev` — starts n8n with hot-reload at `http://localhost:5678`
3. **Build**: `npm run build` — compiles TypeScript to `dist/`
4. **Lint**: `npm run lint` — checks code quality
5. **Publish**: `npm publish` — publish to npm registry
6. **Verify**: Submit to n8n Creator Portal for cloud verification (optional)

### 6.9 Testing Custom Nodes

n8n provides a `WorkflowTestData` interface for testing:

```typescript
export interface WorkflowTestData {
    description: string;
    input: {
        workflowData: {
            nodes: INode[];
            connections: IConnections;
            settings?: IWorkflowSettings;
        };
    };
    output: {
        assertBinaryData?: boolean;
        nodeExecutionOrder?: string[];
        nodeData: { [key: string]: any[][] };
        error?: string;
    };
    nock?: {
        baseUrl: string;
        mocks: Array<{
            method: 'delete' | 'get' | 'patch' | 'post' | 'put';
            path: string;
            requestBody?: RequestBodyMatcher;
            statusCode: number;
            responseBody: string | object;
        }>;
    };
    trigger?: {
        mode: WorkflowExecuteMode;
        input: INodeExecutionData;
    };
    credentials?: Record<string, ICredentialDataDecryptedObject>;
}
```

### 6.10 Node Input/Output Configuration

```typescript
export interface INodeInputConfiguration {
    category?: string;
    displayName?: string;
    required?: boolean;
    type: NodeConnectionType;         // 'main', 'ai_languageModel', etc.
    filter?: INodeFilter;
    maxConnections?: number;
}

export interface INodeOutputConfiguration {
    category?: 'error';
    displayName?: string;
    maxConnections?: number;
    required?: boolean;
    type: NodeConnectionType;
    filter?: INodeFilter;
}
```

---

## 7. Key Patterns Summary

### 7.1 Regular Node Pattern
```
implements INodeType → description + execute()
```

### 7.2 Trigger Node Pattern
```
implements INodeType → description + trigger()
inputs: []  (no inputs)
group: ['trigger']
```

### 7.3 Webhook Node Pattern
```
extends Node → description + webhook()
inputs: []
group: ['trigger']
webhooks: [webhookDescription]
```

### 7.4 Declarative Node Pattern
```
implements INodeType → description with requestDefaults + routing in properties
NO execute() method
```

### 7.5 Polling Node Pattern
```
implements INodeType → description + poll()
polling: true in description
```

### 7.6 Error Handling Pattern
```typescript
try {
    // Node logic
} catch (error) {
    if (this.continueOnFail()) {
        returnData.push({ json: { error: (error as Error).message }, pairedItem: { item: i } });
        continue;
    }
    throw error;
}
```

---

## 8. Sources

- **Primary**: `n8n-io/n8n` GitHub repository, `packages/workflow/src/interfaces.ts` (master branch)
- **Execution context**: `packages/workflow/src/execution-context.ts`
- **Node examples**: `packages/nodes-base/nodes/` (HttpRequest, Set, Schedule, Webhook)
- **Community starter**: `n8n-io/n8n-nodes-starter` repository
- **Documentation**: `https://docs.n8n.io/` (navigation structure confirmed topics exist)
