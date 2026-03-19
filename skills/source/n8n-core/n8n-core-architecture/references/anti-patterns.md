# n8n-core-architecture — Anti-Patterns

> Common mistakes in n8n v1.x node development and workflow design. Each anti-pattern includes what goes wrong and the correct approach.

---

## Data Flow Mistakes

### AP-001: Returning Flat Array from execute()

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        returnData.push({ json: { result: true } });
    }

    // WRONG: returning flat array instead of nested
    return returnData as any;
}
```

**WHY it breaks:** n8n expects `INodeExecutionData[][]` where the outer array represents output indices. A flat array causes runtime errors or silent data loss — downstream nodes receive nothing or corrupted data.

**CORRECT:**
```typescript
return [returnData];  // ALWAYS wrap in outer array
```

---

### AP-002: Mutating Input Items

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();

    for (let i = 0; i < items.length; i++) {
        // WRONG: mutating the input item directly
        items[i].json.processed = true;
        items[i].json.timestamp = Date.now();
    }

    return [items];
}
```

**WHY it breaks:** Input items are shared references. Mutating them corrupts data for other nodes that branch from the same source. This causes non-deterministic behavior that is extremely difficult to debug.

**CORRECT:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        // Create new objects — NEVER mutate input
        returnData.push({
            json: { ...items[i].json, processed: true, timestamp: Date.now() },
            binary: items[i].binary,
            pairedItem: { item: i },
        });
    }

    return [returnData];
}
```

---

### AP-003: Hardcoded Item Index for Parameters

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    // WRONG: using hardcoded 0 for all items
    const url = this.getNodeParameter('url', 0) as string;

    for (let i = 0; i < items.length; i++) {
        const response = await this.helpers.httpRequest({ method: 'GET', url });
        returnData.push({ json: response as IDataObject });
    }

    return [returnData];
}
```

**WHY it breaks:** When users use expressions like `{{ $json.endpoint }}` in the URL parameter, the value differs per item. Using index `0` means ALL items get the URL from the first item only.

**CORRECT:**
```typescript
for (let i = 0; i < items.length; i++) {
    // ALWAYS use the loop index i
    const url = this.getNodeParameter('url', i) as string;
    const response = await this.helpers.httpRequest({ method: 'GET', url });
    returnData.push({ json: response as IDataObject, pairedItem: { item: i } });
}
```

---

### AP-004: Dropping Binary Data

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        // WRONG: creating new item without preserving binary
        returnData.push({
            json: { ...items[i].json, enriched: true },
            // binary is missing — files are lost!
        });
    }

    return [returnData];
}
```

**WHY it breaks:** If upstream nodes attached files (images, PDFs, etc.) to items, they silently disappear. Downstream nodes expecting binary data will fail or produce empty results.

**CORRECT:**
```typescript
returnData.push({
    json: { ...items[i].json, enriched: true },
    binary: items[i].binary,  // ALWAYS preserve binary when passing through
    pairedItem: { item: i },
});
```

---

## Item Handling Errors

### AP-005: Missing pairedItem

**WRONG:**
```typescript
for (let i = 0; i < items.length; i++) {
    returnData.push({
        json: { result: 'processed' },
        // pairedItem is missing!
    });
}
```

**WHY it breaks:** Without `pairedItem`, n8n cannot trace data lineage between nodes. The "Input/Output" panel in the editor cannot show which input item produced which output item. Pin data and error tracing also break.

**CORRECT:**
```typescript
for (let i = 0; i < items.length; i++) {
    returnData.push({
        json: { result: 'processed' },
        pairedItem: { item: i },  // ALWAYS set pairedItem
    });
}
```

---

### AP-006: Missing continueOnFail Check

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        // WRONG: no try/catch, no continueOnFail check
        const response = await this.helpers.httpRequest({
            method: 'GET',
            url: this.getNodeParameter('url', i) as string,
        });
        returnData.push({ json: response as IDataObject });
    }

    return [returnData];
}
```

**WHY it breaks:** If any item fails, the entire node execution stops. Users who configured "Continue On Fail" in the node settings expect failed items to pass through with error info, not crash the entire workflow.

