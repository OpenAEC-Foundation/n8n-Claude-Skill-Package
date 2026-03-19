# Methods Reference -- n8n-syntax-node-types

## INodeType Interface

The core interface every n8n node MUST implement.

```typescript
export interface INodeType {
    description: INodeTypeDescription;

    // Execution methods (implement ONE based on node type)
    execute?(this: IExecuteFunctions): Promise<INodeExecutionData[][]>;
    trigger?(this: ITriggerFunctions): Promise<ITriggerResponse | undefined>;
    poll?(this: IPollFunctions): Promise<INodeExecutionData[][] | null>;
    webhook?(this: IWebhookFunctions): Promise<IWebhookResponseData>;
    supplyData?(this: ISupplyDataFunctions, itemIndex: number): Promise<SupplyData>;

    // Chat message handler
    onMessage?(context: IExecuteFunctions, data: INodeExecutionData): Promise<NodeOutput>;

    // Dynamic methods
    methods?: {
        loadOptions?: { [key: string]: (this: ILoadOptionsFunctions) => Promise<INodePropertyOptions[]> };
        listSearch?: { [key: string]: (this: ILoadOptionsFunctions, filter?: string, paginationToken?: string) => Promise<INodeListSearchResult> };
        credentialTest?: { [name: string]: ICredentialTestFunction };
        resourceMapping?: { [name: string]: (this: ILoadOptionsFunctions) => Promise<ResourceMapperFields> };
        localResourceMapping?: { [name: string]: (this: ILocalLoadOptionsFunctions) => Promise<ResourceMapperFields> };
        actionHandler?: { [name: string]: (this: ILoadOptionsFunctions, payload: IDataObject | string | undefined) => Promise<NodeParameterValueType> };
    };

    // External webhook lifecycle
    webhookMethods?: {
        [name in WebhookType]?: {
            checkExists(this: IHookFunctions): Promise<boolean>;
            create(this: IHookFunctions): Promise<boolean>;
            delete(this: IHookFunctions): Promise<boolean>;
        };
    };

    // Declarative custom operations override
    customOperations?: {
        [resource: string]: {
            [operation: string]: (this: IExecuteFunctions) => Promise<NodeOutput>;
        };
    };
}
```

---

## INodeTypeDescription

Complete description object that defines node metadata, UI, and behavior.

```typescript
export interface INodeTypeDescription extends INodeTypeBaseDescription {
    version: number | number[];
    defaults: { name: string; color?: string };
    inputs: Array<NodeConnectionType | INodeInputConfiguration> | ExpressionString;
    outputs: Array<NodeConnectionType | INodeOutputConfiguration> | ExpressionString;
    properties: INodeProperties[];
    credentials?: INodeCredentialDescription[];
    // Trigger-specific
    eventTriggerDescription?: string;
    activationMessage?: string;
    polling?: true;
    // Webhook-specific
    webhooks?: IWebhookDescription[];
    supportsCORS?: true;
    // Declarative-specific
    requestDefaults?: DeclarativeRestApiSettings.HttpRequestOptions;
    requestOperations?: IN8nRequestOperations;
    // UI
    subtitle?: string;
    hints?: NodeHint[];
    triggerPanel?: TriggerPanelDefinition | boolean;
    parameterPane?: 'wide';
    // Feature flags
    usableAsTool?: true | UsableAsToolDescription;
    maxNodes?: number;
    hidden?: true;
    sensitiveOutputFields?: string[];
}

export interface INodeTypeBaseDescription {
    displayName: string;
    name: string;
    icon?: string | { light: string; dark: string };
    iconColor?: ThemeIconColor;
    group: Array<'input' | 'output' | 'transform' | 'trigger' | 'schedule'>;
    description: string;
    documentationUrl?: string;
    defaultVersion?: number;
    codex?: CodexData;
}
```

### INodeCredentialDescription

```typescript
export interface INodeCredentialDescription {
    name: string;
    required?: boolean;
    displayName?: string;
    displayOptions?: ICredentialsDisplayOptions;
    testedBy?: ICredentialTestRequest | string;
}
```

### INodeInputConfiguration / INodeOutputConfiguration

