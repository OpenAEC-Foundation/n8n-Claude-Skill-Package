# n8n v1.x Vooronderzoek — Deep Research

> Date: 2026-03-19
> Sources: Official n8n documentation, n8n-io/n8n GitHub repository
> Scope: Complete n8n v1.x platform coverage for skill package development

---

## Table of Contents

1. [Execution Model](#1-execution-model)
2. [INodeType Interface](#2-inodetype-interface)
3. [Node Properties (INodeProperties)](#3-node-properties-inodeproperties)
4. [Trigger Nodes](#4-trigger-nodes)
5. [Workflow JSON Structure](#5-workflow-json-structure)
6. [Custom Node Development](#6-custom-node-development)
7. [Expression System](#7-expression-system)
8. [Expression Extension Methods](#8-expression-extension-methods)
9. [JMESPath](#9-jmespath)
10. [Code Node](#10-code-node)
11. [Credential System](#11-credential-system)
12. [Data Handling & Binary Data](#12-data-handling--binary-data)
13. [AI/LLM Nodes](#13-aillm-nodes)
14. [REST API](#14-rest-api)
15. [Docker Deployment](#15-docker-deployment)
16. [Queue Mode & Scaling](#16-queue-mode--scaling)
17. [Webhooks](#17-webhooks)
18. [Error Handling](#18-error-handling)
19. [Environment Variables & Security](#19-environment-variables--security)
20. [CLI & Backup](#20-cli--backup)

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

### 6.11 Key Patterns Summary

#### Regular Node Pattern
```
implements INodeType -> description + execute()
```

#### Trigger Node Pattern
```
implements INodeType -> description + trigger()
inputs: []  (no inputs)
group: ['trigger']
```

#### Webhook Node Pattern
```
extends Node -> description + webhook()
inputs: []
group: ['trigger']
webhooks: [webhookDescription]
```

#### Declarative Node Pattern
```
implements INodeType -> description with requestDefaults + routing in properties
NO execute() method
```

#### Polling Node Pattern
```
implements INodeType -> description + poll()
polling: true in description
```

#### Error Handling Pattern
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

## 7. Expression System

### 7.1 Syntax

n8n expressions use double-curly-brace syntax: `{{ expression }}`. They are embedded directly in node parameter fields to dynamically compute values.

- Expressions are "small pieces of JavaScript-like code you put directly into node parameters"
- They support standard JavaScript operations plus n8n-specific built-in variables and methods
- Expressions provide immediate preview of computed values in the expression editor
- For multi-statement logic, use an IIFE (Immediately Invoked Function Expression):
  ```js
  {{ (function() { const x = $json.value * 2; return x; })() }}
  ```

### 7.2 Built-in Variables

#### Current Node Input

| Variable | Returns | Description |
|----------|---------|-------------|
| `$json` | Object | Shorthand for `$input.item.json` — the current item's JSON data |
| `$binary` | Object | Shorthand for `$input.item.binary` — the current item's binary data |
| `$input.item` | Item | The input item currently being processed |
| `$input.all(branchIndex?, runIndex?)` | Array\<Item\> | All input items from the current node |
| `$input.first(branchIndex?, runIndex?)` | Item | First input item |
| `$input.last(branchIndex?, runIndex?)` | Item | Last input item |
| `$input.params` | NodeParams | Node configuration settings and operation parameters |

#### Other Node Output

| Variable | Returns | Description |
|----------|---------|-------------|
| `$("<node-name>").all(branchIndex?, runIndex?)` | Array\<Item\> | All items from the named node's output |
| `$("<node-name>").first(branchIndex?, runIndex?)` | Item | First item from named node |
| `$("<node-name>").last(branchIndex?, runIndex?)` | Item | Last item from named node |
| `$("<node-name>").item` | Item | The linked item (the item in the specified node used to produce the current item) |
| `$("<node-name>").itemMatching(currentNodeInputIndex)` | Item | Traces back from input item to matching item in named node (preferred in Code node) |
| `$("<node-name>").params` | Object | Query settings/parameters of the given node |
| `$("<node-name>").context` | Object | Only available when working with Loop Over Items node |
| `$("<node-name>").isExecuted` | Boolean | True if the node has executed, false otherwise |

#### Workflow & Execution Metadata

| Variable | Returns | Description |
|----------|---------|-------------|
| `$workflow.id` | String | The workflow ID (also in workflow URL) |
| `$workflow.name` | String | The workflow name as shown in editor |
| `$workflow.active` | Boolean | Whether the workflow is active |
| `$execution.id` | String | Unique ID of the current workflow execution |
| `$execution.mode` | String | `"test"` (manual), `"production"` (auto trigger), or `"evaluation"` (workflow tests) |
| `$execution.resumeUrl` | String | Webhook URL to resume a workflow waiting at a Wait node |
| `$execution.resumeFormUrl` | String | URL to access a form generated by the Wait node |
| `$execution.customData` | CustomData | Set and get custom execution data (see below) |

#### Custom Execution Data Methods

```js
// Set individual key-value pair
$execution.customData.set("user_email", "me@example.com");

// Set multiple pairs at once
$execution.customData.setAll({"key1": "value1", "key2": "value2"});

// Get a specific value
const email = $execution.customData.get("user_email");

// Get all custom data
const allData = $execution.customData.getAll();
// Returns: {"user_email": "me@example.com", "key1": "value1", ...}
```

#### Environment & Configuration

| Variable | Returns | Description |
|----------|---------|-------------|
| `$env` | Object | n8n instance configuration environment variables |
| `$vars` | Object | Variables available in the active environment. Access: `$vars.<variable-name>`. All variables are strings. |
| `$secrets` | Object | External secrets (NOT available in Code node) |

#### Date & Time

| Variable | Returns | Description |
|----------|---------|-------------|
| `$now` | DateTime (Luxon) | Current moment, respects workflow timezone. Equivalent to `DateTime.now()` |
| `$today` | DateTime (Luxon) | Midnight at start of current day, respects instance timezone |

#### Node Execution Context

| Variable | Returns | Description |
|----------|---------|-------------|
| `$prevNode.name` | String | Name of the node that the current input came from |
| `$prevNode.outputIndex` | Number | Index of the output connector the current input came from |
| `$prevNode.runIndex` | Number | Run of the previous node that generated the current input |
| `$runIndex` | Number | How many times n8n has executed the current node (zero-based) |
| `$itemIndex` | Number | Position of the item currently being processed in input list (NOT available in Code node) |
| `$parameter` | Object | Configuration settings of the current node |
| `$nodeVersion` | Number | Current node version |
| `$pageCount` | Number | Number of results pages fetched (HTTP Request node only) |

#### Utility Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `$ifEmpty(value, fallback)` | any | Returns `value` if non-empty, otherwise `fallback`. Empty = `""`, `[]`, `{}`, `null`, `undefined` |
| `$jmespath(object, searchString)` | any | Perform JMESPath query on a JSON object |
| `$getWorkflowStaticData(type)` | Object | Get persistent static data. Type: `"global"` or `"node"` |

#### HTTP Response (HTTP Request Node Only)

| Variable | Returns | Description |
|----------|---------|-------------|
| `$response.body` | Object | Body of the response from the last HTTP call |
| `$response.headers` | Object | Headers returned by the last HTTP call |
| `$response.statusCode` | Number | HTTP status code |
| `$response.statusMessage` | String | Optional message regarding request status |

### 7.3 Paired Items (Item Linking)

"Every item in a node's input data links back to the items used in previous nodes to generate it."

- n8n automatically tracks which input items produced which output items
- Access linked items via `$("<node-name>").item` in expressions
- In Code node, use `$("<node-name>").itemMatching(currentNodeInputIndex)` instead
- Item linking enables drag-and-drop mapping to automatically resolve the correct item
- When a node processes multiple items, expressions like `{{ $json.fruit }}` dynamically resolve to each item's value

### 7.4 Static Workflow Data

Persistent data that survives across executions:

```js
// Global (accessible by every node)
const staticData = $getWorkflowStaticData('global');
const lastRun = staticData.lastExecution;
staticData.lastExecution = new Date().toISOString();
// n8n saves automatically on successful execution

// Node-specific (only accessible by the current node)
const nodeData = $getWorkflowStaticData('node');
nodeData.processedIds = nodeData.processedIds || [];
```

**Limitations:**
- Not available during workflow testing (must be active workflow with trigger/webhook)
- May be unreliable during high-frequency executions
- Data should be small

---

## 8. Expression Extension Methods

### 8.1 String Methods (n8n Extensions)

| Method | Returns | Description |
|--------|---------|-------------|
| `.extractEmail()` | String\|undefined | Extracts first email found in the string |
| `.extractUrl()` | String\|undefined | Extracts first URL (must start with `http`) |
| `.extractDomain()` | String\|undefined | From email/URL, returns domain |
| `.extractUrlPath()` | String\|undefined | Returns URL path after domain |
| `.isEmail()` | Boolean | True if string is an email |
| `.isUrl()` | Boolean | True if string is a valid URL |
| `.isEmpty()` | Boolean | True if no characters or null |
| `.isNotEmpty()` | Boolean | True if at least one character |
| `.removeMarkdown()` | String | Removes Markdown formatting and HTML tags |
| `.toDateTime()` | DateTime | Converts to DateTime. Supports ISO 8601, HTTP, RFC2822, SQL, Unix ms |
| `.toBoolean()` | Boolean | `"0"`, `"false"`, `"no"` -> false; everything else -> true (case-insensitive) |
| `.toSentenceCase()` | String | First letter of each sentence capitalized |
| `.toSnakeCase()` | String | Spaces/dashes -> `_`, symbols removed, lowercased |
| `.toTitleCase()` | String | First letter of each word capitalized |
| `.urlEncode(allChars?)` | String | URL-encodes string. Optional: encode URI syntax chars too |
| `.urlDecode(allChars?)` | String | Decodes URL-encoded string |
| `.base64Encode()` | String | Converts plain text to base64 |
| `.base64Decode()` | String | Converts base64 to plain text |
| `.hash(algo?)` | String | Hash with algorithm. Default: md5. Supports: md5, base64, sha1, sha224, sha256, sha384, sha512, sha3, ripemd160 |
| `.quote(mark?)` | String | Wraps in quotation marks, escapes existing quotes |
| `.replaceSpecialChars()` | String | Replaces special characters with closest ASCII equivalent |

### 8.2 Array Methods (n8n Extensions)

| Method | Returns | Description |
|--------|---------|-------------|
| `.append(elem1, ..., elemN)` | Array | Adds elements to end of array |
| `.average()` | Number | Mean of numeric elements (errors on non-numbers) |
| `.chunk(length)` | Array\<Array\> | Partitions array into sub-arrays of given length |
| `.compact()` | Array | Removes null, "", and undefined values |
| `.difference(otherArray)` | Array | Elements in base array not in otherArray |
| `.intersection(otherArray)` | Array | Elements present in both arrays |
| `.isEmpty()` | Boolean | True if no elements or null |
| `.isNotEmpty()` | Boolean | True if at least one element |
| `.first()` | any | Returns first element |
| `.last()` | any | Returns last element |
| `.max()` | Number | Largest number in array |
| `.min()` | Number | Smallest number in array |
| `.pluck(field1?, field2?, ...)` | Array | Extracts values from specified object fields |
| `.randomItem()` | any | Random element from the array |
| `.removeDuplicates(keys?)` | Array | Removes duplicates. Optional keys param for object arrays |
| `.renameKeys(from, to)` | Array | Renames object field names |
| `.smartJoin(keyField, nameField)` | Object | Creates single Object from array of Objects |
| `.sum()` | Number | Total of all numbers in array |
| `.toJsonString()` | String | Converts to JSON string (equivalent to JSON.stringify) |
| `.union(otherArray)` | Array | Concatenates and removes duplicates |
| `.unique()` | Array | Removes duplicate elements |

### 8.3 Number Methods (n8n Extensions)

| Method | Returns | Description |
|--------|---------|-------------|
| `.abs()` | Number | Absolute value |
| `.ceil()` | Number | Rounds up to next whole number |
| `.floor()` | Number | Rounds down to nearest whole number |
| `.format(locale?, options?)` | String | Formatted string using Intl.NumberFormat |
| `.isEmpty()` | Boolean | False for all numbers, true for null |
| `.isEven()` | Boolean | True if even (errors for non-integers) |
| `.isInteger()` | Boolean | True if whole number |
| `.isOdd()` | Boolean | True if odd (errors for non-integers) |
| `.round(decimalPlaces?)` | Number | Rounds to nearest whole or specified decimals |
| `.toBoolean()` | Boolean | 0 -> false; everything else -> true |
| `.toDateTime(format?)` | DateTime | Converts timestamp (ms/s/excel format) to DateTime |

### 8.4 Object Methods (n8n Extensions)

| Method | Returns | Description |
|--------|---------|-------------|
| `.compact()` | Object | Removes fields with null or "" values |
| `.hasField(name)` | Boolean | True if top-level key exists |
| `.isEmpty()` | Boolean | True if no keys or null |
| `.isNotEmpty()` | Boolean | True if at least one key |
| `.keepFieldsContaining(value)` | Object | Keeps only fields with matching values |
| `.keys()` | Array | All field names |
| `.merge(otherObject)` | Object | Combines objects; base values take precedence |
| `.removeField(key)` | Object | Removes a field |
| `.removeFieldsContaining(value)` | Object | Removes fields with partially matching values |
| `.toJsonString()` | String | Converts to JSON string |
| `.urlEncode()` | String | Generates URL parameter string from keys and values |
| `.values()` | Array | All field values |

### 8.5 Boolean Methods (n8n Extensions)

| Method | Returns | Description |
|--------|---------|-------------|
| `.isEmpty()` | Boolean | False for all booleans, true for null |
| `.toNumber()` | Number | true -> 1, false -> 0 |
| `.toString()` | String | true -> "true", false -> "false" |

### 8.6 DateTime Methods (Luxon-based)

#### Property Accessors

| Property | Returns | Description |
|----------|---------|-------------|
| `.day` | Number | Day of month (1-31) |
| `.hour` | Number | Hour (0-23) |
| `.minute` | Number | Minute (0-59) |
| `.second` | Number | Second (0-59) |
| `.millisecond` | Number | Millisecond (0-999) |
| `.month` | Number | Month (1-12) |
| `.monthLong` | String | Full month name (e.g., "March") |
| `.monthShort` | String | Abbreviated month (e.g., "Mar") |
| `.weekday` | Number | Day of week (1=Monday, 7=Sunday) |
| `.weekdayLong` | String | Full weekday name |
| `.weekdayShort` | String | Abbreviated weekday |
| `.weekNumber` | Number | Week number (1-52) |
| `.quarter` | Number | Quarter (1-4) |
| `.year` | Number | Year |
| `.locale` | String | Locale string (e.g., "en-US") |
| `.zone` | Object | Time zone object with name and validity |
| `.isInDST` | Boolean | Daylight saving status |

#### Transformation Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `.plus(n, unit?)` | DateTime | Adds period. Units: years, months, weeks, days, hours, minutes, seconds, milliseconds |
| `.minus(n, unit?)` | DateTime | Subtracts period |
| `.set(values)` | DateTime | Modifies components: `{year, month, day, hour, minute, second, millisecond}` |
| `.startOf(unit)` | DateTime | Rounds down to start of unit |
| `.endOf(unit)` | DateTime | Rounds up to end of unit |
| `.setZone(zone)` | DateTime | Converts to time zone (e.g., "America/New_York", "UTC+3") |
| `.setLocale(locale)` | DateTime | Sets locale for formatting |
| `.toLocal()` | DateTime | Converts to workflow's local time zone |
| `.toUTC(offset?)` | DateTime | Converts to UTC |

#### Comparison & Analysis (n8n custom)

| Method | Returns | Description |
|--------|---------|-------------|
| `.equals(other)` | Boolean | True if identical moment and timezone |
| `.hasSame(other, unit)` | Boolean | True if same down to specified unit (ignores timezone) |
| `.isBetween(date1, date2)` | Boolean | True if between two moments |
| `.diffTo(other, unit)` | Duration | Difference in given unit(s) |
| `.diffToNow(unit)` | Duration | Difference between now and DateTime |
| `.extract(unit?)` | Number | Extracts part of date (year, month, week, day, hour, minute, second) |

#### String Conversion

| Method | Returns | Description |
|--------|---------|-------------|
| `.format(fmt)` | String | n8n custom — formats using Luxon tokens |
| `.toLocaleString(options?)` | String | Locale-specific string |
| `.toISO()` | String | ISO 8601 string |
| `.toString()` | String | ISO-formatted string |
| `.toRelative(options?)` | String | Relative time string (e.g., "2 days ago") |
| `.toMillis()` | Number | Unix timestamp in milliseconds |
| `.toSeconds()` | Number | Unix timestamp in seconds |

#### Luxon Usage Examples

```js
// Date arithmetic
{{ $today.minus({days: 7}).toLocaleString() }}

// Format a date
{{ $now.format("yyyy-MM-dd HH:mm:ss") }}

// Parse from ISO
{{ DateTime.fromISO("2024-06-23T00:00:00.00") }}

// Parse custom format
{{ DateTime.fromFormat("23-06-2024", "dd-MM-yyyy") }}

// Duration between dates
{{ DateTime.fromISO("2024-06-23").diff(DateTime.fromISO("2024-05-23"), "months").toObject() }}
// Returns: {"months": 1}

// Human-readable relative time
{{ $now.minus({hours: 3}).toRelative() }}
// Returns: "3 hours ago"
```

### 8.7 BinaryFile Properties

| Property | Returns | Description |
|----------|---------|-------------|
| `.directory` | String | Folder path (not set if stored in database) |
| `.fileExtension` | String | File extension (e.g., "txt", "pdf") |
| `.fileName` | String | Complete filename with extension |
| `.fileSize` | String | File size in bytes |
| `.fileType` | String | First part of MIME type (e.g., "image", "video") |
| `.id` | String | Unique identifier for disk/S3 storage |
| `.mimeType` | String | Full MIME type (e.g., "image/jpeg", "text/plain") |

---

## 9. JMESPath

### 9.1 Syntax

```js
// JavaScript (expressions and Code node)
$jmespath(object, searchString)

// Python (Code node only)
_jmespath(object, searchString)
```

**Important:** The parameter order is `(object, searchString)`, which differs from the JMESPath specification's `search(searchString, object)` pattern. The n8n implementation follows the JMESPath JavaScript library convention.

### 9.2 Supported Features

#### List Projections
```js
// Extract specific field from all array elements
$jmespath($json.body.people, "[*].first")
// Returns: ["James", "Jacob", "Jayden"]
```

#### Slice Projections
```js
// Get subset of array elements
$jmespath($json.body.people, "[:2].first")
// Returns: ["James", "Jacob"]
```

#### Object Projections
```js
// Extract values across object properties
$jmespath($json.body.dogs, "*.age")
// Returns: [7, 5]
```

#### Multiselect (Lists and Objects)
```js
// Select multiple fields into new list
$jmespath($json.body.people, "[].[first, last]")
// Returns: [["James","Green"],["Jacob","Jones"],["Jayden","Smith"]]
```

#### Filter Expressions
```js
// Filter and extract specific items
$jmespath($("Code").all(), "[?json.name=='Lenovo'].json.category_id")
```

#### Flatten Projections and Pipe Expressions
JMESPath also supports flatten projections (`[]`) and pipe expressions (`|`) for chaining operations.

### 9.3 Python Usage

```python
firstNames = _jmespath(_json.body.people, "[*].first")
return {"firstNames": firstNames}
```

**Note:** Python JMESPath is only available in Code nodes, not in expressions.

---

## 10. Code Node

### 10.1 Overview

The Code node allows custom JavaScript or Python code in n8n workflows. It runs in a sandboxed environment.

**Supported Languages:**
- **JavaScript (Node.js)** — default, always available
- **Python (Native)** — requires `N8N_PYTHON_ENABLED=true` environment variable (stable since v1.111.0+)

### 10.2 Execution Modes

| Mode | Description | Default Variable |
|------|-------------|-----------------|
| **Run Once for All Items** | Executes once regardless of input item count (default) | `items` (JS) / `_items` (Python) |
| **Run Once for Each Item** | Executes once per input item | `$input.item` (JS) / `_item` (Python) |

### 10.3 JavaScript Variables in Code Node

All `$` prefixed variables from the expression system are available, EXCEPT:
- `$itemIndex` — NOT available in Code node
- `$secrets` — NOT available in Code node

Additional Code node specific:
- `items` — Array of all input items (in "Run Once for All Items" mode)
- `$input.item` — Current item (in "Run Once for Each Item" mode)

### 10.4 Python Variables

Python uses `_` prefix instead of `$`:
- `_items` — all items (all-items mode)
- `_item` — current item (each-item mode)
- `_execution`, `_workflow`, `_env`, `_vars` — metadata
- `_jmespath()` — JMESPath queries
- `_getWorkflowStaticData()` — static data access

**Python limitations:**
- Cannot use dot notation for item access: use `item["json"]["field"]` (bracket notation)
- On n8n Cloud: no external library imports
- Self-hosted: can import standard library and allowlisted third-party modules
- `getBinaryDataBuffer()` is NOT supported in Python

### 10.5 Built-in Modules

**n8n Cloud:**
- Node.js `crypto` module
- `moment` package
- Luxon (via expressions)

**Self-hosted:**
- All of the above
- External npm modules if enabled via environment configuration

### 10.6 Binary Data in Code Node

```js
// Get binary data buffer (JavaScript only)
const buffer = await this.helpers.getBinaryDataBuffer(itemIndex, binaryPropertyName);

// Example: get buffer for first item's 'data' property
const binaryDataBuffer = await this.helpers.getBinaryDataBuffer(0, 'data');
```

**Parameters:**
- `itemIndex` (number): Position of the item in input data
- `binaryPropertyName` (string): Name of binary property (default: `"data"` for Read/Write File From Disk)

**Best practice:** ALWAYS use `getBinaryDataBuffer()` instead of deprecated direct access like `items[0].binary.data.data`.

### 10.7 Common Patterns

#### Count items from previous node
```js
// JavaScript
if (Object.keys(items[0].json).length === 0) {
  return [{ json: { results: 0 } }];
}
return [{ json: { results: items.length } }];
```

#### Console logging
```js
let a = "apple";
console.log(a);
// Output visible in browser developer console
```

#### Execution data access
```js
let executionId = $execution.id;
$execution.customData.set("key", "value");
$execution.customData.setAll({"key1": "value1", "key2": "value2"});
let customData = $execution.customData.getAll();
```

### 10.8 Return Format

Both JavaScript and Python must return data in n8n's item format:

```js
// JavaScript — Run Once for All Items
return items.map(item => ({
  json: { processedField: item.json.originalField }
}));

// JavaScript — Run Once for Each Item
return { json: { processedField: $input.item.json.originalField } };
```

### 10.9 Restrictions

- Cannot access the file system directly — use Read/Write Files nodes
- Cannot make HTTP requests — use the HTTP Request node
- Runs slower than native nodes (sandboxed execution)
- Recommended to use native nodes when possible for: data manipulation, filtering, conditional routing, array splitting, aggregation, deduplication, limiting items, merging, date/time operations, HTML creation

---

## 11. Credential System

### 11.1 File Structure

Credentials are defined in `<node-name>.credentials.ts` files that implement the `ICredentialType` interface from `n8n-workflow`.

```typescript
import { ICredentialType, INodeProperties } from 'n8n-workflow';

export class ExampleApi implements ICredentialType {
  name = 'exampleApi';
  displayName = 'Example API';
  documentationUrl = 'https://docs.example.com';
  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      default: '',
    },
  ];
}
```

### 11.2 ICredentialType Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Internal identifier (e.g., `'exampleNodeApi'`) |
| `displayName` | string | GUI label shown to users |
| `documentationUrl` | string | Link to credentials documentation |
| `properties` | INodeProperties[] | Array of credential input fields |
| `authenticate` | object | How credentials are injected into requests |
| `test` | ICredentialTestRequest | Endpoint for credential validation |

### 11.3 Credential Property Types

Each property in the `properties` array supports these fields:
- `displayName` — User-facing label
- `name` — Internal reference name
- `type` — Data type: `'string'`, `'number'`, `'boolean'`, `'options'`
- `default` — Default value
- `typeOptions` — Additional options, e.g., `{ password: true }` for password fields

### 11.4 Authentication Methods

The `authenticate` property defines how credentials are injected into HTTP requests. All methods require `type: 'generic'`:

#### Query String
```typescript
authenticate = {
  type: 'generic',
  properties: {
    qs: {
      'api_key': '={{$credentials.apiKey}}'
    }
  }
};
```

#### Header
```typescript
authenticate = {
  type: 'generic',
  properties: {
    header: {
      Authorization: '=Bearer {{$credentials.authToken}}'
    }
  }
};
```

#### Request Body
```typescript
authenticate = {
  type: 'generic',
  properties: {
    body: {
      username: '={{$credentials.username}}',
      password: '={{$credentials.password}}'
    }
  }
};
```

#### Basic Auth
```typescript
authenticate = {
  type: 'generic',
  properties: {
    auth: {
      username: '={{$credentials.username}}',
      password: '={{$credentials.password}}'
    }
  }
};
```

### 11.5 Credential Testing

The `test` property enables credential validation during setup:

```typescript
test: ICredentialTestRequest = {
  request: {
    baseURL: '={{$credentials?.domain}}',
    url: '/bearer',
  }
};
```

n8n calls this endpoint when users test credentials in the GUI. A successful response validates the credentials.

### 11.6 Encryption

n8n encrypts all credential data using an encryption key. Credentials are never stored in plain text.

---

## 12. Data Handling & Binary Data

### 12.1 Data Structure

n8n organizes ALL inter-node data as **an array of objects**. Each item has this structure:

```json
[
  {
    "json": {
      "key": "value",
      "nested": { "field": "data" }
    },
    "binary": {
      "data": {
        "mimeType": "image/jpeg",
        "fileExtension": "jpg",
        "fileName": "photo.jpg",
        "fileSize": "12345",
        "fileType": "image",
        "data": "<base64-encoded-content>"
      }
    }
  }
]
```

- **json** — Standard JSON data wrapped under a `"json"` key
- **binary** — Optional file/binary data under a `"binary"` key with base64 encoding and metadata

### 12.2 Data Flow Between Nodes

- Nodes process multiple items automatically — if a node receives 10 items, it performs its operation on each one individually
- Expressions like `{{ $json.fruit }}` dynamically resolve per item during iteration
- The Function and Code nodes auto-wrap results in the required structure; custom nodes must return properly formatted data

### 12.3 Data Mapping

Data mapping = "accessing information from previous nodes in your workflow."

Two methods:
1. **Drag-and-drop** — From the Input panel in the UI (creates expressions automatically)
2. **Manual expressions** — Type expressions directly in parameter fields (e.g., `{{ $json.body.city }}`)

Access patterns:
- Dot notation: `{{ $json.body.city }}`
- Bracket notation: `{{ $json['body']['city'] }}`

### 12.4 Item Linking / Paired Items

Every item links back to the items that produced it in previous nodes. This enables:
- Automatic item resolution in expressions
- Restoring fields removed by intermediate nodes
- `$("<node-name>").itemMatching(index)` to trace item origins

### 12.5 Binary Data Nodes

| Node | Purpose |
|------|---------|
| Convert to File | Transforms input data into file format output |
| Extract From File | Retrieves data from binary formats -> JSON |
| Read/Write Files from Disk | File I/O on n8n host machine |
| Compression | File compression tasks |
| Edit Image | Image manipulation |
| FTP | Remote file operations |
| Local File Trigger | Workflow trigger based on file system changes |

### 12.6 Data Pinning

n8n supports data pinning — saving test data so you don't need to re-execute nodes during development.

### 12.7 Data Transformation Approaches

- **Expressions** — Inline transforms in parameter fields
- **Edit Fields (Set) node** — Add, modify, remove, or rename fields
- **Code node** — Complex multi-step transformations
- **AI Transform node** — GPT-powered code generation for transformations (not available self-hosted, no Python)

### 12.8 Custom Node Development: Data Handling

#### Node File Structure

```
nodes/
  MyNode/
    MyNode.node.ts          # Main node implementation
    MyNode.node.json        # Codex file (metadata)
credentials/
  MyNode.credentials.ts    # Credentials definition
package.json               # npm module configuration
```

For modular nodes:
```
nodes/
  MyNode/
    MyNode.node.ts
    actions/
      resource1/
        index.ts
        create.operation.ts
        get.operation.ts
    methods/              # Dynamic parameter functions
    transport/            # Communication implementation
```

#### INodeProperties (UI Elements)

Available `type` values for node properties:

| Type | Description |
|------|-------------|
| `'string'` | Text input (supports password, textarea, drag-and-drop) |
| `'number'` | Numeric input (supports min/max, precision) |
| `'boolean'` | Toggle switch |
| `'options'` | Single-select dropdown |
| `'multiOptions'` | Multi-select list |
| `'collection'` | Group of optional related fields |
| `'fixedCollection'` | Semantically related field groups (supports repeating) |
| `'color'` | Color picker |
| `'dateTime'` | Date picker |
| `'json'` | Raw JSON editor |
| `'html'` | HTML editor with expression support |
| `'filter'` | Data matching/evaluation |
| `'assignmentCollection'` | Drag-and-drop name-value pairs |
| `'resourceLocator'` | Resource finder (ID, URL, or list modes) |
| `'resourceMapper'` | Insert/update/upsert data mapping |
| `'notice'` | Yellow informational box |

#### displayOptions for Conditional Visibility

```typescript
displayOptions: {
  show: {
    resource: ['contact'],
    operation: ['create']
  }
}
```

#### HTTP Helper Methods

```typescript
// Unauthenticated request
const response = await this.helpers.httpRequest(options);

// Authenticated request
const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'credentialTypeName', options
);
```

**Options object:**

| Property | Type | Description |
|----------|------|-------------|
| `url` | string | Required. Request URL |
| `method` | string | GET, POST, PUT, DELETE, HEAD (default: GET) |
| `headers` | object | Key-value header pairs |
| `body` | any | FormData, Array, string, number, object, Buffer, URLSearchParams |
| `qs` | object | Query string parameters |
| `auth` | object | Basic auth: `{username, password}` |
| `encoding` | string | arraybuffer, blob, document, json, text, stream |
| `returnFullResponse` | boolean | Returns `{body, headers, statusCode, statusMessage}` |
| `skipSslCertificateValidation` | boolean | Skip SSL verification |
| `timeout` | number | Request timeout in ms |
| `proxy` | object | `{host, port, auth, protocol}` |
| `json` | boolean | Parse response as JSON |
| `disableFollowRedirect` | boolean | Don't follow redirects |
| `arrayFormat` | string | indices, brackets, repeat, comma |

#### Error Handling in Custom Nodes

Two error classes:

##### NodeApiError (external service failures)
```typescript
// HTTP request failures, API errors, auth failures, rate limiting
throw new NodeApiError(this.getNode(), error as JsonObject, {
  message: 'Resource not found',
  description: `The resource with ID ${id} was not found`,
});
```

##### NodeOperationError (internal/validation errors)
```typescript
// Validation, configuration, input errors, data transformation errors
throw new NodeOperationError(this.getNode(), 'Invalid email format', {
  itemIndex: i,  // Include for batch processing
});
```

##### continueOnFail Pattern
```typescript
if (this.continueOnFail()) {
  returnData.push({ json: { error: error.message } });
  continue;
}
throw error;
```

#### Code Standards

- All n8n code is TypeScript
- NEVER modify incoming data from `this.getInputData()` — always clone first
- Use built-in HTTP helpers instead of third-party libraries (axios internally)
- Reuse internal field names across operations to preserve user data when switching
- Run the n8n node linter before publishing

---

## 13. AI/LLM Nodes

### 13.1 Overview

n8n integrates with LangChain to provide advanced AI capabilities. Requires n8n v1.19.4+ (Cloud or self-hosted).

AI nodes use a **cluster node architecture** — groups of root nodes and sub-nodes that work together.

### 13.2 Root Nodes (Standalone Entry Points)

#### AI Agents
| Node | Description |
|------|-------------|
| Conversational Agent | General chat-based agent |
| OpenAI Functions Agent | Uses OpenAI function calling |
| Plan and Execute Agent | Plans steps then executes |
| ReAct Agent | Reasoning and acting pattern |
| SQL Agent | Database query agent |
| Tools Agent | Generic tool-using agent |

#### Chains
| Node | Description |
|------|-------------|
| Basic LLM Chain | Simple prompt -> response |
| Question and Answer Chain | RAG-based Q&A |
| Summarization Chain | Text summarization |

#### Specialized
| Node | Description |
|------|-------------|
| Information Extractor | Structured data extraction |
| Text Classifier | Text categorization |
| Sentiment Analysis | Sentiment detection |
| LangChain Code | Custom LangChain code |

### 13.3 Sub-Nodes (Supporting Components)

#### Chat Models
Anthropic, AWS Bedrock, Azure OpenAI, Cohere, DeepSeek, Google Gemini, Google Vertex, Groq, Mistral, Ollama (local), OpenAI, OpenRouter

#### Embedding Models
OpenAI, Cohere, Google, HuggingFace, Mistral, Ollama, Azure OpenAI

#### Memory Types
| Memory | Description |
|--------|-------------|
| Simple Memory | Buffer window memory (default 5 exchanges) |
| Chat Memory Manager | Advanced memory management |
| MongoDB Chat Memory | Persistent MongoDB storage |
| Redis Chat Memory | Redis-backed memory |
| Postgres Chat Memory | PostgreSQL-backed memory |
| Motorhead | Motorhead memory server |
| Zep | Zep memory service |
| Xata | Xata database memory |

#### Vector Stores
Azure AI Search, Simple Vector Store, Milvus, MongoDB Atlas, PGVector, Chroma, Pinecone, Qdrant, Redis, Supabase, Weaviate

#### Document Loaders
Various loaders for ingesting documents into vector stores.

#### Text Splitters
| Splitter | Description |
|----------|-------------|
| Character Text Splitter | Splits by character count |
| Recursive Character Text Splitter | Recommended — recursive splitting |
| Token Text Splitter | Splits by token count |

#### Output Parsers
| Parser | Description |
|--------|-------------|
| Auto-fixing Parser | Automatically corrects parsing errors |
| Item List Parser | Parses comma-separated lists |
| Structured Output Parser | Parses to defined schema |

#### Retrievers
| Retriever | Description |
|-----------|-------------|
| Vector Store Retriever | Basic vector similarity search |
| MultiQuery Retriever | Generates multiple query variants |
| Contextual Compression Retriever | Compresses retrieved documents |
| Workflow Retriever | Uses another workflow for retrieval |

#### Tools
Calculator, Custom Code Tool, SearXNG, SerpApi, Wikipedia, Wolfram|Alpha, Vector Store Q&A Tool

### 13.4 Sub-Node Connection Types

Sub-nodes connect to root nodes through typed connectors:
- **Chat Model** — Language model connection
- **Memory** — Conversation memory connection
- **Tool** — Agent tool connection
- **Output Parser** — Response parsing connection
- **Embeddings** — Embedding model connection
- **Document Loader** — Document ingestion connection
- **Text Splitter** — Text chunking connection
- **Retriever** — Document retrieval connection
- **Vector Store** — Vector database connection

### 13.5 RAG (Retrieval-Augmented Generation) Patterns

#### Data Insertion Workflow
1. Fetch source data using appropriate nodes
2. Use Vector Store node with "Insert Documents" operation
3. Select embedding model to convert text -> vectors
4. Add Default Data Loader node with text splitter
5. Optionally add metadata

**Embedding model guidance:**
- Small models (e.g., `text-embedding-ada-002`): general documents
- Large models (e.g., `text-embedding-3-large`): complex topics

**Text splitting guidance:**
- Chunks of 200-500 tokens for fine-grained retrieval
- Proper overlap helps models understand context

#### Data Retrieval Methods
1. **Agent-based**: Add vector store as a tool to an AI agent
2. **Direct**: Use vector store node's "Get Many" operation with query parameters

### 13.6 Human-in-the-Loop

n8n supports human approval before AI agents execute specific tools:

- Approve or Deny tool execution
- 9 notification channels: Chat, Slack, Discord, Telegram, Microsoft Teams, Gmail, WhatsApp, Google Chat, Microsoft Outlook
- `$tool.name` — the tool node identifier
- `$tool.parameters` — AI-determined parameter values
- `$fromAI()` — function for dynamic parameter specification

**Best practice:** Include human review setup information in the system prompt so the AI understands the approval workflow.

### 13.7 LangChain Code Node

The LangChain Code node provides special built-in methods for LangChain operations. These methods are ONLY available in the LangChain Code node, not in regular Code nodes or other nodes.

### 13.8 n8n MCP Server

n8n provides an MCP (Model Context Protocol) server for AI agent access to n8n functionality.

### 13.9 AI Workflow Builder

n8n includes an AI Workflow Builder that assists in creating workflows through natural language descriptions.

### 13.10 Key Patterns & Anti-Patterns

#### Patterns (DO)

1. **Use `$json` for current item data** — it's the most common expression variable
2. **Use `$("<Node Name>").item` in expressions** for linked item access
3. **Use `$("<Node Name>").itemMatching(index)` in Code node** for item tracing
4. **Use bracket notation in Python Code node** — `item["json"]["field"]`
5. **Clone data before modifying** in custom nodes — never mutate `getInputData()` directly
6. **Use built-in HTTP helpers** instead of third-party request libraries
7. **Include `itemIndex` in error options** when processing batches
8. **Use Edit Fields (Set) node** for data transformation before more complex nodes
9. **Use `$ifEmpty(value, fallback)`** for safe null/undefined handling
10. **Use native nodes over Code node** whenever possible for performance

#### Anti-Patterns (DON'T)

1. **DON'T use `new Date()`** — use Luxon's `DateTime.fromISO()` or `$now` instead
2. **DON'T use direct buffer access** like `items[0].binary.data.data` — use `getBinaryDataBuffer()`
3. **DON'T use dot notation in Python** — it causes errors; use bracket notation
4. **DON'T import external libraries on n8n Cloud Python** — not supported
5. **DON'T use `$itemIndex` or `$secrets` in Code node** — they're not available there
6. **DON'T use `items.length` in Python** — use `len(items)` instead
7. **DON'T make HTTP requests in Code node** — use HTTP Request node
8. **DON'T access file system in Code node** — use Read/Write Files nodes
9. **DON'T rely on static data during testing** — requires active workflow with trigger
10. **DON'T store large data in workflow static data** — keep it small

---

## 14. REST API

### 14.1 Overview

The n8n Public REST API enables programmatic management of workflows, executions, credentials, users, and more. The API follows REST conventions with JSON request/response bodies.

- **Base URL**: `<N8N_HOST>:<N8N_PORT>/<N8N_PATH>/api/v1/`
- **Authentication**: API key via `X-N8N-API-KEY` header
- **Pagination**: Cursor-based (NOT offset-based)
- **Content-Type**: `application/json`
- **Availability**: Requires a paid plan (not available during free trial)

### 14.2 Authentication

API keys are created in **Settings > n8n API**:

1. Navigate to Settings > n8n API
2. Select "Create API Key"
3. Provide a label and set an expiration period
4. On enterprise plans, select specific scopes for resource access
5. Copy the generated key

**Header format:**
```
X-N8N-API-KEY: <your-api-key>
```

**Example request (self-hosted):**
```bash
curl -X 'GET' \
  '<N8N_HOST>:<N8N_PORT>/<N8N_PATH>/api/v1/workflows?active=true' \
  -H 'accept: application/json' \
  -H 'X-N8N-API-KEY: <your-api-key>'
```

Non-enterprise keys grant full access to all account resources. Enterprise instances support scope-based restrictions.

### 14.3 Pagination

- **Type**: Cursor-based pagination
- **Default page size**: 100 results
- **Maximum page size**: 250 results
- **Parameters**: `limit` (page size), `cursor` (encoded string for next page)
- **Response fields**: `data` (array of results), `nextCursor` (encoded string for next page)

**Example:**
```
GET /api/v1/workflows?active=true&limit=150
-> Response includes { data: [...], nextCursor: "MTIzNA==" }

GET /api/v1/workflows?active=true&limit=150&cursor=MTIzNA==
-> Next page of results
```

### 14.4 Complete API Endpoint Reference

All endpoints are under `/api/v1/`. Authentication required for all endpoints.

#### Workflows

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/workflows` | `workflow:list` | List workflows (paginated). Query: `offset`, `limit`, `active`, `tags`, `name`, `projectId`, `excludePinnedData` |
| POST | `/workflows` | `workflow:create` | Create a new workflow |
| GET | `/workflows/:id` | `workflow:read` | Get single workflow. Query: `excludePinnedData` |
| PUT | `/workflows/:id` | `workflow:update` | Update workflow content |
| DELETE | `/workflows/:id` | `workflow:delete` | Delete workflow permanently |
| GET | `/workflows/:id/versions/:versionId` | `workflow:read` | Get specific workflow version |
| POST | `/workflows/:id/activate` | `workflow:activate` | Activate a workflow |
| POST | `/workflows/:id/deactivate` | `workflow:deactivate` | Deactivate a workflow |
| POST | `/workflows/:id/transfer` | `workflow:move` | Transfer workflow to another project. Body: `{ destinationProjectId }` |
| GET | `/workflows/:id/tags` | `workflowTags:list` | Get workflow tags |
| PUT | `/workflows/:id/tags` | `workflowTags:update` | Update workflow tags. Body: array of tag IDs |

#### Executions

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/executions` | `execution:list` | List executions (paginated). Query: `lastId`, `limit`, `status`, `workflowId`, `projectId`, `includeData` |
| GET | `/executions/:id` | `execution:read` | Get single execution. Query: `includeData` |
| DELETE | `/executions/:id` | `execution:delete` | Delete execution (cannot delete running) |
| POST | `/executions/:id/retry` | `execution:retry` | Retry a failed execution |
| POST | `/executions/:id/stop` | `execution:stop` | Stop a running execution |
| POST | `/executions/stop` | `execution:stop` | Stop multiple executions. Body: `{ status[], workflowId?, startedAfter?, startedBefore? }` |
| GET | `/executions/:id/tags` | `executionTags:list` | Get execution tags |
| PATCH | `/executions/:id/tags` | `executionTags:update` | Update execution tags |

#### Credentials

| Method | Path | Scope | Description |
|--------|------|-------|-------------|
| GET | `/credentials` | `credential:list` | List credentials (paginated). Query: `offset`, `limit` |
| POST | `/credentials` | `credential:create` | Create new credential |
| PUT | `/credentials/:id` | `credential:update` | Update credential |
| DELETE | `/credentials/:id` | `credential:delete` | Delete credential |
| POST | `/credentials/:id/transfer` | `credential:move` | Transfer credential to another project |
| GET | `/credential-types/:credentialTypeName` | — | Get credential type schema |

#### Users

| Method | Path | Description |
|--------|------|-------------|
| GET | `/users` | List users |
| GET | `/users/:id` | Get single user |
| — | `/users/:id/role` | Manage user role |

#### Tags

| Method | Path | Description |
|--------|------|-------------|
| GET | `/tags` | List tags |
| POST | `/tags` | Create tag |
| GET | `/tags/:id` | Get single tag |
| PUT/DELETE | `/tags/:id` | Update/delete tag |

#### Variables

| Method | Path | Description |
|--------|------|-------------|
| GET | `/variables` | List variables |
| POST | `/variables` | Create variable |
| GET | `/variables/:id` | Get single variable |
| PUT/DELETE | `/variables/:id` | Update/delete variable |

#### Projects

| Method | Path | Description |
|--------|------|-------------|
| GET | `/projects` | List projects |
| POST | `/projects` | Create project |
| GET | `/projects/:projectId` | Get single project |
| PUT/DELETE | `/projects/:projectId` | Update/delete project |
| GET | `/projects/:projectId/users` | List project users |
| PUT | `/projects/:projectId/users/:userId` | Manage project user |

#### Other

| Method | Path | Description |
|--------|------|-------------|
| POST | `/audit` | Run security audit |
| POST | `/source-control/pull` | Pull from source control |
| CRUD | `/data-tables/*` | Data table operations (rows, update, upsert, delete) |

### 14.5 API Configuration Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_PUBLIC_API_DISABLED` | `false` | Disable public API entirely |
| `N8N_PUBLIC_API_ENDPOINT` | `api` | Path for public API endpoints |
| `N8N_PUBLIC_API_SWAGGERUI_DISABLED` | `false` | Disable Swagger UI (API playground) |

The API playground (Scalar UI) is available at `<base-url>/api/v1/docs` on self-hosted instances.

---

## 15. Docker Deployment

### 15.1 Basic Docker Run

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Europe/Amsterdam" \
  -e TZ="Europe/Amsterdam" \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_RUNNERS_ENABLED=true \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

**Key points:**
- Docker image: `docker.n8n.io/n8nio/n8n`
- Default port: 5678
- Data volume: `/home/node/.n8n` (contains SQLite DB, encryption key, settings)
- Access: `http://localhost:5678`

### 15.2 Docker Run with PostgreSQL

```bash
docker volume create n8n_data

docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Europe/Amsterdam" \
  -e TZ="Europe/Amsterdam" \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_RUNNERS_ENABLED=true \
  -e DB_TYPE=postgresdb \
  -e DB_POSTGRESDB_DATABASE=n8n \
  -e DB_POSTGRESDB_HOST=postgres-host \
  -e DB_POSTGRESDB_PORT=5432 \
  -e DB_POSTGRESDB_USER=postgres \
  -e DB_POSTGRESDB_SCHEMA=public \
  -e DB_POSTGRESDB_PASSWORD=secret \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

### 15.3 Docker Compose with Traefik (Production)

**`.env` file:**
```env
DOMAIN_NAME=example.com
SUBDOMAIN=n8n
GENERIC_TIMEZONE=Europe/Berlin
SSL_EMAIL=user@example.com
```

**`docker-compose.yml`:**
```yaml
services:
  traefik:
    image: traefik:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
    # Traefik labels for HTTPS redirect, Let's Encrypt ACME

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files

volumes:
  traefik_data:
  n8n_data:
```

**Launch:**
```bash
sudo docker compose up -d
```

### 15.4 Volume Mounts

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `n8n_data` | `/home/node/.n8n` | SQLite database, encryption key, settings |
| `traefik_data` | `/letsencrypt` | TLS/SSL certificates (Let's Encrypt) |
| `./local-files` | `/files` | Host directory for file sharing with workflows |

### 15.5 Updating Docker

```bash
# Pull latest version
docker pull docker.n8n.io/n8nio/n8n

# Pull specific version
docker pull docker.n8n.io/n8nio/n8n:1.81.0

# Pull next (pre-release) version
docker pull docker.n8n.io/n8nio/n8n:next

# Replace running container
docker ps -a
docker stop <container_id>
docker rm <container_id>
docker run --name=<container_name> [options] -d docker.n8n.io/n8nio/n8n
```

### 15.6 Configuration Methods

n8n supports three configuration methods:

1. **Environment variables** (primary method)
   - Bash: `export N8N_TEMPLATES_ENABLED=false`
   - Docker CLI: `-e N8N_TEMPLATES_ENABLED=false`
   - Docker Compose: under `environment:` section

2. **Docker Compose file** — variables in `docker-compose.yml`

3. **`_FILE` suffix** — load sensitive values from files (Docker Secrets, Kubernetes Secrets)
   - Example: `DB_POSTGRESDB_PASSWORD_FILE=/run/secrets/db_password`
   - Supported on most credential/connection variables

---

## 16. Queue Mode & Scaling

### 16.1 Architecture

Queue mode separates n8n into distinct components:
- **Main instance**: Handles triggers, webhooks, generates execution IDs (does NOT run workflows)
- **Worker instances**: Execute workflows retrieved from the queue
- **Redis**: Message broker for the execution queue (BullMQ)
- **Database**: Stores persistent data (PostgreSQL required)

**Flow:**
1. Main instance generates execution ID (without running the workflow)
2. Execution ID is passed to Redis queue
3. Available worker retrieves message from Redis
4. Worker fetches workflow details from database
5. Worker executes workflow and writes results back to database
6. Worker notifies Redis of completion

### 16.2 Requirements

- **PostgreSQL 13+** required (SQLite NOT supported with queue mode)
- **Redis** for message queue
- **S3-compatible storage** required for binary data in queue mode
- All instances MUST share the same n8n version
- All instances MUST share the same `N8N_ENCRYPTION_KEY`
- Load balancer with **session persistence** required for multi-main setups

### 16.3 Environment Variables

**Core queue configuration:**

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_MODE` | `regular` | Set to `queue` to enable queue mode |
| `N8N_ENCRYPTION_KEY` | random | Shared encryption key across ALL instances |

**Redis connection:**

| Variable | Default | Description |
|----------|---------|-------------|
| `QUEUE_BULL_REDIS_HOST` | `localhost` | Redis host |
| `QUEUE_BULL_REDIS_PORT` | `6379` | Redis port |
| `QUEUE_BULL_REDIS_USERNAME` | — | Redis username (optional) |
| `QUEUE_BULL_REDIS_PASSWORD` | — | Redis password (optional) |
| `QUEUE_BULL_REDIS_DB` | `0` | Redis database number |
| `QUEUE_BULL_REDIS_TIMEOUT_THRESHOLD` | `10000` | Redis unavailability tolerance (ms) |

**Webhook and process configuration:**

| Variable | Default | Description |
|----------|---------|-------------|
| `WEBHOOK_URL` | — | Webhook routing URL |
| `N8N_DISABLE_PRODUCTION_MAIN_PROCESS` | `false` | Disable webhook processing on main |
| `N8N_MULTI_MAIN_SETUP_ENABLED` | `false` | Enable multi-main deployment |
| `QUEUE_HEALTH_CHECK_ACTIVE` | — | Enable health check endpoints |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Worker shutdown grace period (seconds) |

### 16.4 Redis Setup

```bash
docker run --name some-redis -p 6379:6379 -d redis
```

### 16.5 Worker Commands

**Native:**
```bash
n8n worker
n8n worker --concurrency=5
```

**Docker:**
```bash
docker run --name n8n-worker \
  -e EXECUTIONS_MODE=queue \
  -e QUEUE_BULL_REDIS_HOST=redis-host \
  -e N8N_ENCRYPTION_KEY=<shared-key> \
  docker.n8n.io/n8nio/n8n worker
```

### 16.6 Webhook Processor

Separate webhook processor for high-traffic setups:

**Native:**
```bash
n8n webhook
```

**Docker:**
```bash
docker run --name n8n-webhook -p 5679:5678 \
  -e EXECUTIONS_MODE=queue \
  docker.n8n.io/n8nio/n8n webhook
```

---

## 17. Webhooks

### 17.1 Webhook Node

The Webhook node is a trigger node that listens for incoming HTTP requests.

**Supported HTTP methods:** DELETE, GET, HEAD, PATCH, POST, PUT

### 17.2 Test vs Production URLs

| Type | When Active | Data Display | URL Path |
|------|-------------|--------------|----------|
| **Test URL** | During "Listen for Test Event" or manual execution | Shown in workflow editor | `<base>/webhook-test/<path>` |
| **Production URL** | When workflow is published/active | Only in Executions tab | `<base>/webhook/<path>` |

**URL endpoint configuration variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ENDPOINT_WEBHOOK` | `webhook` | Production webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_TEST` | `webhook-test` | Test webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_WAIT` | `webhook-waiting` | Waiting webhook path prefix |

### 17.3 Path Configuration

The Path field supports dynamic route parameters:
- `/:variable`
- `/path/:variable`
- `/:variable/path`
- `/:variable1/path/:variable2`
- `/:variable1/:variable2`

### 17.4 Response Modes

| Mode | Behavior |
|------|----------|
| **Immediately** | Returns response code and "Workflow got started" message |
| **When Last Node Finishes** | Returns response code and final node's data |
| **Using 'Respond to Webhook' Node** | Responds per Respond to Webhook node configuration |
| **Streaming** | Real-time data streaming for streaming-capable nodes |

**Response Data options (for "When Last Node Finishes"):**
- **All Entries**: Array of all last node entries
- **First Entry JSON**: JSON object of first entry
- **First Entry Binary**: Binary file of first entry
- **No Response Body**: Empty response

### 17.5 Authentication Options

| Method | Description |
|--------|-------------|
| **None** | No authentication |
| **Basic Auth** | Username/password authentication |
| **Header Auth** | Custom header-based authentication |
| **JWT Auth** | JSON Web Token authentication |

### 17.6 Additional Options

| Option | Description |
|--------|-------------|
| **Binary Data** | Enable binary property reception (images, audio, files) |
| **IP Whitelist** | Comma-separated allowed IPs; unlisted IPs receive 403 |
| **CORS** | Cross-origin domains; `*` allows all (default) |
| **Ignore Bots** | Filter out bot requests |
| **Raw Body** | Receive unformatted data (JSON, XML) |
| **Response Headers** | Custom headers in webhook responses |
| **Response Code** | Custom HTTP status codes |
| **Property Name** | Return specific JSON keys instead of all data |

**Payload limit**: 16MB maximum (configurable via `N8N_PAYLOAD_SIZE_MAX`)

### 17.7 Respond to Webhook Node

The Respond to Webhook node controls what data is sent back to the webhook caller. It pairs with the Webhook trigger node.

**Setup:**
1. Add a Webhook node as workflow trigger
2. Set Webhook node's **Respond** parameter to "Using 'Respond to Webhook' node"
3. Place the Respond to Webhook node after processing nodes

**Respond With options:**

| Option | Description |
|--------|-------------|
| **All Incoming Items** | Return all JSON items from input |
| **Binary File** | Return a binary file |
| **First Incoming Item** | Return first item's JSON |
| **JSON** | Return custom JSON from Response Body |
| **JWT Token** | Return a JSON Web Token |
| **No Data** | Empty response payload |
| **Redirect** | Redirect to specified URL |
| **Text** | Return text (sent as HTML by default) |

**Additional options:**
- **Response Code**: Custom HTTP status code
- **Response Headers**: Custom response headers
- **Put Response in Field**: Rename response data field
- **Enable Streaming**: For streaming-configured triggers

**Behavior notes:**
- The node runs ONCE, using the first incoming data item
- Skipping the node returns 200 with standard message
- Pre-execution errors return 500
- Multiple Respond nodes: only the first executes
- Non-webhook contexts: node is ignored
- Since v1.103.0: HTML responses are wrapped in `<iframe>` for security (sandboxing prevents JS access to top-level window and local storage)

---

## 18. Error Handling

### 18.1 Error Workflows

Each workflow can designate a dedicated error workflow in **Workflow Settings**. When the primary workflow fails, the error workflow automatically executes. One error workflow can serve multiple workflows.

**Setup:**
1. Create a new workflow starting with the **Error Trigger** node
2. Add notification/recovery logic (email, Slack, database logging, etc.)
3. In the main workflow, open **Workflow Settings**
4. Select the error workflow from the dropdown

The Error Trigger node receives error data containing execution context information.

### 18.2 Stop And Error Node

The **Stop And Error** node deliberately causes a workflow execution failure. This triggers the configured error workflow.

**Use cases:**
- Custom validation failures
- Business logic violations
- Forcing error workflow activation based on data conditions

### 18.3 Error Investigation

- Review executions for individual or multiple workflows
- Load data from previous executions for debugging
- Enable Log Streaming for real-time monitoring

### 18.4 Retry on Fail

Individual nodes support retry-on-fail configuration in their **Settings** tab:

- **Retry On Fail**: Enable/disable (boolean)
- **Max Tries**: Number of retry attempts (default: 3)
- **Wait Between Tries (ms)**: Delay between retries (default: 1000ms)

### 18.5 Continue On Fail

Each node has a **Continue On Fail** setting (in Settings tab) that, when enabled, allows the workflow to continue even if that node fails. The error data is passed to the next node in `$json.error` instead of stopping execution.

### 18.6 Execution Timeouts

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_TIMEOUT` | `-1` (disabled) | Default timeout for all workflows (seconds) |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Maximum timeout users can set per workflow (seconds) |

### 18.7 Error Handling Patterns

**Pattern 1: Error Workflow with Notifications**
```
Main Workflow -> (fails) -> Error Trigger -> Slack/Email notification
```

**Pattern 2: Graceful Degradation**
```
HTTP Request (continueOnFail=true) -> IF ($json.error) -> Fallback path
                                                        -> Success path
```

**Pattern 3: Retry with Fallback**
```
API Call (retryOnFail=true, maxTries=3) -> (still fails) -> Error Workflow
```

**Pattern 4: Deliberate Failure**
```
IF (invalid data) -> Stop And Error -> triggers Error Workflow
```

---

## 19. Environment Variables & Security

### 19.1 Deployment & Core

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_HOST` | `localhost` | Hostname |
| `N8N_PORT` | `5678` | HTTP port |
| `N8N_PROTOCOL` | `http` | Protocol (`http` or `https`) |
| `N8N_PATH` | `/` | Deployment path |
| `N8N_LISTEN_ADDRESS` | `::` | IP address to listen on |
| `N8N_ENCRYPTION_KEY` | random | Encryption key for credentials in database |
| `N8N_USER_FOLDER` | `user-folder` | Path for `.n8n` data folder |
| `N8N_EDITOR_BASE_URL` | — | Public URL for editor access |
| `N8N_DISABLE_UI` | `false` | Disable the web UI |
| `N8N_PUSH_BACKEND` | `websocket` | SSE or WebSocket for UI updates |
| `NODE_ENV` | — | Set to `production` for production deployments |
| `N8N_GRACEFUL_SHUTDOWN_TIMEOUT` | `30` | Shutdown grace period (seconds) |
| `N8N_PROXY_HOPS` | `0` | Number of reverse proxies in front of n8n |

### 19.2 Database

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_TYPE` | `sqlite` | Database type: `sqlite` or `postgresdb` |
| `DB_TABLE_PREFIX` | — | Table name prefix |
| `DB_PING_INTERVAL_SECONDS` | `2` | Database ping interval |
| `DB_POSTGRESDB_DATABASE` | `n8n` | PostgreSQL database name |
| `DB_POSTGRESDB_HOST` | `localhost` | PostgreSQL host |
| `DB_POSTGRESDB_PORT` | `5432` | PostgreSQL port |
| `DB_POSTGRESDB_USER` | `postgres` | PostgreSQL user |
| `DB_POSTGRESDB_PASSWORD` | — | PostgreSQL password |
| `DB_POSTGRESDB_SCHEMA` | `public` | PostgreSQL schema |
| `DB_POSTGRESDB_POOL_SIZE` | `2` | Connection pool size |
| `DB_POSTGRESDB_CONNECTION_TIMEOUT` | `20000` | Connection timeout (ms) |
| `DB_POSTGRESDB_IDLE_CONNECTION_TIMEOUT` | `30000` | Idle connection timeout (ms) |
| `DB_POSTGRESDB_SSL_ENABLED` | `false` | Enable SSL |
| `DB_POSTGRESDB_SSL_CA` | — | SSL CA certificate |
| `DB_POSTGRESDB_SSL_CERT` | — | SSL certificate |
| `DB_POSTGRESDB_SSL_KEY` | — | SSL key |
| `DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED` | `true` | Reject unauthorized SSL |
| `DB_SQLITE_POOL_SIZE` | `0` | SQLite pool (0=journal, >0=WAL mode) |
| `DB_SQLITE_VACUUM_ON_STARTUP` | `false` | Optimize SQLite on startup |

### 19.3 Execution

| Variable | Default | Description |
|----------|---------|-------------|
| `EXECUTIONS_MODE` | `regular` | `regular` or `queue` |
| `EXECUTIONS_TIMEOUT` | `-1` | Default workflow timeout (seconds) |
| `EXECUTIONS_TIMEOUT_MAX` | `3600` | Max timeout per workflow (seconds) |
| `EXECUTIONS_DATA_SAVE_ON_ERROR` | `all` | Save execution data on error |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS` | `all` | Save execution data on success |
| `EXECUTIONS_DATA_SAVE_ON_PROGRESS` | `false` | Save progress per node |
| `EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS` | `true` | Save manual execution data |
| `EXECUTIONS_DATA_PRUNE` | `true` | Auto-delete old executions |
| `EXECUTIONS_DATA_MAX_AGE` | `336` | Max execution age (hours) = 14 days |
| `EXECUTIONS_DATA_PRUNE_MAX_COUNT` | `10000` | Max execution count before pruning |
| `EXECUTIONS_DATA_HARD_DELETE_BUFFER` | `1` | Hours before hard deletion |
| `EXECUTIONS_DATA_PRUNE_HARD_DELETE_INTERVAL` | `15` | Hard delete interval (minutes) |
| `EXECUTIONS_DATA_PRUNE_SOFT_DELETE_INTERVAL` | `60` | Soft delete interval (minutes) |
| `N8N_CONCURRENCY_PRODUCTION_LIMIT` | `-1` | Max concurrent production executions |
| `N8N_AI_TIMEOUT_MAX` | `3600000` | AI/LLM node HTTP timeout (ms) |

### 19.4 Credentials

| Variable | Default | Description |
|----------|---------|-------------|
| `CREDENTIALS_OVERWRITE_DATA` | — | Overwrite data for credentials |
| `CREDENTIALS_OVERWRITE_ENDPOINT` | — | API endpoint to fetch credentials |
| `CREDENTIALS_OVERWRITE_PERSISTENCE` | `false` | Enable DB persistence for overwrites (required for queue mode) |
| `CREDENTIALS_DEFAULT_NAME` | `My credentials` | Default credential name |

### 19.5 Endpoints & Webhooks

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ENDPOINT_REST` | `rest` | REST API path |
| `N8N_ENDPOINT_WEBHOOK` | `webhook` | Production webhook path |
| `N8N_ENDPOINT_WEBHOOK_TEST` | `webhook-test` | Test webhook path |
| `N8N_ENDPOINT_WEBHOOK_WAIT` | `webhook-waiting` | Waiting webhook path |
| `N8N_ENDPOINT_HEALTH` | `healthz` | Health check endpoint path |
| `WEBHOOK_URL` | — | Manual webhook URL (for reverse proxies) |
| `N8N_DISABLE_PRODUCTION_MAIN_PROCESS` | `false` | Disable production webhooks on main |
| `N8N_PAYLOAD_SIZE_MAX` | — | Maximum request payload size |
| `N8N_FORMDATA_FILE_SIZE_MAX` | — | Maximum form data file size |

### 19.6 Security

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Block environment variable access in expressions/Code node |
| `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES` | `true` | Block access to `.n8n` directory |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` | `false` | Set 0600 permissions on settings file |
| `N8N_RESTRICT_FILE_ACCESS_TO` | — | Limit file access to specified directories (semicolon-separated) |
| `N8N_SECURITY_AUDIT_DAYS_ABANDONED_WORKFLOW` | `90` | Days before workflow considered abandoned |
| `N8N_CONTENT_SECURITY_POLICY` | `{}` | CSP headers (helmet.js format) |
| `N8N_SECURE_COOKIE` | `true` | HTTPS-only cookies |
| `N8N_SAMESITE_COOKIE` | `lax` | Cross-site cookie policy (`strict`, `lax`, `none`) |
| `N8N_GIT_NODE_DISABLE_BARE_REPOS` | `false` | Prevent Git node bare repos |
| `N8N_GIT_NODE_ENABLE_HOOKS` | `false` | Allow Git node hooks |

### 19.7 SSL/TLS

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_SSL_KEY` | — | SSL key file path |
| `N8N_SSL_CERT` | — | SSL certificate file path |

### 19.8 Node Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_COMMUNITY_PACKAGES_ENABLED` | `true` | Enable community node installation |
| `N8N_COMMUNITY_PACKAGES_PREVENT_LOADING` | `false` | Prevent loading installed community nodes |
| `N8N_COMMUNITY_PACKAGES_REGISTRY` | `https://registry.npmjs.org` | NPM registry for community packages |
| `N8N_CUSTOM_EXTENSIONS` | — | Path to custom node directories |
| `N8N_PYTHON_ENABLED` | `true` | Enable Python in Code node |
| `NODE_FUNCTION_ALLOW_BUILTIN` | — | Allowed built-in modules in Code node (`*` for all) |
| `NODE_FUNCTION_ALLOW_EXTERNAL` | — | Allowed external modules in Code node |
| `NODES_ERROR_TRIGGER_TYPE` | `n8n-nodes-base.errorTrigger` | Error trigger node type |
| `NODES_EXCLUDE` | `["n8n-nodes-base.executeCommand", "n8n-nodes-base.localFileTrigger"]` | Nodes to exclude from loading |
| `NODES_INCLUDE` | — | Nodes to include (whitelist) |

### 19.9 Workflow Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_ONBOARDING_FLOW_DISABLED` | `false` | Disable onboarding tips |
| `N8N_WORKFLOW_ACTIVATION_BATCH_SIZE` | `1` | Workflows activated concurrently at startup |
| `N8N_WORKFLOW_CALLER_POLICY_DEFAULT_OPTION` | `workflowsFromSameOwner` | Default sub-workflow caller policy |
| `N8N_WORKFLOW_TAGS_DISABLED` | `false` | Disable workflow tags |
| `WORKFLOWS_DEFAULT_NAME` | `My workflow` | Default workflow name |
| `N8N_WORKFLOW_AUTODEACTIVATION_ENABLED` | `false` | Auto-unpublish after repeated crashes |
| `N8N_WORKFLOW_AUTODEACTIVATION_MAX_LAST_EXECUTIONS` | `3` | Crash count before auto-deactivation |

### 19.10 Proxy

| Variable | Default | Description |
|----------|---------|-------------|
| `HTTP_PROXY` | — | Proxy for HTTP requests |
| `HTTPS_PROXY` | — | Proxy for HTTPS requests |
| `ALL_PROXY` | — | Proxy for all requests |
| `NO_PROXY` | — | Comma-separated hostnames to bypass proxy |

### 19.11 Binary Data & External Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_DEFAULT_BINARY_DATA_MODE` | `default` | Storage mode: `default` (memory), `filesystem`, `s3` |
| `N8N_AVAILABLE_BINARY_DATA_MODES` | `filesystem` | Comma-separated available modes |
| `N8N_BINARY_DATA_STORAGE_PATH` | `<N8N_USER_FOLDER>/binaryData` | Filesystem storage path |
| `N8N_EXTERNAL_STORAGE_S3_HOST` | — | S3 host |
| `N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME` | — | S3 bucket name |
| `N8N_EXTERNAL_STORAGE_S3_BUCKET_REGION` | — | S3 bucket region |
| `N8N_EXTERNAL_STORAGE_S3_ACCESS_KEY` | — | S3 access key |
| `N8N_EXTERNAL_STORAGE_S3_ACCESS_SECRET` | — | S3 access secret |
| `N8N_EXTERNAL_STORAGE_S3_AUTH_AUTO_DETECT` | — | Use AWS credential provider chain |

### 19.12 Task Runners

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_RUNNERS_ENABLED` | `false` | Enable task runners |
| `N8N_RUNNERS_MODE` | `internal` | `internal` or `external` |
| `N8N_RUNNERS_AUTH_TOKEN` | — | Shared auth secret for runners |
| `N8N_RUNNERS_BROKER_PORT` | `5679` | Broker connection port |
| `N8N_RUNNERS_BROKER_LISTEN_ADDRESS` | `127.0.0.1` | Broker listen address |
| `N8N_RUNNERS_MAX_PAYLOAD` | `1073741824` | Max communication payload (bytes) |
| `N8N_RUNNERS_MAX_OLD_SPACE_SIZE` | — | Node.js memory allocation (MB) |
| `N8N_RUNNERS_MAX_CONCURRENCY` | `5` | Concurrent task capacity |
| `N8N_RUNNERS_TASK_TIMEOUT` | `300` | Task execution limit (seconds) |
| `N8N_RUNNERS_HEARTBEAT_INTERVAL` | `30` | Health check frequency (seconds) |

### 19.13 Telemetry & Misc

| Variable | Default | Description |
|----------|---------|-------------|
| `GENERIC_TIMEZONE` | — | Timezone for schedule nodes |
| `TZ` | — | System timezone |
| `N8N_TEMPLATES_ENABLED` | `false` | Enable workflow templates |
| `N8N_TEMPLATES_HOST` | `https://api.n8n.io` | Custom template library endpoint |
| `N8N_DIAGNOSTICS_ENABLED` | `true` | Enable anonymous telemetry |
| `N8N_VERSION_NOTIFICATIONS_ENABLED` | `true` | Version update notifications |
| `N8N_PERSONALIZATION_ENABLED` | `true` | Personalization questions |
| `N8N_HIRING_BANNER_ENABLED` | `true` | Hiring banner visibility |
| `N8N_REINSTALL_MISSING_PACKAGES` | `false` | Auto-reinstall missing packages |
| `N8N_TUNNEL_SUBDOMAIN` | — | Custom tunnel subdomain |
| `N8N_DEV_RELOAD` | `false` | Auto-reload during development |
| `N8N_PREVIEW_MODE` | `false` | Preview mode |
| `VUE_APP_URL_BASE_API` | `http://localhost:5678/` | Frontend API endpoint |

### 19.14 Metrics

| Variable | Default | Description |
|----------|---------|-------------|
| `N8N_METRICS` | — | Enable Prometheus metrics |
| `N8N_METRICS_PREFIX` | — | Metrics name prefix |
| `N8N_METRICS_INCLUDE_*` | — | Various include flags for metric types |

---

## 20. CLI & Backup

### 20.1 Workflow Execution

```bash
# Execute a saved workflow by ID
n8n execute --id <ID>
```

### 20.2 Workflow Status Management

```bash
# Activate/deactivate single workflow
n8n update:workflow --id=<ID> --active=false
n8n update:workflow --id=<ID> --active=true

# Activate/deactivate all workflows
n8n update:workflow --all --active=false
n8n update:workflow --all --active=true
```

### 20.3 Workflow Export

```bash
# Export all workflows to terminal
n8n export:workflow --all

# Export by ID to file
n8n export:workflow --id=<ID> --output=file.json

# Export all to directory
n8n export:workflow --all --output=backups/latest/file.json

# Backup (convenience flag)
n8n export:workflow --backup --output=backups/latest/

# Flags: --help, --all, --backup, --id, --output, --pretty, --separate
```

### 20.4 Credential Export

```bash
# Export all credentials (encrypted)
n8n export:credentials --all

# Export by ID
n8n export:credentials --id=<ID> --output=file.json

# Export all to directory
n8n export:credentials --all --output=backups/latest/file.json

# Backup credentials
n8n export:credentials --backup --output=backups/latest/

# Export DECRYPTED credentials (security risk!)
n8n export:credentials --all --decrypted --output=backups/decrypted.json

# Flags: --help, --all, --backup, --id, --output, --pretty, --separate, --decrypted
```

### 20.5 Entity Export (Combined)

```bash
# Export all entities with execution history
n8n export:entities --outputDir=./outputs --includeExecutionHistoryDataTables=true

# Flags: --help, --outputDir, --includeExecutionHistoryDataTables
```

### 20.6 Workflow Import

```bash
# Import from file
n8n import:workflow --input=file.json

# Import from directory
n8n import:workflow --separate --input=backups/latest/

# Flags: --help, --input, --projectId, --separate, --userId, --skipMigrationChecks
```

### 20.7 Credential Import

```bash
# Import from file
n8n import:credentials --input=file.json

# Import from directory
n8n import:credentials --separate --input=backups/latest/
```

### 20.8 Entity Import (Combined)

```bash
# Import all entities
n8n import:entities --inputDir ./outputs --truncateTables true

# Flags: --help, --inputDir, --truncateTables
```

### 20.9 License Management

```bash
# Clear license
n8n license:clear

# Display license info
n8n license:info
```

### 20.10 User Management

```bash
# Reset user management
n8n user-management:reset

# Disable MFA for a user
n8n mfa:disable --email=johndoe@example.com

# Reset LDAP settings
n8n ldap:reset
```

### 20.11 Community Node Management

```bash
# Uninstall community node
n8n community-node --uninstall --package <COMMUNITY_NODE_NAME>

# Uninstall community node credential
n8n community-node --uninstall --credential <CREDENTIAL_TYPE> --userId <ID>
```

### 20.12 Security Audit

```bash
# Run security audit
n8n audit
```

### 20.13 Backup Strategies

**Strategy 1: CLI Export (Recommended for small instances)**
```bash
# Daily backup script
n8n export:workflow --backup --output=/backups/workflows/
n8n export:credentials --backup --output=/backups/credentials/
```

**Strategy 2: Database Backup (Recommended for production)**
```bash
# PostgreSQL dump
pg_dump -U postgres n8n > /backups/n8n-$(date +%Y%m%d).sql

# With Docker
docker exec postgres pg_dump -U postgres n8n > /backups/n8n-$(date +%Y%m%d).sql
```

**Strategy 3: Volume Backup (SQLite)**
```bash
# Stop n8n, backup volume
docker stop n8n
docker run --rm -v n8n_data:/data -v $(pwd)/backup:/backup alpine tar czf /backup/n8n-data.tar.gz /data
docker start n8n
```

**Critical backup items:**
- Workflow definitions (JSON)
- Credentials (encrypted — need same `N8N_ENCRYPTION_KEY` to restore)
- `N8N_ENCRYPTION_KEY` itself (stored in `.n8n/config` or as env var)
- Database (PostgreSQL dump or SQLite file)
- Binary data (filesystem or S3)

---

## Appendix: Quick Reference

### Default Ports

| Service | Port | Description |
|---------|------|-------------|
| n8n | 5678 | Main HTTP port |
| Redis | 6379 | Queue message broker |
| PostgreSQL | 5432 | Database |
| Task Runner Broker | 5679 | Task runner communication |
| Launcher Health Check | 5680 | Task runner health |

### Minimum Production Setup Checklist

1. PostgreSQL database (not SQLite)
2. `N8N_ENCRYPTION_KEY` explicitly set and backed up
3. `NODE_ENV=production`
4. `N8N_PROTOCOL=https` with TLS termination
5. `WEBHOOK_URL` set to public URL
6. `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`
7. `N8N_RUNNERS_ENABLED=true`
8. Execution data pruning configured
9. Regular backups (database + encryption key)
10. Reverse proxy (Traefik, nginx, Caddy) for TLS

### SQLite vs PostgreSQL Decision

| Factor | SQLite | PostgreSQL |
|--------|--------|------------|
| Setup complexity | Minimal | Moderate |
| Queue mode support | NO | YES |
| Concurrent access | Limited | Full |
| Production recommended | No (single user/dev) | Yes |
| Backup | File copy | pg_dump |
| Scaling | Single instance only | Multi-instance |

---

## Sources

### Fragment 1: Core Architecture
- **Primary**: `n8n-io/n8n` GitHub repository, `packages/workflow/src/interfaces.ts` (master branch)
- **Execution context**: `packages/workflow/src/execution-context.ts`
- **Node examples**: `packages/nodes-base/nodes/` (HttpRequest, Set, Schedule, Webhook)
- **Community starter**: `n8n-io/n8n-nodes-starter` repository
- **Documentation**: `https://docs.n8n.io/` (navigation structure confirmed topics exist)

### Fragment 2: Expressions, Credentials & Data
| Source | Type | URL |
|--------|------|-----|
| n8n-docs GitHub (main) | Raw markdown | github.com/n8n-io/n8n-docs/main/docs/ |
| n8n Code node snippet | Source | _snippets/integrations/builtin/core-nodes/code-node.md |
| n8n Expression reference | Docs | docs/data/expression-reference/*.md |
| n8n Credential files ref | Docs | docs/integrations/creating-nodes/build/reference/credentials-files.md |
| n8n UI Elements ref | Docs | docs/integrations/creating-nodes/build/reference/ui-elements.md |
| n8n Error handling ref | Docs | docs/integrations/creating-nodes/build/reference/error-handling.md |
| n8n HTTP helpers ref | Docs | docs/integrations/creating-nodes/build/reference/http-helpers.md |
| n8n Code standards ref | Docs | docs/integrations/creating-nodes/build/reference/code-standards.md |
| n8n Advanced AI | Docs | docs/advanced-ai/*.md |
| n8n JMESPath | Docs | docs/data/specific-data-types/jmespath.md |
| n8n Luxon | Docs | docs/data/specific-data-types/luxon.md |
| n8n Binary data | Docs | docs/data/specific-data-types/binary-data.md |
| n8n Data structure | Docs | docs/data/data-structure.md |
| n8n Data mapping | Docs | docs/data/data-mapping/*.md |
| n8n Code node source | Source | packages/nodes-base/nodes/Code/Code.node.ts |

### Fragment 3: REST API, Deployment & Operations
- Official n8n documentation (docs.n8n.io)
- n8n GitHub repository (github.com/n8n-io/n8n)