**CORRECT:**
```typescript
for (let i = 0; i < items.length; i++) {
    try {
        const response = await this.helpers.httpRequest({
            method: 'GET',
            url: this.getNodeParameter('url', i) as string,
        });
        returnData.push({
            json: response as IDataObject,
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
```

---

### AP-007: Accessing Binary Without Null Check

**WRONG:**
```typescript
for (let i = 0; i < items.length; i++) {
    // WRONG: accessing binary without checking existence
    const fileName = items[i].binary!.data.fileName;
    const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');
}
```

**WHY it breaks:** Not all items carry binary data. Using the non-null assertion (`!`) causes a runtime crash with `TypeError: Cannot read properties of undefined` when an item lacks binary data.

**CORRECT:**
```typescript
for (let i = 0; i < items.length; i++) {
    if (!items[i].binary?.data) {
        if (this.continueOnFail()) {
            returnData.push({
                json: { error: 'No binary data found on item' },
                pairedItem: { item: i },
            });
            continue;
        }
        throw new Error(`Item ${i} has no binary data in property "data"`);
    }

    const binaryData = this.helpers.assertBinaryData(i, 'data');
    const buffer = await this.helpers.getBinaryDataBuffer(i, 'data');
}
```

---

### AP-008: Using External HTTP Libraries Instead of Helpers

**WRONG:**
```typescript
import axios from 'axios';

async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    // WRONG: using axios directly bypasses n8n's credential injection,
    // proxy settings, retry logic, and error handling
    const response = await axios.get('https://api.example.com/data');
    return [[{ json: response.data }]];
}
```

**WHY it breaks:** External HTTP libraries bypass n8n's built-in authentication handling, proxy configuration, timeout management, and error formatting. Credentials configured in the n8n UI will not be applied.

**CORRECT:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const response = await this.helpers.httpRequest({
        method: 'GET',
        url: 'https://api.example.com/data',
    });
    return [[{ json: response as IDataObject }]];
}
```

---

## Workflow Settings Mistakes

### AP-009: Using executionOrder v0 in New Workflows

**WRONG:**
```json
{
    "settings": {
        "executionOrder": "v0"
    }
}
```

**WHY it breaks:** The v0 execution order is legacy and produces non-deterministic behavior when nodes have multiple inputs. Nodes may execute in unexpected order, causing race conditions and inconsistent results.

**CORRECT:**
```json
{
    "settings": {
        "executionOrder": "v1"
    }
}
```

ALWAYS use `executionOrder: "v1"` for new workflows. The v1 algorithm uses breadth-first execution with deterministic ordering.

---

### AP-010: Trigger Node with Inputs Defined

**WRONG:**
```typescript
description: INodeTypeDescription = {
    displayName: 'My Trigger',
    name: 'myTrigger',
    group: ['trigger'],
    // WRONG: trigger nodes must have empty inputs
    inputs: [NodeConnectionTypes.Main],
    outputs: [NodeConnectionTypes.Main],
    // ...
};
```

**WHY it breaks:** Trigger nodes are workflow entry points. They NEVER receive input from other nodes. Defining inputs on a trigger node creates an invalid node configuration that confuses the workflow editor and execution engine.

**CORRECT:**
```typescript
description: INodeTypeDescription = {
    displayName: 'My Trigger',
    name: 'myTrigger',
    group: ['trigger'],
    inputs: [],  // ALWAYS empty for trigger nodes
    outputs: [NodeConnectionTypes.Main],
    // ...
};
```

---

### AP-011: Forgetting closeFunction in Trigger Nodes

**WRONG:**
```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const interval = setInterval(() => {
        this.emit([this.helpers.returnJsonArray([{ tick: Date.now() }])]);
    }, 60000);

    // WRONG: no closeFunction — interval keeps running forever
    return {};
}
```

**WHY it breaks:** Without a `closeFunction`, resources (intervals, event listeners, connections) are never cleaned up when the workflow is deactivated. This causes memory leaks and phantom executions.

**CORRECT:**
```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const interval = setInterval(() => {
        this.emit([this.helpers.returnJsonArray([{ tick: Date.now() }])]);
    }, 60000);

    return {
        closeFunction: async () => {
            clearInterval(interval);  // ALWAYS clean up resources
        },
    };
}
```