```typescript
export interface INodeInputConfiguration {
    type: NodeConnectionType;
    displayName?: string;
    required?: boolean;
    maxConnections?: number;
    filter?: INodeFilter;
    category?: string;
}

export interface INodeOutputConfiguration {
    type: NodeConnectionType;
    displayName?: string;
    required?: boolean;
    maxConnections?: number;
    category?: 'error';
}
```

### NodeConnectionType Values

| Constant | Value | Purpose |
|----------|-------|---------|
| `NodeConnectionTypes.Main` | `'main'` | Standard data flow |
| `NodeConnectionTypes.AiAgent` | `'ai_agent'` | AI agent connection |
| `NodeConnectionTypes.AiLanguageModel` | `'ai_languageModel'` | LLM connection |
| `NodeConnectionTypes.AiMemory` | `'ai_memory'` | Memory connection |
| `NodeConnectionTypes.AiTool` | `'ai_tool'` | Tool connection |
| `NodeConnectionTypes.AiVectorStore` | `'ai_vectorStore'` | Vector store |
| `NodeConnectionTypes.AiEmbedding` | `'ai_embedding'` | Embedding model |
| `NodeConnectionTypes.AiDocument` | `'ai_document'` | Document loader |
| `NodeConnectionTypes.AiRetriever` | `'ai_retriever'` | Retriever |
| `NodeConnectionTypes.AiOutputParser` | `'ai_outputParser'` | Output parser |
| `NodeConnectionTypes.AiTextSplitter` | `'ai_textSplitter'` | Text splitter |

---

## INodeProperties (All 22 Property Types)

### Core Interface

```typescript
export interface INodeProperties {
    displayName: string;
    name: string;
    type: NodePropertyTypes;
    default: NodeParameterValueType;
    description?: string;
    hint?: string;
    placeholder?: string;
    required?: boolean;
    noDataExpression?: boolean;
    typeOptions?: INodePropertyTypeOptions;
    displayOptions?: IDisplayOptions;
    options?: Array<INodePropertyOptions | INodeProperties | INodePropertyCollection>;
    routing?: INodePropertyRouting;
    extractValue?: INodePropertyValueExtractor;
    modes?: INodePropertyMode[];
    validateType?: FieldType;
    requiresDataPath?: 'single' | 'multiple';
}
```

### Property Type Details

#### 1. `'boolean'`
```typescript
{ displayName: 'Active', name: 'active', type: 'boolean', default: false }
```

#### 2. `'string'`
```typescript
// Plain text
{ displayName: 'Name', name: 'name', type: 'string', default: '' }

// Textarea
{ displayName: 'Body', name: 'body', type: 'string', default: '',
  typeOptions: { rows: 5 } }

// Password
{ displayName: 'Token', name: 'token', type: 'string', default: '',
  typeOptions: { password: true } }

// Code editor
{ displayName: 'Code', name: 'code', type: 'string', default: '',
  typeOptions: { editor: 'jsEditor' } }
```

#### 3. `'number'`
```typescript
{ displayName: 'Limit', name: 'limit', type: 'number', default: 50,
  typeOptions: { minValue: 1, maxValue: 1000, numberPrecision: 0 } }
```

#### 4. `'options'` (single-select dropdown)
```typescript
{
    displayName: 'Resource',
    name: 'resource',
    type: 'options',
    noDataExpression: true,
    options: [
        { name: 'Contact', value: 'contact', description: 'Manage contacts' },
        { name: 'Deal', value: 'deal', description: 'Manage deals' },
    ],
    default: 'contact',
}
```

#### 5. `'multiOptions'` (multi-select dropdown)
```typescript
{
    displayName: 'Fields',
    name: 'fields',
    type: 'multiOptions',
    options: [
        { name: 'Name', value: 'name' },
        { name: 'Email', value: 'email' },
    ],
    default: [],
}
```

#### 6. `'collection'` (optional field group)
```typescript
{
    displayName: 'Additional Fields',
    name: 'additionalFields',
    type: 'collection',
    placeholder: 'Add Field',
    default: {},
    options: [
        { displayName: 'Title', name: 'title', type: 'string', default: '' },
        { displayName: 'Priority', name: 'priority', type: 'number', default: 0 },
    ],
}
```

