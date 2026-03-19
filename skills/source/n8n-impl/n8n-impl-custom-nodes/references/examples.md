# Custom Node Development: Complete Examples

## Example 1: Complete Programmatic Node

A fully functional node with resource/operation pattern, error handling, and credential usage.

```typescript
import type {
    IDataObject,
    IExecuteFunctions,
    INodeExecutionData,
    INodeType,
    INodeTypeDescription,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class TaskManager implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Task Manager',
        name: 'taskManager',
        icon: 'file:taskManager.svg',
        group: ['transform'],
        version: 1,
        subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
        description: 'Interact with the Task Manager API',
        defaults: {
            name: 'Task Manager',
        },
        inputs: [NodeConnectionTypes.Main],
        outputs: [NodeConnectionTypes.Main],
        credentials: [
            {
                name: 'taskManagerApi',
                required: true,
            },
        ],
        properties: [
            // Resource
            {
                displayName: 'Resource',
                name: 'resource',
                type: 'options',
                noDataExpression: true,
                options: [
                    { name: 'Task', value: 'task' },
                    { name: 'Project', value: 'project' },
                ],
                default: 'task',
            },
            // Operations for Task
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: {
                    show: { resource: ['task'] },
                },
                options: [
                    { name: 'Create', value: 'create', action: 'Create a task' },
                    { name: 'Get', value: 'get', action: 'Get a task' },
                    { name: 'Get Many', value: 'getAll', action: 'Get many tasks' },
                    { name: 'Update', value: 'update', action: 'Update a task' },
                    { name: 'Delete', value: 'delete', action: 'Delete a task' },
                ],
                default: 'get',
            },
            // Task ID (for get, update, delete)
            {
                displayName: 'Task ID',
                name: 'taskId',
                type: 'string',
                required: true,
                default: '',
                displayOptions: {
                    show: { resource: ['task'], operation: ['get', 'update', 'delete'] },
                },
            },
            // Title (for create)
            {
                displayName: 'Title',
                name: 'title',
                type: 'string',
                required: true,
                default: '',
                displayOptions: {
                    show: { resource: ['task'], operation: ['create'] },
                },
            },
            // Additional Fields (for create and update)
            {
                displayName: 'Additional Fields',
                name: 'additionalFields',
                type: 'collection',
                placeholder: 'Add Field',
                default: {},
                displayOptions: {
                    show: { resource: ['task'], operation: ['create', 'update'] },
                },
                options: [
                    {
                        displayName: 'Description',
                        name: 'description',
                        type: 'string',
                        default: '',
                    },
                    {
                        displayName: 'Due Date',
                        name: 'dueDate',
                        type: 'dateTime',
                        default: '',
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
                    },
                ],
            },
            // Limit (for getAll)
            {
                displayName: 'Limit',
                name: 'limit',
                type: 'number',
                default: 50,
                typeOptions: { minValue: 1, maxValue: 100 },
                displayOptions: {
                    show: { resource: ['task'], operation: ['getAll'] },
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
        const credentials = await this.getCredentials('taskManagerApi');

        for (let i = 0; i < items.length; i++) {
            try {
                if (resource === 'task') {
                    if (operation === 'create') {
                        const title = this.getNodeParameter('title', i) as string;
                        const additionalFields = this.getNodeParameter('additionalFields', i) as IDataObject;
                        const body: IDataObject = { title, ...additionalFields };

                        const response = await this.helpers.httpRequest({
                            method: 'POST',
                            url: `${credentials.baseUrl}/tasks`,
                            body,
                            headers: { Authorization: `Bearer ${credentials.apiKey}` },
                            json: true,
                        });
                        returnData.push({ json: response as IDataObject });
                    }

                    if (operation === 'get') {
                        const taskId = this.getNodeParameter('taskId', i) as string;
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `${credentials.baseUrl}/tasks/${taskId}`,
                            headers: { Authorization: `Bearer ${credentials.apiKey}` },
                        });
                        returnData.push({ json: response as IDataObject });
                    }

                    if (operation === 'getAll') {
                        const limit = this.getNodeParameter('limit', i) as number;
                        const response = await this.helpers.httpRequest({
                            method: 'GET',
                            url: `${credentials.baseUrl}/tasks`,
                            qs: { limit },
                            headers: { Authorization: `Bearer ${credentials.apiKey}` },
                        });
                        const tasks = response as IDataObject[];
                        for (const task of tasks) {
                            returnData.push({ json: task });
                        }
                    }

                    if (operation === 'update') {
                        const taskId = this.getNodeParameter('taskId', i) as string;
                        const additionalFields = this.getNodeParameter('additionalFields', i) as IDataObject;
                        const response = await this.helpers.httpRequest({
                            method: 'PATCH',
                            url: `${credentials.baseUrl}/tasks/${taskId}`,
                            body: additionalFields,
                            headers: { Authorization: `Bearer ${credentials.apiKey}` },
                            json: true,
                        });
                        returnData.push({ json: response as IDataObject });
                    }

                    if (operation === 'delete') {
                        const taskId = this.getNodeParameter('taskId', i) as string;
                        await this.helpers.httpRequest({
                            method: 'DELETE',
                            url: `${credentials.baseUrl}/tasks/${taskId}`,
                            headers: { Authorization: `Bearer ${credentials.apiKey}` },
                        });
                        returnData.push({ json: { success: true, taskId } });
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

## Example 2: Complete Declarative Node

REST API wrapper using routing configuration — no `execute()` method needed.

```typescript
import type { INodeType, INodeTypeDescription } from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class BookmarkApi implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Bookmark API',
        name: 'bookmarkApi',
        icon: 'file:bookmark.svg',
        group: ['input'],
        version: 1,
        subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
        description: 'Interact with the Bookmark API',
        defaults: { name: 'Bookmark API' },
        usableAsTool: true,
        inputs: [NodeConnectionTypes.Main],
        outputs: [NodeConnectionTypes.Main],
        credentials: [
            {
                name: 'bookmarkApi',
                required: true,
            },
        ],
        requestDefaults: {
            baseURL: 'https://api.bookmarks.example.com/v1',
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
                    { name: 'Bookmark', value: 'bookmark' },
                ],
                default: 'bookmark',
            },
            {
                displayName: 'Operation',
                name: 'operation',
                type: 'options',
                noDataExpression: true,
                displayOptions: { show: { resource: ['bookmark'] } },
                options: [
                    {
                        name: 'Create',
                        value: 'create',
                        action: 'Create a bookmark',
                        routing: {
                            request: {
                                method: 'POST',
                                url: '/bookmarks',
                            },
                        },
                    },
                    {
                        name: 'Get Many',
                        value: 'getAll',
                        action: 'Get many bookmarks',
                        routing: {
                            request: {
                                method: 'GET',
                                url: '/bookmarks',
                            },
                            output: {
                                postReceive: [
                                    {
                                        type: 'rootProperty',
                                        properties: { property: 'data' },
                                    },
                                ],
                            },
                        },
                    },
                    {
                        name: 'Delete',
                        value: 'delete',
                        action: 'Delete a bookmark',
                        routing: {
                            request: {
                                method: 'DELETE',
                                url: '=/bookmarks/{{$parameter.bookmarkId}}',
                            },
                        },
                    },
                ],
                default: 'getAll',
            },
            // Parameters with routing.send
            {
                displayName: 'URL',
                name: 'url',
                type: 'string',
                required: true,
                default: '',
                displayOptions: { show: { operation: ['create'] } },
                routing: {
                    send: { type: 'body', property: 'url' },
                },
            },
            {
                displayName: 'Title',
                name: 'title',
                type: 'string',
                default: '',
                displayOptions: { show: { operation: ['create'] } },
                routing: {
                    send: { type: 'body', property: 'title' },
                },
            },
            {
                displayName: 'Bookmark ID',
                name: 'bookmarkId',
                type: 'string',
                required: true,
                default: '',
                displayOptions: { show: { operation: ['delete'] } },
            },
            {
                displayName: 'Limit',
                name: 'limit',
                type: 'number',
                default: 50,
                typeOptions: { minValue: 1 },
                displayOptions: { show: { operation: ['getAll'] } },
                routing: {
                    send: { type: 'query', property: 'limit' },
                },
            },
        ],
    };

    // No execute() method — routing handles everything
}
```

## Example 3: Complete Credential with OAuth2

```typescript
import type { ICredentialType, INodeProperties } from 'n8n-workflow';

