# Examples Reference -- n8n-syntax-node-types

## Example 1: Complete Programmatic Node

A full node with resource/operation pattern, credentials, error handling, and binary data.

```typescript
import type {
    IExecuteFunctions,
    INodeExecutionData,
    INodeType,
    INodeTypeDescription,
    IDataObject,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class MyService implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'My Service',
        name: 'myService',
        icon: 'file:myService.svg',
        group: ['transform'],
        version: 1,
        subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
        description: 'Interact with My Service API',
        defaults: { name: 'My Service' },
        inputs: [NodeConnectionTypes.Main],
        outputs: [NodeConnectionTypes.Main],
        credentials: [
            {
                name: 'myServiceApi',
                required: true,
            },
        ],
        properties: [
            // Resource selector
            {
                displayName: 'Resource',
                name: 'resource',
                type: 'options',
                noDataExpression: true,
                options: [
                    { name: 'User', value: 'user' },
                    { name: 'Document', value: 'document' },
                ],
                default: 'user',
            },
            // Operation selector (conditional on resource)
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: {
                    show: { resource: ['user'] },
                },
                options: [
                    { name: 'Create', value: 'create', action: 'Create a user' },
                    { name: 'Get', value: 'get', action: 'Get a user' },
                    { name: 'Get Many', value: 'getAll', action: 'Get many users' },
                    { name: 'Delete', value: 'delete', action: 'Delete a user' },
                ],
                default: 'get',
            },
            // Conditional parameters
            {
                displayName: 'User ID',
                name: 'userId',
                type: 'string',
                required: true,
                default: '',
                displayOptions: {
                    show: { resource: ['user'], operation: ['get', 'delete'] },
                },
                description: 'The ID of the user',
            },
            {
                displayName: 'Email',
                name: 'email',
                type: 'string',
                required: true,
                default: '',
                placeholder: 'user@example.com',
                displayOptions: {
                    show: { resource: ['user'], operation: ['create'] },
                },
            },
            {
                displayName: 'Return All',
                name: 'returnAll',
                type: 'boolean',
                default: false,
                displayOptions: {
                    show: { resource: ['user'], operation: ['getAll'] },
                },
            },
            {
                displayName: 'Limit',
                name: 'limit',
                type: 'number',
                default: 50,
                typeOptions: { minValue: 1, maxValue: 100 },
                displayOptions: {
                    show: {
                        resource: ['user'],
                        operation: ['getAll'],
                        returnAll: [false],
                    },
                },
            },
            // Additional fields (collection)
            {
                displayName: 'Additional Fields',
                name: 'additionalFields',
                type: 'collection',
                placeholder: 'Add Field',
                default: {},
                displayOptions: {
                    show: { resource: ['user'], operation: ['create'] },
                },
                options: [
                    {
                        displayName: 'Name',
                        name: 'name',
                        type: 'string',
                        default: '',
                    },
                    {
                        displayName: 'Role',
                        name: 'role',
                        type: 'options',
                        options: [
                            { name: 'Admin', value: 'admin' },
                            { name: 'User', value: 'user' },
                            { name: 'Viewer', value: 'viewer' },
                        ],
                        default: 'user',
                    },
                ],
            },
        ],
    };

    async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
        const items = this.getInputData();
        const returnData: INodeExecutionData[] = [];
        const resource = this.getNodeParameter('resource', 0) as string;
        const operation = this.getNodeParameter('operation', 0) as string;
        const credentials = await this.getCredentials('myServiceApi');

        for (let i = 0; i < items.length; i++) {
            try {
                if (resource === 'user') {
                    if (operation === 'get') {
                        const userId = this.getNodeParameter('userId', i) as string;
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `${credentials.baseUrl}/users/${userId}`,
                            headers: {
                                Authorization: `Bearer ${credentials.apiKey}`,
                            },
                        });
                        returnData.push({ json: response as IDataObject });
                    }

                    if (operation === 'create') {
                        const email = this.getNodeParameter('email', i) as string;
                        const additionalFields = this.getNodeParameter(
                            'additionalFields', i,
                        ) as IDataObject;
                        const body: IDataObject = { email, ...additionalFields };
                        const response = await this.helpers.httpRequest({
                            method: 'POST',
                            url: `${credentials.baseUrl}/users`,
                            headers: {
                                Authorization: `Bearer ${credentials.apiKey}`,
                            },
                            body,
                        });
                        returnData.push({ json: response as IDataObject });
                    }

                    if (operation === 'getAll') {
                        const returnAll = this.getNodeParameter('returnAll', i) as boolean;
                        const limit = returnAll
                            ? 0
                            : (this.getNodeParameter('limit', i) as number);
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `${credentials.baseUrl}/users`,
                            qs: returnAll ? {} : { limit },
                            headers: {
                                Authorization: `Bearer ${credentials.apiKey}`,
                            },
                        });
                        const users = response as IDataObject[];
                        for (const user of users) {
                            returnData.push({ json: user });
                        }
                    }

                    if (operation === 'delete') {
                        const userId = this.getNodeParameter('userId', i) as string;
                        await this.helpers.httpRequest({
                            method: 'DELETE',
                            url: `${credentials.baseUrl}/users/${userId}`,
                            headers: {
                                Authorization: `Bearer ${credentials.apiKey}`,
                            },
                        });
                        returnData.push({ json: { deleted: true, userId } });
                    }
                }
            } catch (error) {
                if (this.continueOnFail()) {
                    returnData.push({
                        json: { error: (error as Error).message },
                        pairedItem: { item: i },
                    });
                    continue;
                }
                throw error;
            }
        }

        return [returnData];
    }
}
```

