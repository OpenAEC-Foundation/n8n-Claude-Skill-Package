# Trigger Node Methods Reference

> Complete interface definitions for all three trigger patterns in n8n v1.x.

## ITriggerFunctions

Available as `this` inside the `trigger()` method of event/timer trigger nodes.

```typescript
export interface ITriggerFunctions
    extends FunctionsBaseWithRequiredKeys<'getMode' | 'getActivationMode'> {

    // Emit data to start workflow execution
    // CRITICAL: data MUST be double-wrapped — [[items]]
    emit(
        data: INodeExecutionData[][],
        responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>,
        donePromise?: IDeferredPromise<IRun>,
    ): void;

    // Report non-fatal execution error (workflow stays active)
    saveFailedExecution(error: ExecutionError): void;

    // Report fatal error (deactivates workflow, queues for reactivation with backoff)
    emitError(
        error: Error,
        responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>,
    ): void;

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

### Key ITriggerFunctions Methods

| Method | Purpose | Notes |
|--------|---------|-------|
| `emit(data)` | Send items into the workflow | ALWAYS double-wrap: `[[items]]` |
| `saveFailedExecution(error)` | Log error without deactivating | Workflow stays active |
| `emitError(error)` | Fatal error — deactivates workflow | Triggers reactivation backoff |
| `getNodeParameter(name, itemIndex?)` | Read node parameter value | Same as regular nodes |
| `getMode()` | Returns `'manual'` or `'trigger'` | Use to branch manual vs production |
| `getActivationMode()` | Returns activation reason | `'init'`, `'create'`, `'update'`, `'activate'`, `'manual'` |

### SchedulingFunctions (via helpers)

```typescript
interface SchedulingFunctions {
    registerCron(cronExpression: string, onTick: () => void): void;
}
```

Use `this.helpers.registerCron()` for cron-based schedules. n8n handles cleanup automatically — no `closeFunction` needed for cron-registered triggers.

### returnJsonArray Helper

```typescript
// Converts plain objects to INodeExecutionData[]
this.helpers.returnJsonArray([{ id: 1 }, { id: 2 }]);
// Returns: [{ json: { id: 1 } }, { json: { id: 2 } }]
```

ALWAYS use `returnJsonArray` when converting plain objects to n8n items.

---

## ITriggerResponse

Returned from the `trigger()` method. Defines cleanup and manual execution behavior.

```typescript
export interface ITriggerResponse {
    // Cleanup function — called when workflow is deactivated
    closeFunction?: CloseFunction;  // () => Promise<void>

    // Function to simulate trigger during "Test Workflow" in editor
    manualTriggerFunction?: () => Promise<void>;

    // Alternative: promise that resolves with emitted data for manual mode
    manualTriggerResponse?: Promise<INodeExecutionData[][]>;
}
```

### closeFunction

- ALWAYS implement when your trigger allocates persistent resources
- Called when: workflow deactivated, n8n shuts down, workflow updated
- Must clean up: intervals, timeouts, event listeners, WebSocket connections, file watchers

```typescript
// Example: cleaning up an interval
const intervalId = setInterval(check, 60_000);
return {
    closeFunction: async () => {
        clearInterval(intervalId);
    },
};
```

### manualTriggerFunction vs manualTriggerResponse

| Approach | When to Use |
|----------|-------------|
| `manualTriggerFunction` | Trigger can produce data immediately or on-demand |
| `manualTriggerResponse` | Trigger needs to wait for an external event even during test |

```typescript
// manualTriggerFunction — immediate emission
return {
    manualTriggerFunction: async () => {
        this.emit([[ { json: { test: true } } ]]);
    },
};

// manualTriggerResponse — deferred emission
return {
    manualTriggerResponse: new Promise((resolve) => {
        someEmitter.once('data', (data) => {
            resolve([[ { json: data } ]]);
        });
    }),
};
```

---

## IPollFunctions

Available as `this` inside the `poll()` method of polling trigger nodes.

```typescript
export interface IPollFunctions
    extends FunctionsBaseWithRequiredKeys<'getMode' | 'getActivationMode'> {

    // Internal emit — used by the polling framework (DO NOT call directly)
    __emit(
        data: INodeExecutionData[][],
        responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>,
        donePromise?: IDeferredPromise<IRun>,
    ): void;

    // Internal error emit (DO NOT call directly)
    __emitError(
        error: Error,
        responsePromise?: IDeferredPromise<IExecuteResponsePromiseData>,
    ): void;

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
        SchedulingFunctions;
}
```

### Key Differences from ITriggerFunctions

- **No `emit()` method** — polling triggers return data directly from `poll()`
- **No `closeFunction`** — n8n manages the poll interval lifecycle
- Return `INodeExecutionData[][] | null` — return `null` when no new data
- NEVER return an empty array `[]` — ALWAYS return `null` for "no data"

### Static Data for Deduplication

```typescript
// Get persistent storage scoped to this node
const staticData = this.getWorkflowStaticData('node');

// Read previous state
const lastId = staticData.lastId as string | undefined;