export class MyOAuth2Api implements ICredentialType {
    name = 'myOAuth2Api';
    displayName = 'My Service OAuth2 API';
    documentationUrl = 'https://docs.example.com/oauth';
    extends = ['oAuth2Api'];  // Extends base OAuth2 credential
    properties: INodeProperties[] = [
        {
            displayName: 'Grant Type',
            name: 'grantType',
            type: 'hidden',
            default: 'authorizationCode',
        },
        {
            displayName: 'Authorization URL',
            name: 'authUrl',
            type: 'hidden',
            default: 'https://auth.example.com/oauth/authorize',
        },
        {
            displayName: 'Access Token URL',
            name: 'accessTokenUrl',
            type: 'hidden',
            default: 'https://auth.example.com/oauth/token',
        },
        {
            displayName: 'Scope',
            name: 'scope',
            type: 'hidden',
            default: 'read write',
        },
        {
            displayName: 'Auth URI Query Parameters',
            name: 'authQueryParameters',
            type: 'hidden',
            default: '',
        },
        {
            displayName: 'Authentication',
            name: 'authentication',
            type: 'hidden',
            default: 'body',  // 'body' or 'header'
        },
    ];
}
```

## Example 4: Complete Credential with API Key

```typescript
import type { IAuthenticate, ICredentialType, INodeProperties, ICredentialTestRequest } from 'n8n-workflow';