---

## Example 2: Property Definitions with All Common Types

```typescript
const properties: INodeProperties[] = [
    // Boolean toggle
    {
        displayName: 'Return All',
        name: 'returnAll',
        type: 'boolean',
        default: false,
        description: 'Whether to return all results or limit',
    },

    // String with expression support
    {
        displayName: 'Name',
        name: 'name',
        type: 'string',
        default: '',
        required: true,
        description: 'The name of the resource',
    },

    // Number with constraints
    {
        displayName: 'Timeout (ms)',
        name: 'timeout',
        type: 'number',
        default: 5000,
        typeOptions: {
            minValue: 100,
            maxValue: 60000,
            numberPrecision: 0,
        },
    },

    // Options dropdown
    {
        displayName: 'Method',
        name: 'method',
        type: 'options',
        options: [
            { name: 'GET', value: 'GET' },
            { name: 'POST', value: 'POST' },
            { name: 'PUT', value: 'PUT' },
            { name: 'DELETE', value: 'DELETE' },
        ],
        default: 'GET',
    },

    // Multi-select
    {
        displayName: 'Tags',
        name: 'tags',
        type: 'multiOptions',
        options: [
            { name: 'Important', value: 'important' },
            { name: 'Urgent', value: 'urgent' },
            { name: 'Low Priority', value: 'low' },
        ],
        default: [],
    },

    // Collection (optional fields)
    {
        displayName: 'Options',
        name: 'options',
        type: 'collection',
        placeholder: 'Add Option',
        default: {},
        options: [
            {
                displayName: 'Encoding',
                name: 'encoding',
                type: 'string',
                default: 'utf-8',
            },
            {
                displayName: 'Retry',
                name: 'retry',
                type: 'boolean',
                default: false,
            },
        ],
    },

    // Fixed collection (key-value pairs)
    {
        displayName: 'Headers',
        name: 'headers',
        type: 'fixedCollection',
        default: {},
        typeOptions: { multipleValues: true },
        options: [
            {
                displayName: 'Header',
                name: 'header',
                values: [
                    {
                        displayName: 'Name',
                        name: 'name',
                        type: 'string',
                        default: '',
                    },
                    {
                        displayName: 'Value',
                        name: 'value',
                        type: 'string',
                        default: '',
                    },
                ],
            },
        ],
    },

    // JSON editor
    {
        displayName: 'JSON Body',
        name: 'jsonBody',
        type: 'json',
        default: '{}',
        typeOptions: { alwaysOpenEditWindow: true },
    },

    // Date/time picker
    {
        displayName: 'Due Date',
        name: 'dueDate',
        type: 'dateTime',
        default: '',
    },

    // Resource locator
    {
        displayName: 'Spreadsheet',
        name: 'spreadsheet',
        type: 'resourceLocator',
        default: { mode: 'list', value: '' },
        modes: [
            {
                displayName: 'From List',
                name: 'list',
                type: 'list',
                typeOptions: {
                    searchListMethod: 'searchSpreadsheets',
                    searchable: true,
                },
            },
            {
                displayName: 'By URL',
                name: 'url',
                type: 'string',
                placeholder: 'https://docs.google.com/spreadsheets/d/...',
                extractValue: {
                    type: 'regex',
                    regex: '/spreadsheets/d/([a-zA-Z0-9-_]+)',
                },
            },
            {
                displayName: 'By ID',
                name: 'id',
                type: 'string',
                placeholder: 'e.g., 1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms',
            },
        ],
    },

    // Code editor (string with editor typeOption)
    {
        displayName: 'JavaScript Code',
        name: 'jsCode',
        type: 'string',
        default: 'return items;',
        typeOptions: {
            editor: 'jsEditor',
            rows: 10,
        },
    },

    // Notice (display only)
    {
        displayName: 'This operation will permanently delete the resource.',
        name: 'deleteNotice',
        type: 'notice',
        default: '',
    },

    // Filter conditions
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
    },

    // Assignment collection (Set node style)
    {
        displayName: 'Fields to Set',
        name: 'assignments',
        type: 'assignmentCollection',
        default: {},
    },
];
```