// Update state (persisted across poll cycles and workflow restarts)
staticData.lastId = latestItem.id;
```

---

## IWebhookFunctions

Available as `this` (classic) or `context` (Node class) inside the `webhook()` method.

```typescript
export interface IWebhookFunctions extends FunctionsBaseWithRequiredKeys<'getMode'> {
    getBodyData(): IDataObject;
    getHeaderData(): IncomingHttpHeaders;
    getInputConnectionData(
        connectionType: AINodeConnectionType,
        itemIndex: number,
        inputIndex?: number,
    ): Promise<unknown>;
    getNodeParameter(
        parameterName: string,
        fallbackValue?: any,
        options?: IGetNodeParameterOptions,
    ): NodeParameterValueType | object;
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

### Request Data Methods

| Method | Returns | Content |
|--------|---------|---------|
| `getBodyData()` | `IDataObject` | Parsed JSON/form body |
| `getHeaderData()` | `IncomingHttpHeaders` | All HTTP headers (lowercase keys) |
| `getQueryData()` | `object` | URL query string parameters (`?key=value`) |
| `getParamsData()` | `object` | URL path parameters (`:id` segments) |
| `getRequestObject()` | `express.Request` | Full Express request — use for raw body, files, etc. |
| `getResponseObject()` | `express.Response` | Full Express response — use for custom response handling |

### IWebhookResponseData

```typescript
export interface IWebhookResponseData {
    workflowData?: INodeExecutionData[][];   // Data to pass into the workflow
    webhookResponse?: any;                    // Custom response to send to caller
    noWebhookResponse?: boolean;             // true if you already sent response manually
}
```

**Rules**:
- ALWAYS set `workflowData` to pass data into the workflow
- Set `webhookResponse` to customize what the HTTP caller receives
- Set `noWebhookResponse: true` ONLY when you called `res.send()` / `res.json()` yourself

---

## webhookMethods (Webhook Lifecycle)

For nodes that register webhooks on EXTERNAL services. Defined on the `INodeType` object.

```typescript
webhookMethods: {
    default: {  // Matches the webhook name in webhooks[] array
        async checkExists(this: IHookFunctions): Promise<boolean> {
            // Check if webhook is already registered on external service
            // Return true → skip create; Return false → call create
        },

        async create(this: IHookFunctions): Promise<boolean> {
            // Register n8n's webhook URL on the external service
            // ALWAYS store the webhook ID for later deletion
            const webhookUrl = this.getNodeWebhookUrl('default');
            const staticData = this.getWorkflowStaticData('node');
            // ... API call ...
            staticData.webhookId = response.id;
            return true;  // true = success
        },

        async delete(this: IHookFunctions): Promise<boolean> {
            // Remove webhook from external service
            // Called when workflow is deactivated
            const staticData = this.getWorkflowStaticData('node');
            const webhookId = staticData.webhookId as string;
            // ... API call to delete ...
            delete staticData.webhookId;
            return true;
        },
    },
},
```

### Lifecycle Order

1. Workflow **activated** → `checkExists()` called
2. If `checkExists()` returns `false` → `create()` called
3. Webhook receives HTTP request → `webhook()` method called
4. Workflow **deactivated** → `delete()` called

### IHookFunctions Key Methods

| Method | Purpose |
|--------|---------|
| `getNodeWebhookUrl('default')` | Get the n8n URL to register on external service |
| `getWorkflowStaticData('node')` | Persistent storage for webhook IDs |
| `getNodeParameter(name)` | Read node parameter values |
| `getCredentials(name)` | Get credential data for API calls |

---

## WebhookResponseMode

Controls WHEN n8n sends the HTTP response back to the webhook caller.

```typescript
export type WebhookResponseMode =
    | 'onReceived'     // Respond immediately after webhook node
    | 'lastNode'       // Respond after last node in workflow
    | 'responseNode'   // Respond from a "Respond to Webhook" node
    | 'formPage'       // Form-based response
    | 'hostedChat'     // Chat-based response
    | 'streaming';     // Streaming response
```

### Mode Comparison

| Mode | Response Timing | Response Content | Best For |
|------|----------------|-----------------|----------|
| `onReceived` | Immediate (after webhook node) | Webhook node's `webhookResponse` or default | Fire-and-forget, fast acknowledgment |
| `lastNode` | After entire workflow completes | Last node's output data | Synchronous API endpoints |
| `responseNode` | When "Respond to Webhook" node executes | Custom body/headers/status | Full control over response |

### WebhookResponseData (What to Return)

```typescript
export type WebhookResponseData =
    | 'allEntries'         // Return all output items
    | 'firstEntryJson'     // Return first item's JSON only
    | 'firstEntryBinary'   // Return first item's binary data
    | 'noData';            // Return empty response
```

---

## IWebhookDescription

Defines webhook registration in the node description's `webhooks` array.

```typescript
export interface IWebhookDescription {
    name: WebhookType;                    // 'default' or 'setup'
    httpMethod: IHttpRequestMethods | string;  // 'GET', 'POST', or expression
    path: string;                         // URL path or expression
    isFullPath?: boolean;                 // true = use path as-is, false = prefix with node path
    responseMode?: WebhookResponseMode | string;
    responseData?: WebhookResponseData | string;
    responseContentType?: string;
    responseBinaryPropertyName?: string;
    responsePropertyName?: string;
    restartWebhook?: boolean;             // Restart after each execution
    nodeType?: 'webhook' | 'form' | 'mcp';
}
```

### WebhookType

| Type | Purpose |
|------|---------|
| `'default'` | Main webhook that triggers workflow execution |
| `'setup'` | Setup/verification webhook (e.g., for Slack URL verification challenges) |
