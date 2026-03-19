# Trigger Node Examples

> Complete, production-ready implementations for all three trigger patterns.

## Example 1: Schedule/Event Trigger (Interval-Based)

A trigger that emits data on a configurable interval, with proper cleanup and manual mode support.

```typescript
import type {
    ITriggerFunctions,
    INodeType,
    INodeTypeDescription,
    ITriggerResponse,
    INodeExecutionData,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class IntervalTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Interval Trigger',
        name: 'intervalTrigger',
        icon: 'fa:clock',
        group: ['trigger'],
        version: 1,
        description: 'Triggers the workflow at a fixed interval',
        eventTriggerDescription: 'Waiting for the next scheduled interval',
        activationMessage: 'Your interval trigger is now active.',
        defaults: { name: 'Interval Trigger' },
        inputs: [],
        outputs: [NodeConnectionTypes.Main],
        properties: [
            {
                displayName: 'Interval (Seconds)',
                name: 'intervalSeconds',
                type: 'number',
                default: 60,
                typeOptions: { minValue: 1 },
                description: 'How often to trigger (in seconds)',
            },
            {
                displayName: 'Include Timestamp',
                name: 'includeTimestamp',
                type: 'boolean',
                default: true,
                description: 'Whether to include the trigger timestamp in output',
            },
        ],
    };

    async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
        const intervalSeconds = this.getNodeParameter('intervalSeconds') as number;
        const includeTimestamp = this.getNodeParameter('includeTimestamp') as boolean;

        const executeTrigger = () => {
            const item: INodeExecutionData = {
                json: {
                    triggered: true,
                    ...(includeTimestamp && { timestamp: new Date().toISOString() }),
                },
            };
            // CRITICAL: Double-wrap — outer array = outputs, inner array = items
            this.emit([[item]]);
        };

        if (this.getMode() === 'manual') {
            // Manual mode: execute once immediately for testing
            return {
                manualTriggerFunction: async () => {
                    executeTrigger();
                },
            };
        }

        // Production mode: set up recurring interval
        const intervalMs = intervalSeconds * 1000;
        const intervalId = setInterval(executeTrigger, intervalMs);

        // ALWAYS return closeFunction to clean up the interval
        return {
            closeFunction: async () => {
                clearInterval(intervalId);
            },
        };
    }
}
```

## Example 2: Schedule Trigger (Cron-Based)

Using n8n's built-in cron scheduling via `this.helpers.registerCron()`.

```typescript
import type {
    ITriggerFunctions,
    INodeType,
    INodeTypeDescription,
    ITriggerResponse,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class CronTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'Cron Trigger',
        name: 'cronTrigger',
        icon: 'fa:calendar',
        group: ['trigger', 'schedule'],
        version: 1,
        description: 'Triggers on a cron schedule',
        defaults: { name: 'Cron Trigger' },
        inputs: [],
        outputs: [NodeConnectionTypes.Main],
        properties: [
            {
                displayName: 'Cron Expression',
                name: 'cronExpression',
                type: 'string',
                default: '0 * * * *',
                description: 'Cron expression (e.g., "0 * * * *" for every hour)',
            },
        ],
    };

    async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
        const cronExpression = this.getNodeParameter('cronExpression') as string;

        const executeTrigger = () => {
            this.emit([
                this.helpers.returnJsonArray([
                    {
                        timestamp: new Date().toISOString(),
                        cronExpression,
                    },
                ]),
            ]);
        };

        if (this.getMode() === 'manual') {
            return {
                manualTriggerFunction: async () => executeTrigger(),
            };
        }

        // registerCron handles cleanup automatically — no closeFunction needed
        this.helpers.registerCron(cronExpression, executeTrigger);

        return {};
    }
}
```

## Example 3: Webhook Trigger (Full-Featured)

A webhook trigger with method selection, response mode, and header/query access.