---

## Example 3: displayOptions Patterns

### Basic Show/Hide

```typescript
// Show when resource is 'user'
{
    displayOptions: {
        show: { resource: ['user'] },
    },
}

// Show when resource is 'user' AND operation is 'get'
{
    displayOptions: {
        show: {
            resource: ['user'],
            operation: ['get'],
        },
    },
}

// Show when operation is 'get' OR 'update'
{
    displayOptions: {
        show: {
            operation: ['get', 'update'],  // OR within same key
        },
    },
}

// Hide when returnAll is true
{
    displayOptions: {
        hide: { returnAll: [true] },
    },
}
```

### Rich Conditions with Operators

```typescript
// Show for node version 2 and above
{
    displayOptions: {
        show: {
            '@version': [{ _cnd: { gte: 2 } }],
        },
    },
}

// Show when status is NOT 'archived'
{
    displayOptions: {
        show: {
            status: [{ _cnd: { not: 'archived' } }],
        },
    },
}

// Show when count is between 1 and 100
{
    displayOptions: {
        show: {
            count: [{ _cnd: { between: { from: 1, to: 100 } } }],
        },
    },
}

// Show when URL starts with 'https'
{
    displayOptions: {
        show: {
            url: [{ _cnd: { startsWith: 'https' } }],
        },
    },
}

// Show when value exists (is not empty/null)
{
    displayOptions: {
        show: {
            apiKey: [{ _cnd: { exists: true } }],
        },
    },
}

// Show when used as an AI tool
{
    displayOptions: {
        show: {
            '@tool': [true],
        },
    },
}
```

---

## Example 4: Complete Declarative Node

