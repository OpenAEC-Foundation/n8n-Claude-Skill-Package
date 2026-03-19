# n8n-core-architecture — Examples

> Working code examples verified against n8n v1.x source code and official documentation.

---

## Item Processing

### Basic Item Loop (Standard Pattern)

ALWAYS use this pattern for processing items in a custom node:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        try {
            const value = this.getNodeParameter('myParam', i) as string;
            const inputJson = items[i].json;

            returnData.push({
                json: { ...inputJson, processed: true, value },
                pairedItem: { item: i },
            });
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
```

### Multi-Output Node (IF-Style)

Return multiple arrays in the outer array for nodes with multiple outputs:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const trueItems: INodeExecutionData[] = [];
    const falseItems: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const condition = this.getNodeParameter('condition', i) as boolean;

        if (condition) {
            trueItems.push({
                json: items[i].json,
                binary: items[i].binary,
                pairedItem: { item: i },
            });
        } else {
            falseItems.push({
                json: items[i].json,
                binary: items[i].binary,
                pairedItem: { item: i },
            });
        }
    }

    return [trueItems, falseItems];
}
```

### Aggregation (Many Items to One)

When aggregating multiple items into a single output, link all source items:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const allValues: string[] = [];

    for (let i = 0; i < items.length; i++) {
        allValues.push(items[i].json.name as string);
    }

    const returnData: INodeExecutionData[] = [{
        json: {
            count: items.length,
            names: allValues,
        },
        pairedItem: items.map((_, i) => ({ item: i })),
    }];

    return [returnData];
}
```

### Expansion (One Item to Many)

When expanding one item into multiple outputs, link all back to the source:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const records = items[i].json.records as IDataObject[];

        for (const record of records) {
            returnData.push({
                json: record,
                pairedItem: { item: i },
            });
        }
    }

    return [returnData];
}
```

---

## Binary Data Handling

### Reading Binary Data from Input

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        // Assert binary data exists (throws if missing)
        const binaryData = this.helpers.assertBinaryData(i, 'data');

        // Get the actual file content as a Buffer
        const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');

        // Access metadata
        const fileName = binaryData.fileName ?? 'unknown';
        const mimeType = binaryData.mimeType;
        const fileSize = buffer.length;

        returnData.push({
            json: { fileName, mimeType, fileSize },
            pairedItem: { item: i },
        });
    }

    return [returnData];
}
```

### Creating Binary Data Output

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const content = this.getNodeParameter('content', i) as string;
        const buffer = Buffer.from(content, 'utf-8');

        // prepareBinaryData handles storage (memory or filesystem)
        const binaryData = await this.helpers.prepareBinaryData(
            buffer,
            'output.txt',
            'text/plain',
        );

        returnData.push({
            json: items[i].json,
            binary: { data: binaryData },
            pairedItem: { item: i },
        });
    }

    return [returnData];
}
```

### Passing Through Binary Data

ALWAYS preserve binary data when passing items through unchanged:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        returnData.push({
            json: { ...items[i].json, enriched: true },
            binary: items[i].binary,  // Pass through binary data
            pairedItem: { item: i },
        });
    }

    return [returnData];
}
```

### Multiple Binary Properties

An item can carry multiple binary files under different keys:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const pdfBuffer = Buffer.from('...pdf content...');
        const csvBuffer = Buffer.from('col1,col2\nval1,val2');

        const pdfBinary = await this.helpers.prepareBinaryData(
            pdfBuffer, 'report.pdf', 'application/pdf',
        );
        const csvBinary = await this.helpers.prepareBinaryData(
            csvBuffer, 'data.csv', 'text/csv',
        );

        returnData.push({
            json: { filesGenerated: 2 },
            binary: {
                pdf: pdfBinary,
                csv: csvBinary,
            },
            pairedItem: { item: i },
        });
    }

    return [returnData];
}
```

---

## Paired Items

### Standard 1:1 Mapping

```typescript
// Input item at index i produces output item linked back to i
returnData.push({
    json: { result: 'processed' },
    pairedItem: { item: i },
});
```

### Shorthand (Number Only)

```typescript
// Equivalent to { item: i } when input index is 0
returnData.push({
    json: { result: 'processed' },
    pairedItem: i,
});
```

### Using constructExecutionMetaData

For API responses that return arrays, use this helper to set paired items in bulk:

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const response = await this.helpers.httpRequest({
            method: 'GET',
            url: 'https://api.example.com/users',
        });

        const users = (response as IDataObject[]).map(
            (user) => ({ json: user }) as INodeExecutionData,
        );

        // Sets pairedItem on all items in the array
        const executionData = this.helpers.constructExecutionMetaData(
            users,
            { itemData: { item: i } },
        );

        returnData.push(...executionData);
    }

    return [returnData];
}
```

---

## Workflow Settings

### Setting Workflow Configuration

In a workflow JSON file, configure settings at the top level:

```json
{
    "settings": {
        "timezone": "Europe/Amsterdam",
        "executionOrder": "v1",
        "errorWorkflow": "wf_error_handler_id",
        "executionTimeout": 300,
        "saveDataErrorExecution": "all",
        "saveDataSuccessExecution": "none",
        "saveManualExecutions": true,
        "callerPolicy": "workflowsFromSameOwner"
    }
}
```

### Accessing Workflow Info in Expressions

```
{{ $workflow.id }}          → Workflow ID
{{ $workflow.name }}        → Workflow name
{{ $workflow.active }}      → Whether workflow is active (boolean)
{{ $execution.id }}         → Current execution ID
{{ $execution.mode }}       → "test" or "production"
{{ $execution.resumeUrl }}  → URL to resume a waiting execution
```

### Using $execution.customData

Store custom metadata during execution for later retrieval:

```
// In a Code node or expression:
$execution.customData.set("orderId", $json.order_id);
$execution.customData.set("customerName", $json.name);

// Later in the same execution:
$execution.customData.get("orderId");     // Returns the stored value
$execution.customData.getAll();           // Returns all stored key-value pairs
```

---

## Trigger Node Patterns

### Event Trigger (Schedule)

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const executeTrigger = () => {
        this.emit([
            this.helpers.returnJsonArray([
                { timestamp: new Date().toISOString() },
            ]),
        ]);
    };

    if (this.getMode() !== 'manual') {
        // Production: register recurring trigger
        this.helpers.registerCron('0 * * * *', () => executeTrigger());
        return {};
    } else {
        // Manual: fire once for testing
        return {
            manualTriggerFunction: async () => { executeTrigger(); },
        };
    }
}
```

### Webhook Trigger

```typescript
async webhook(context: IWebhookFunctions): Promise<IWebhookResponseData> {
    const body = context.getBodyData();
    const headers = context.getHeaderData();
    const query = context.getQueryData();

    return {
        workflowData: [[
            {
                json: {
                    body,
                    headers,
                    query,
                },
            },
        ]],
    };
}
```

### Error Handling in Triggers

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const poll = async () => {
        try {
            const data = await fetchData();
            this.emit([this.helpers.returnJsonArray(data)]);
        } catch (error) {
            // Non-fatal: log error but keep workflow active
            this.saveFailedExecution(error as Error);

            // OR fatal: deactivate workflow and queue for reactivation
            // this.emitError(error as Error);
        }
    };

    // ...
}
```