```typescript
import type {
    IWebhookFunctions,
    INodeType,
    INodeTypeDescription,
    IWebhookResponseData,
    INodeExecutionData,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class MyWebhookTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'My Webhook',
        name: 'myWebhook',
        icon: 'fa:bolt',
        group: ['trigger'],
        version: 1,
        description: 'Starts workflow on HTTP request',
        defaults: { name: 'My Webhook' },
        inputs: [],
        outputs: [NodeConnectionTypes.Main],
        webhooks: [
            {
                name: 'default',
                httpMethod: '={{$parameter["httpMethod"]}}',
                path: '={{$parameter["path"]}}',
                responseMode: '={{$parameter["responseMode"]}}',
                responseData: '={{$parameter["responseData"]}}',
            },
        ],
        properties: [
            {
                displayName: 'HTTP Method',
                name: 'httpMethod',
                type: 'options',
                options: [
                    { name: 'GET', value: 'GET' },
                    { name: 'POST', value: 'POST' },
                    { name: 'PUT', value: 'PUT' },
                    { name: 'DELETE', value: 'DELETE' },
                ],
                default: 'POST',
                description: 'The HTTP method to listen for',
            },
            {
                displayName: 'Path',
                name: 'path',
                type: 'string',
                default: 'my-webhook',
                required: true,
                description: 'The webhook URL path',
            },
            {
                displayName: 'Response Mode',
                name: 'responseMode',
                type: 'options',
                options: [
                    {
                        name: 'On Received',
                        value: 'onReceived',
                        description: 'Respond immediately',
                    },
                    {
                        name: 'Last Node',
                        value: 'lastNode',
                        description: 'Respond after workflow completes',
                    },
                    {
                        name: 'Response Node',
                        value: 'responseNode',
                        description: 'Respond from a Respond to Webhook node',
                    },
                ],
                default: 'onReceived',
            },
            {
                displayName: 'Response Data',
                name: 'responseData',
                type: 'options',
                options: [
                    { name: 'All Entries', value: 'allEntries' },
                    { name: 'First Entry (JSON)', value: 'firstEntryJson' },
                    { name: 'No Data', value: 'noData' },
                ],
                default: 'firstEntryJson',
                displayOptions: {
                    show: { responseMode: ['lastNode'] },
                },
            },
        ],
    };

    async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
        const body = this.getBodyData();
        const headers = this.getHeaderData();
        const query = this.getQueryData();

        const item: INodeExecutionData = {
            json: {
                body,
                headers,
                query,
            },
        };

        return {
            workflowData: [[item]],
        };
    }
}
```

## Example 4: Webhook with External Registration (webhookMethods)

A trigger that registers a webhook on an external service (e.g., a SaaS API).

```typescript
import type {
    IHookFunctions,
    IWebhookFunctions,
    INodeType,
    INodeTypeDescription,
    IWebhookResponseData,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class ExternalServiceTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'External Service Trigger',
        name: 'externalServiceTrigger',
        group: ['trigger'],
        version: 1,
        inputs: [],
        outputs: [NodeConnectionTypes.Main],
        defaults: { name: 'External Service Trigger' },
        credentials: [
            {
                name: 'externalServiceApi',
                required: true,
            },
        ],
        webhooks: [
            {
                name: 'default',
                httpMethod: 'POST',
                path: 'webhook',
                responseMode: 'onReceived',
            },
        ],
        properties: [
            {
                displayName: 'Events',
                name: 'events',
                type: 'multiOptions',
                options: [
                    { name: 'Item Created', value: 'item.created' },
                    { name: 'Item Updated', value: 'item.updated' },
                    { name: 'Item Deleted', value: 'item.deleted' },
                ],
                default: ['item.created'],
                required: true,
            },
        ],
    };

    webhookMethods = {
        default: {
            async checkExists(this: IHookFunctions): Promise<boolean> {
                const credentials = await this.getCredentials('externalServiceApi');
                const webhookUrl = this.getNodeWebhookUrl('default');
                const staticData = this.getWorkflowStaticData('node');

                // Check if we previously stored a webhook ID
                if (!staticData.webhookId) {
                    return false;
                }

                // Verify it still exists on the external service
                try {
                    await this.helpers.httpRequest({
                        method: 'GET',
                        url: `https://api.external.com/webhooks/${staticData.webhookId}`,
                        headers: {
                            Authorization: `Bearer ${credentials.apiKey}`,
                        },
                    });
                    return true;
                } catch {
                    // Webhook was deleted externally
                    delete staticData.webhookId;
                    return false;
                }
            },

            async create(this: IHookFunctions): Promise<boolean> {
                const credentials = await this.getCredentials('externalServiceApi');
                const webhookUrl = this.getNodeWebhookUrl('default');
                const events = this.getNodeParameter('events') as string[];
                const staticData = this.getWorkflowStaticData('node');

                const response = await this.helpers.httpRequest({
                    method: 'POST',
                    url: 'https://api.external.com/webhooks',
                    headers: {
                        Authorization: `Bearer ${credentials.apiKey}`,
                    },
                    body: {
                        url: webhookUrl,
                        events,
                    },
                });

                // ALWAYS store webhook ID for deletion later
                staticData.webhookId = (response as { id: string }).id;
                return true;
            },

            async delete(this: IHookFunctions): Promise<boolean> {
                const credentials = await this.getCredentials('externalServiceApi');
                const staticData = this.getWorkflowStaticData('node');
                const webhookId = staticData.webhookId as string;

                if (!webhookId) {
                    return true; // Nothing to delete
                }

                try {
                    await this.helpers.httpRequest({
                        method: 'DELETE',
                        url: `https://api.external.com/webhooks/${webhookId}`,
                        headers: {
                            Authorization: `Bearer ${credentials.apiKey}`,
                        },
                    });
                } catch {
                    // Best effort — webhook may already be gone
                }

                delete staticData.webhookId;
                return true;
            },
        },
    };

    async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
        const body = this.getBodyData();

        // Validate the incoming webhook (e.g., check signature)
        const headers = this.getHeaderData();
        const signature = headers['x-webhook-signature'] as string;

        if (!signature) {
            // Return error response to caller, do NOT trigger workflow
            return {
                webhookResponse: { error: 'Missing signature' },
                workflowData: undefined,
            };
        }

        return {
            workflowData: [[
                {
                    json: {
                        event: body.event,
                        data: body.data,
                        receivedAt: new Date().toISOString(),
                    },
                },
            ]],
        };
    }
}
```

## Example 5: Polling Trigger with Deduplication

A polling trigger that checks an API for new items, using static data for deduplication.

```typescript
import type {
    IPollFunctions,
    INodeType,
    INodeTypeDescription,
    INodeExecutionData,
    IDataObject,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';

export class NewItemsPollingTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'New Items (Polling)',
        name: 'newItemsPolling',
        icon: 'fa:sync',
        group: ['trigger'],
        version: 1,
        description: 'Polls for new items from an API',
        polling: true,               // REQUIRED for polling triggers
        defaults: { name: 'New Items' },
        inputs: [],
        outputs: [NodeConnectionTypes.Main],
        credentials: [
            {
                name: 'externalServiceApi',
                required: true,
            },
        ],
        properties: [
            {
                displayName: 'Resource URL',
                name: 'resourceUrl',
                type: 'string',
                default: '',
                required: true,
                description: 'The API endpoint to poll for new items',
            },
            {
                displayName: 'Timestamp Field',
                name: 'timestampField',
                type: 'string',
                default: 'created_at',
                description: 'The field name containing the item creation timestamp',
            },
        ],
    };

    async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
        const resourceUrl = this.getNodeParameter('resourceUrl') as string;
        const timestampField = this.getNodeParameter('timestampField') as string;
        const credentials = await this.getCredentials('externalServiceApi');

        // Get persistent storage for deduplication
        const staticData = this.getWorkflowStaticData('node');
        const lastTimestamp = staticData.lastTimestamp as string | undefined;

        // Build query with "since" filter if we have a previous timestamp
        const qs: IDataObject = { sort: timestampField, order: 'asc' };
        if (lastTimestamp) {
            qs.since = lastTimestamp;
        }

        const response = await this.helpers.httpRequest({
            method: 'GET',
            url: resourceUrl,
            qs,
            headers: {
                Authorization: `Bearer ${credentials.apiKey}`,
            },
        });

        const items = response as IDataObject[];

        // ALWAYS return null when no new data — NEVER return empty array
        if (!items || items.length === 0) {
            return null;
        }

        // Filter out the item matching lastTimestamp exactly (avoid reprocessing)
        const newItems = lastTimestamp
            ? items.filter((item) => item[timestampField] !== lastTimestamp)
            : items;

        if (newItems.length === 0) {
            return null;
        }

        // Update stored timestamp to the latest item
        const latestItem = newItems[newItems.length - 1];
        staticData.lastTimestamp = latestItem[timestampField] as string;

        // Convert to n8n items and return
        return [newItems.map((item) => ({ json: item }))];
    }
}
```

## Example 6: WebSocket Event Trigger

An event trigger that connects to a WebSocket and emits on each message.

```typescript
import type {
    ITriggerFunctions,
    INodeType,
    INodeTypeDescription,
    ITriggerResponse,
} from 'n8n-workflow';
import { NodeConnectionTypes } from 'n8n-workflow';
import WebSocket from 'ws';