```typescript
import type { INodeType, INodeTypeDescription } from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class TodoApi implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Todo API',
        name: 'todoApi',
        icon: 'file:todo.svg',
        group: ['input'],
        version: 1,
        subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
        description: 'Interact with Todo API',
        defaults: { name: 'Todo API' },
        inputs: [NodeConnectionTypes.Main],
        outputs: [NodeConnectionTypes.Main],
        usableAsTool: true,
        credentials: [
            {
                name: 'todoApi',
                required: true,
            },
        ],
        requestDefaults: {
            baseURL: 'https://api.todo.example.com/v1',
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
                    { name: 'Task', value: 'task' },
                    { name: 'List', value: 'list' },
                ],
                default: 'task',
            },
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: { show: { resource: ['task'] } },
                options: [
                    {
                        name: 'Create',
                        value: 'create',
                        action: 'Create a task',
                        routing: {
                            request: {
                                method: 'POST',
                                url: '/tasks',
                            },
                        },
                    },
                    {
                        name: 'Get Many',
                        value: 'getAll',
                        action: 'Get many tasks',
                        routing: {
                            request: {
                                method: 'GET',
                                url: '/tasks',
                            },
                            output: {
                                maxResults: '={{$parameter.limit}}',
                            },
                        },
                    },
                    {
                        name: 'Update',
                        value: 'update',
                        action: 'Update a task',
                        routing: {
                            request: {
                                method: 'PATCH',
                                url: '=/tasks/{{$parameter.taskId}}',
                            },
                        },
                    },
                    {
                        name: 'Delete',
                        value: 'delete',
                        action: 'Delete a task',
                        routing: {
                            request: {
                                method: 'DELETE',
                                url: '=/tasks/{{$parameter.taskId}}',
                            },
                        },
                    },
                ],
                default: 'getAll',
            },
            // Task ID (for get, update, delete)
            {
                displayName: 'Task ID',
                name: 'taskId',
                type: 'string',
                required: true,
                default: '',
                displayOptions: {
                    show: {
                        resource: ['task'],
                        operation: ['update', 'delete'],
                    },
                },
            },
            // Title (for create) -- sent in request body via routing
            {
                displayName: 'Title',
                name: 'title',
                type: 'string',
                required: true,
                default: '',
                displayOptions: {
                    show: { resource: ['task'], operation: ['create'] },
                },
                routing: {
                    send: {
                        type: 'body',
                        property: 'title',
                    },
                },
            },
            // Limit (for getAll) -- sent as query parameter
            {
                displayName: 'Limit',
                name: 'limit',
                type: 'number',
                default: 50,
                typeOptions: { minValue: 1, maxValue: 100 },
                displayOptions: {
                    show: { resource: ['task'], operation: ['getAll'] },
                },
                routing: {
                    send: {
                        type: 'query',
                        property: 'limit',
                    },
                },
            },
            // Optional body fields for update
            {
                displayName: 'Update Fields',
                name: 'updateFields',
                type: 'collection',
                placeholder: 'Add Field',
                default: {},
                displayOptions: {
                    show: { resource: ['task'], operation: ['update'] },
                },
                options: [
                    {
                        displayName: 'Title',
                        name: 'title',
                        type: 'string',
                        default: '',
                        routing: {
                            send: { type: 'body', property: 'title' },
                        },
                    },
                    {
                        displayName: 'Completed',
                        name: 'completed',
                        type: 'boolean',
                        default: false,
                        routing: {
                            send: { type: 'body', property: 'completed' },
                        },
                    },
                    {
                        displayName: 'Priority',
                        name: 'priority',
                        type: 'options',
                        options: [
                            { name: 'Low', value: 'low' },
                            { name: 'Medium', value: 'medium' },
                            { name: 'High', value: 'high' },
                        ],
                        default: 'medium',
                        routing: {
                            send: { type: 'body', property: 'priority' },
                        },
                    },
                ],
            },
        ],
    };

    // NO execute() method -- n8n handles all HTTP requests via routing config
}
```

---

## Example 5: Trigger Node with emit()