#### 7. `'fixedCollection'` (structured field groups)
```typescript
{
    displayName: 'Fields',
    name: 'fields',
    type: 'fixedCollection',
    default: {},
    typeOptions: { multipleValues: true },
    options: [
        {
            displayName: 'Field',
            name: 'field',
            values: [
                { displayName: 'Key', name: 'key', type: 'string', default: '' },
                { displayName: 'Value', name: 'value', type: 'string', default: '' },
            ],
        },
    ],
}
```

#### 8. `'json'`
```typescript
{ displayName: 'JSON Data', name: 'jsonData', type: 'json', default: '{}',
  typeOptions: { alwaysOpenEditWindow: true } }
```

#### 9. `'dateTime'`
```typescript
{ displayName: 'Start Date', name: 'startDate', type: 'dateTime', default: '' }
```

#### 10. `'color'`
```typescript
{ displayName: 'Color', name: 'color', type: 'color', default: '#ff0000' }
```

#### 11. `'hidden'`
```typescript
{ displayName: 'Version', name: 'version', type: 'hidden', default: 'v2' }
```

#### 12. `'notice'`
```typescript
{ displayName: 'This node requires API access.', name: 'notice', type: 'notice', default: '' }
```

#### 13. `'callout'`
```typescript
{ displayName: 'Warning: this action is irreversible.', name: 'warning',
  type: 'callout', default: '' }
```

#### 14. `'button'`
```typescript
{ displayName: 'Generate', name: 'generate', type: 'button', default: '',
  typeOptions: { buttonConfig: { label: 'Generate Key', action: 'generateKey' } } }
```

#### 15. `'icon'`
```typescript
{ displayName: 'Icon', name: 'icon', type: 'icon', default: 'file' }
```

#### 16. `'credentialsSelect'`
```typescript
{ displayName: 'Credential Type', name: 'credentialType', type: 'credentialsSelect', default: '',
  credentialTypes: ['has:authenticate'] }
```

#### 17. `'resourceLocator'`
```typescript
{
    displayName: 'Document',
    name: 'document',
    type: 'resourceLocator',
    default: { mode: 'list', value: '' },
    modes: [
        { displayName: 'From List', name: 'list', type: 'list',
          typeOptions: { searchListMethod: 'searchDocuments' } },
        { displayName: 'By URL', name: 'url', type: 'string',
          extractValue: { type: 'regex', regex: '/documents/d/([a-zA-Z0-9-_]+)' } },
        { displayName: 'By ID', name: 'id', type: 'string',
          placeholder: 'e.g., 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms' },
    ],
}
```

#### 18. `'curlImport'`
```typescript
{ displayName: 'Import cURL', name: 'curlImport', type: 'curlImport', default: '' }
```

#### 19. `'resourceMapper'`
```typescript
{
    displayName: 'Columns',
    name: 'columns',
    type: 'resourceMapper',
    default: { mappingMode: 'defineBelow', value: null },
    typeOptions: {
        resourceMapper: {
            resourceMapperMethod: 'getColumns',
            mode: 'add',
            addAllFields: true,
        },
    },
}
```

#### 20. `'filter'`
```typescript
{
    displayName: 'Conditions',
    name: 'conditions',
    type: 'filter',
    default: {},
    typeOptions: {
        filter: {
            caseSensitive: '={{!$parameter.options.ignoreCase}}',
            typeValidation: 'strict',
        },
    },
}
```

#### 21. `'assignmentCollection'`
```typescript
{
    displayName: 'Fields to Set',
    name: 'assignments',
    type: 'assignmentCollection',
    default: {},
}
```

#### 22. `'workflowSelector'`
```typescript
{ displayName: 'Workflow', name: 'workflowId', type: 'workflowSelector', default: '' }
```

---

## IDisplayOptions

Controls when a property is visible in the UI.

```typescript
export interface IDisplayOptions {
    show?: {
        [parameterName: string]: Array<NodeParameterValue | DisplayCondition> | undefined;
        '@version'?: Array<number | DisplayCondition>;
        '@feature'?: Array<string | DisplayCondition>;
        '@tool'?: boolean[];
    };
    hide?: {
        [parameterName: string]: Array<NodeParameterValue | DisplayCondition> | undefined;
    };
    hideOnCloud?: boolean;
}
```

### DisplayCondition Operators