export class WebSocketTrigger implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'WebSocket Trigger',
        name: 'webSocketTrigger',
        group: ['trigger'],
        version: 1,
        description: 'Triggers on WebSocket messages',
        defaults: { name: 'WebSocket Trigger' },
        inputs: [],
        outputs: [NodeConnectionTypes.Main],
        properties: [
            {
                displayName: 'WebSocket URL',
                name: 'wsUrl',
                type: 'string',
                default: '',
                required: true,
                placeholder: 'wss://stream.example.com/events',
            },
        ],
    };

    async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
        const wsUrl = this.getNodeParameter('wsUrl') as string;

        if (this.getMode() === 'manual') {
            return {
                manualTriggerFunction: async () => {
                    // Connect briefly, emit first message, then disconnect
                    const ws = new WebSocket(wsUrl);
                    return new Promise<void>((resolve, reject) => {
                        ws.on('message', (data) => {
                            const parsed = JSON.parse(data.toString());
                            this.emit([[ { json: parsed } ]]);
                            ws.close();
                            resolve();
                        });
                        ws.on('error', reject);
                        // Timeout after 30 seconds
                        setTimeout(() => {
                            ws.close();
                            reject(new Error('Manual trigger timed out'));
                        }, 30_000);
                    });
                },
            };
        }

        // Production mode: persistent connection
        const ws = new WebSocket(wsUrl);

        ws.on('message', (data) => {
            try {
                const parsed = JSON.parse(data.toString());
                this.emit([[ { json: parsed } ]]);
            } catch (error) {
                // Non-fatal: log and continue listening
                this.saveFailedExecution(error as Error);
            }
        });

        ws.on('error', (error) => {
            // Fatal: deactivate and trigger reactivation backoff
            this.emitError(error);
        });

        // CRITICAL: Clean up WebSocket on workflow deactivation
        return {
            closeFunction: async () => {
                ws.close();
            },
        };
    }
}
```