```typescript
import type {
    ITriggerFunctions,
    INodeType,
    INodeTypeDescription,
    ITriggerResponse,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class MyTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'My Trigger',
        name: 'myTrigger',
        icon: 'fa:bolt',
        group: ['trigger'],
        version: 1,
        description: 'Starts workflow on events',
        defaults: { name: 'My Trigger' },
        inputs: [],  // Trigger nodes have NO inputs
        outputs: [NodeConnectionTypes.Main],
        properties: [
            {
                displayName: 'Event Type',
                name: 'eventType',
                type: 'options',
                options: [
                    { name: 'Created', value: 'created' },
                    { name: 'Updated', value: 'updated' },
                ],
                default: 'created',
            },
        ],
    };

    async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
        const eventType = this.getNodeParameter('eventType') as string;

        const executeTrigger = (data: object) => {
            this.emit([this.helpers.returnJsonArray([data])]);
        };

        if (this.getMode() === 'manual') {
            // Manual execution: emit test data immediately
            const manualTriggerFunction = async () => {
                executeTrigger({ event: eventType, test: true });
            };
            return { manualTriggerFunction };
        }

        // Production: set up listener
        const intervalId = setInterval(() => {
            executeTrigger({ event: eventType, timestamp: new Date().toISOString() });
        }, 60000);

        // Return cleanup function
        return {
            closeFunction: async () => {
                clearInterval(intervalId);
            },
        };
    }
}
```

---

## Example 6: Versioned Node

```typescript
import type { INodeTypeBaseDescription, IVersionedNodeType, INodeType } from 'n8n-workflow';

class MyNodeV1 implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'My Node',
        name: 'myNode',
        group: ['transform'],
        version: 1,
        // ... V1 properties
    };
    async execute(this: IExecuteFunctions) { /* V1 logic */ }
}

class MyNodeV2 implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'My Node',
        name: 'myNode',
        group: ['transform'],
        version: 2,
        // ... V2 properties (may differ from V1)
    };
    async execute(this: IExecuteFunctions) { /* V2 logic */ }
}

export class MyNode implements IVersionedNodeType {
    currentVersion = 2;

    description: INodeTypeBaseDescription = {
        displayName: 'My Node',
        name: 'myNode',
        group: ['transform'],
        description: 'My node description',
        defaultVersion: 2,
    };

    nodeVersions: { [key: number]: INodeType } = {
        1: new MyNodeV1(),
        2: new MyNodeV2(),
    };

    getNodeType(version?: number): INodeType {
        return this.nodeVersions[version ?? this.currentVersion];
    }
}
```

---

## Example 7: Node with Dynamic Options Loading

```typescript
export class DynamicNode implements INodeType {
    description: INodeTypeDescription = {
        // ...
        properties: [
            {
                displayName: 'Project',
                name: 'project',
                type: 'options',
                typeOptions: {
                    loadOptionsMethod: 'getProjects',
                },
                default: '',
                description: 'Select a project',
            },
            {
                displayName: 'Board',
                name: 'board',
                type: 'options',
                typeOptions: {
                    loadOptionsMethod: 'getBoards',
                    loadOptionsDependsOn: ['project'],  // Reload when project changes
                },
                default: '',
            },
        ],
    };

    methods = {
        loadOptions: {
            async getProjects(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
                const credentials = await this.getCredentials('myApi');
                const projects = await this.helpers.httpRequest({
                    method: 'GET',
                    url: `${credentials.baseUrl}/projects`,
                    headers: { Authorization: `Bearer ${credentials.apiKey}` },
                });
                return (projects as any[]).map((p) => ({
                    name: p.name,
                    value: p.id,
                }));
            },
            async getBoards(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
                const projectId = this.getNodeParameter('project') as string;
                const credentials = await this.getCredentials('myApi');
                const boards = await this.helpers.httpRequest({
                    method: 'GET',
                    url: `${credentials.baseUrl}/projects/${projectId}/boards`,
                    headers: { Authorization: `Bearer ${credentials.apiKey}` },
                });
                return (boards as any[]).map((b) => ({
                    name: b.name,
                    value: b.id,
                }));
            },
        },
    };
}
```

---

## Example 8: Multi-Output Node

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const successItems: INodeExecutionData[] = [];
    const errorItems: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const status = this.getNodeParameter('status', i) as string;
        if (status === 'success') {
            successItems.push(items[i]);
        } else {
            errorItems.push(items[i]);
        }
    }

    // Output 0: success items, Output 1: error items
    return [successItems, errorItems];
}
```

Configure in description:
```typescript
outputs: [NodeConnectionTypes.Main, NodeConnectionTypes.Main],
outputNames: ['Success', 'Error'],
```