export class TaskManagerApi implements ICredentialType {
    name = 'taskManagerApi';
    displayName = 'Task Manager API';
    documentationUrl = 'https://docs.taskmanager.example.com';
    properties: INodeProperties[] = [
        {
            displayName: 'API Key',
            name: 'apiKey',
            type: 'string',
            typeOptions: { password: true },
            default: '',
            required: true,
        },
        {
            displayName: 'Base URL',
            name: 'baseUrl',
            type: 'string',
            default: 'https://api.taskmanager.example.com',
            required: true,
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

    test: ICredentialTestRequest = {
        request: {
            baseURL: '={{$credentials.baseUrl}}',
            url: '/me',
        },
    };
}
```

## Example 5: Test Setup with WorkflowTestData

```typescript
import type { WorkflowTestData } from 'n8n-workflow';

const testData: WorkflowTestData = {
    description: 'Should get a task by ID',
    input: {
        workflowData: {
            nodes: [
                {
                    id: 'trigger-1',
                    name: 'Manual Trigger',
                    type: 'n8n-nodes-base.manualTrigger',
                    typeVersion: 1,
                    position: [250, 300],
                    parameters: {},
                },
                {
                    id: 'node-1',
                    name: 'Task Manager',
                    type: 'n8n-nodes-myservice.taskManager',
                    typeVersion: 1,
                    position: [450, 300],
                    parameters: {
                        resource: 'task',
                        operation: 'get',
                        taskId: 'task-123',
                    },
                    credentials: {
                        taskManagerApi: {
                            id: 'cred-1',
                            name: 'Test Credential',
                        },
                    },
                },
            ],
            connections: {
                'Manual Trigger': {
                    main: [
                        [
                            { node: 'Task Manager', type: 'main', index: 0 },
                        ],
                    ],
                },
            },
        },
    },
    output: {
        nodeExecutionOrder: ['Manual Trigger', 'Task Manager'],
        nodeData: {
            'Task Manager': [
                [
                    {
                        json: {
                            id: 'task-123',
                            title: 'Test Task',
                            status: 'active',
                        },
                    },
                ],
            ],
        },
    },
    nock: {
        baseUrl: 'https://api.taskmanager.example.com',
        mocks: [
            {
                method: 'get',
                path: '/tasks/task-123',
                statusCode: 200,
                responseBody: {
                    id: 'task-123',
                    title: 'Test Task',
                    status: 'active',
                },
            },
        ],
    },
    credentials: {
        taskManagerApi: {
            apiKey: 'test-api-key',
            baseUrl: 'https://api.taskmanager.example.com',
        },
    },
};
```

### WorkflowTestData Structure

| Field | Purpose |
|-------|---------|
| `description` | Test case name |
| `input.workflowData` | Workflow definition (nodes + connections) |
| `output.nodeExecutionOrder` | Expected execution order of nodes |
| `output.nodeData` | Expected output data per node |
| `nock` | HTTP mock definitions (base URL + request/response pairs) |
| `credentials` | Test credential values |

## Example 6: Versioned Node

```typescript
import type { IVersionedNodeType, INodeTypeDescription } from 'n8n-workflow';
import { VersionedNodeType } from 'n8n-workflow';

import { MyNodeV1 } from './v1/MyNodeV1.node';
import { MyNodeV2 } from './v2/MyNodeV2.node';

export class MyNode extends VersionedNodeType {
    constructor() {
        const baseDescription: INodeTypeDescription = {
            displayName: 'My Node',
            name: 'myNode',
            icon: 'file:myNode.svg',
            group: ['transform'],
            description: 'Interact with My Service',
            defaultVersion: 2,
        };

        const nodeVersions = {
            1: new MyNodeV1(baseDescription),
            2: new MyNodeV2(baseDescription),
        };

        super(nodeVersions, baseDescription);
    }
}
```

This pattern allows backward compatibility — existing workflows using v1 continue to work while new workflows get v2 by default.