| Operator | Syntax | Example |
|----------|--------|---------|
| Equals | `{ _cnd: { eq: value } }` | `{ _cnd: { eq: 'active' } }` |
| Not equals | `{ _cnd: { not: value } }` | `{ _cnd: { not: 'archived' } }` |
| Greater than | `{ _cnd: { gt: number } }` | `{ _cnd: { gt: 5 } }` |
| Greater or equal | `{ _cnd: { gte: number } }` | `{ _cnd: { gte: 2 } }` |
| Less than | `{ _cnd: { lt: number } }` | `{ _cnd: { lt: 100 } }` |
| Less or equal | `{ _cnd: { lte: number } }` | `{ _cnd: { lte: 10 } }` |
| Between | `{ _cnd: { between: { from, to } } }` | `{ _cnd: { between: { from: 1, to: 10 } } }` |
| Starts with | `{ _cnd: { startsWith: string } }` | `{ _cnd: { startsWith: 'http' } }` |
| Ends with | `{ _cnd: { endsWith: string } }` | `{ _cnd: { endsWith: '.json' } }` |
| Includes | `{ _cnd: { includes: string } }` | `{ _cnd: { includes: 'admin' } }` |
| Regex | `{ _cnd: { regex: string } }` | `{ _cnd: { regex: '^v\\d+' } }` |
| Exists | `{ _cnd: { exists: true } }` | `{ _cnd: { exists: true } }` |

### Display Logic Rules

- `show` conditions use AND logic between keys: ALL conditions must be true
- Within a single key, values use OR logic: ANY value can match
- `hide` takes precedence over `show` when both match
- Simple values (non-`_cnd`) match by strict equality

---

## IExecuteFunctions (Complete Method Reference)

### Parameter Access

| Method | Return Type | Purpose |
|--------|-------------|---------|
| `getNodeParameter(name, itemIndex)` | `any` | Get parameter value for item |
| `getNodeParameter(name, itemIndex, fallback)` | `any` | With fallback default |
| `getInputData(inputIndex?)` | `INodeExecutionData[]` | Get all input items |
| `getInputConnectionData(type, itemIndex)` | `Promise<unknown>` | Get AI node input |

### Credentials

| Method | Return Type | Purpose |
|--------|-------------|---------|
| `getCredentials(type)` | `Promise<ICredentialDataDecryptedObject>` | Get decrypted credential data |

### Execution Control

| Method | Return Type | Purpose |
|--------|-------------|---------|
| `continueOnFail()` | `boolean` | Check if continue-on-fail is enabled |
| `executeWorkflow(info, data?)` | `Promise<ExecuteWorkflowData>` | Execute sub-workflow |
| `putExecutionToWait(date)` | `Promise<void>` | Pause until date |
| `sendMessageToUI(msg)` | `void` | Send message to editor |
| `sendResponse(data)` | `void` | Send webhook response |

### Helper Functions

| Helper | Purpose |
|--------|---------|
| `this.helpers.httpRequest(options)` | Make HTTP requests |
| `this.helpers.requestWithAuthentication(credType, options)` | Authenticated HTTP request |
| `this.helpers.getBinaryDataBuffer(itemIndex, property)` | Read binary data to Buffer |
| `this.helpers.prepareBinaryData(buffer, fileName?, mimeType?)` | Create IBinaryData |
| `this.helpers.assertBinaryData(itemIndex, property)` | Assert binary data exists |
| `this.helpers.normalizeItems(items)` | Normalize item format |
| `this.helpers.constructExecutionMetaData(items, options)` | Add pairedItem metadata |
| `this.helpers.copyInputItems(items, properties)` | Copy specific properties |

---

## IVersionedNodeType

For nodes that support multiple versions with different behavior.

```typescript
export interface IVersionedNodeType {
    nodeVersions: { [key: number]: INodeType };
    currentVersion: number;
    description: INodeTypeBaseDescription;
    getNodeType(version?: number): INodeType;
}
```

---

## INodePropertyRouting (Declarative Nodes)

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

## Node Return Type

```typescript
export type NodeOutput =
    | INodeExecutionData[][]
    | NodeExecutionWithMetadata[][]
    | EngineRequest
    | null;
```

**ALWAYS** return `INodeExecutionData[][]` from `execute()`. The `null` and `EngineRequest` variants are for internal/advanced use only.
