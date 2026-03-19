# Anti-Patterns Reference -- n8n-syntax-node-types

## Node Type Mistakes

### AP-1: Wrong Return Type from execute()

**WRONG**: Returning a flat array of items.
```typescript
// BROKEN -- returns INodeExecutionData[] instead of INodeExecutionData[][]
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    return items.map(item => ({ json: item.json })); // ERROR: flat array
}
```

**CORRECT**: ALWAYS wrap in an outer array.
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData = items.map(item => ({ json: item.json }));
    return [returnData]; // Correct: [items] for single output
}
```

**Why**: The outer array represents output connectors. `[returnData]` means "all items go to output 0". Returning a flat array causes runtime errors or silent data loss.

---

### AP-2: Using `this` with Node Base Class

**WRONG**: Accessing `this` methods in a class that extends `Node`.
```typescript
export class MyNode extends Node {
    async execute(context: IExecuteFunctions) {
        const items = this.getInputData(); // ERROR: 'this' is the Node instance, not IExecuteFunctions
    }
}
```

**CORRECT**: Use the `context` parameter.
```typescript
export class MyNode extends Node {
    async execute(context: IExecuteFunctions) {
        const items = context.getInputData(); // Correct
        const value = context.getNodeParameter('param', 0);
    }
}
```

**Why**: The `Node` base class passes context as a function parameter. The `implements INodeType` pattern uses `this` binding. Mixing them causes `TypeError: this.getInputData is not a function`.

---

### AP-3: Adding execute() to Declarative Nodes

**WRONG**: Defining execute() alongside routing configuration.
```typescript
export class ApiNode implements INodeType {
    description = {
        requestDefaults: { baseURL: 'https://api.example.com' },
        properties: [{
            routing: { request: { method: 'GET', url: '/items' } },
            // ...
        }],
    };

    // WRONG -- this overrides the declarative routing
    async execute(this: IExecuteFunctions) {
        // ...
    }
}
```

**CORRECT**: For declarative nodes, do NOT define execute(). Let n8n handle requests.
```typescript
export class ApiNode implements INodeType {
    description = {
        requestDefaults: { baseURL: 'https://api.example.com' },
        properties: [{
            routing: { request: { method: 'GET', url: '/items' } },
        }],
    };
    // No execute() -- n8n handles HTTP requests based on routing
}
```

**Why**: When `execute()` exists, n8n calls it instead of processing the declarative routing. The routing config becomes dead code.

---

### AP-4: Defining Inputs on Trigger Nodes

**WRONG**: Giving a trigger node input connections.
```typescript
description = {
    group: ['trigger'],
    inputs: [NodeConnectionTypes.Main], // WRONG -- triggers cannot receive input
    outputs: [NodeConnectionTypes.Main],
};
```

**CORRECT**: Trigger nodes ALWAYS have empty inputs.
```typescript
description = {
    group: ['trigger'],
    inputs: [],  // Correct -- triggers start the workflow
    outputs: [NodeConnectionTypes.Main],
};
```

**Why**: Trigger nodes are entry points. They start workflow execution, they do not receive data from other nodes.

---

### AP-5: Not Handling continueOnFail()

**WRONG**: Throwing errors without checking the setting.
```typescript
for (let i = 0; i < items.length; i++) {
    const response = await this.helpers.httpRequest({ /* ... */ });
    returnData.push({ json: response });
    // If httpRequest throws, entire workflow fails even with "Continue On Fail" enabled
}
```

**CORRECT**: Catch errors and check continueOnFail().
```typescript
for (let i = 0; i < items.length; i++) {
    try {
        const response = await this.helpers.httpRequest({ /* ... */ });
        returnData.push({ json: response as IDataObject });
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

**Why**: Users enable "Continue On Fail" to gracefully handle errors on individual items. Without this check, a single bad item kills the entire execution.

---

## Property Configuration Errors

### AP-6: Missing noDataExpression on Resource/Operation

**WRONG**: Allowing expressions on resource/operation selectors.
```typescript
{
    displayName: 'Resource',
    name: 'resource',
    type: 'options',
    // Missing noDataExpression: true
    options: [
        { name: 'User', value: 'user' },
    ],
    default: 'user',
}
```

**CORRECT**: ALWAYS set noDataExpression: true on resource and operation.
```typescript
{
    displayName: 'Resource',
    name: 'resource',
    type: 'options',
    noDataExpression: true,  // Required
    options: [
        { name: 'User', value: 'user' },
    ],
    default: 'user',
}
```

**Why**: Resource and operation fields control which other fields are displayed via displayOptions. If they use expressions, n8n cannot determine at design time which fields to show, breaking the UI.

---

### AP-7: Wrong displayOptions Logic

**WRONG**: Thinking show conditions across keys use OR logic.
```typescript
// Developer intends: show when resource is 'user' OR operation is 'get'
displayOptions: {
    show: {
        resource: ['user'],    // These are AND-ed, not OR-ed
        operation: ['get'],
    },
}
// Actual behavior: show when resource is 'user' AND operation is 'get'
```

**CORRECT**: Values within the SAME key use OR. Different keys use AND.
```typescript
// OR within same key: operation is 'get' OR 'update'
displayOptions: {
    show: {
        operation: ['get', 'update'],
    },
}

// AND across keys: resource is 'user' AND operation is 'get'
displayOptions: {
    show: {
        resource: ['user'],
        operation: ['get'],
    },
}
```

**Why**: n8n evaluates show conditions as: ALL keys must match (AND), within each key ANY value can match (OR). Misunderstanding this leads to fields appearing at wrong times.

---

### AP-8: Not Providing Default Value Matching Property Type

**WRONG**: Mismatched defaults.
```typescript
// Boolean property with string default
{ type: 'boolean', default: 'false' }  // WRONG: should be false (boolean)

// Options with no matching default
{ type: 'options', options: [{ name: 'A', value: 'a' }], default: 'b' }  // WRONG: 'b' not in options

// Collection with array default
{ type: 'collection', default: [] }  // WRONG: should be {}

// MultiOptions with object default
{ type: 'multiOptions', default: {} }  // WRONG: should be []
```

**CORRECT**: Match default type to property type.
```typescript
{ type: 'boolean', default: false }
{ type: 'options', options: [{ name: 'A', value: 'a' }], default: 'a' }
{ type: 'collection', default: {} }
{ type: 'multiOptions', default: [] }
{ type: 'number', default: 0 }
{ type: 'string', default: '' }
{ type: 'json', default: '{}' }
```

---

### AP-9: Accessing Parameters Outside Item Loop Without Index

**WRONG**: Using index 0 for item-dependent parameters.
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    // WRONG: reads parameter only for first item -- expressions resolve differently per item
    const name = this.getNodeParameter('name', 0) as string;

    for (let i = 0; i < items.length; i++) {
        returnData.push({ json: { name } }); // Same name for ALL items
    }
}
```

**CORRECT**: Read parameters inside the loop with the item index.
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();

    for (let i = 0; i < items.length; i++) {
        // Correct: reads parameter for each item (expressions may differ)
        const name = this.getNodeParameter('name', i) as string;
        returnData.push({ json: { name } });
    }
}
```

**Why**: Parameters can contain expressions like `{{ $json.name }}` that resolve to different values per item. Using a fixed index means all items get the same value.

**Exception**: Resource and operation selectors (with `noDataExpression: true`) are safe to read once at index 0 because they cannot contain expressions.

---

### AP-10: Missing pairedItem in Error Items

**WRONG**: Pushing error items without pairedItem.
```typescript
if (this.continueOnFail()) {
    returnData.push({ json: { error: (error as Error).message } });
    continue;
}
```

**CORRECT**: ALWAYS include pairedItem on error items.
```typescript
if (this.continueOnFail()) {
    returnData.push({
        json: { error: (error as Error).message },
        pairedItem: { item: i },
    });
    continue;
}
```

**Why**: pairedItem connects output items to their source input items. Without it, n8n cannot trace which input item caused the error, breaking data lineage in the UI.

---

### AP-11: Forgetting json Property on Output Items

**WRONG**: Returning items without the json property.
```typescript
returnData.push({ name: 'test', value: 42 }); // WRONG: missing json wrapper
```

**CORRECT**: ALWAYS wrap data in the json property.
```typescript
returnData.push({ json: { name: 'test', value: 42 } }); // Correct
```

**Why**: `INodeExecutionData` requires the `json` property. Data placed directly on the item object is ignored by downstream nodes.

---

### AP-12: Declarative Routing with Wrong URL Expression Syntax

**WRONG**: Using standard JavaScript template literals in URL.
```typescript
routing: {
    request: {
        method: 'GET',
        url: `/users/${parameter.userId}`,  // WRONG: JS template literal
    },
}
```

**CORRECT**: Use n8n expression syntax with `=` prefix.
```typescript
routing: {
    request: {
        method: 'GET',
        url: '=/users/{{$parameter.userId}}',  // Correct: n8n expression
    },
}
```

**Why**: Declarative routing URLs are evaluated by n8n's expression engine, not JavaScript. The `=` prefix indicates an expression, and `{{$parameter.name}}` accesses parameter values.

---

### AP-13: Using loadOptionsMethod Without loadOptionsDependsOn

**WRONG**: Dynamic options that depend on another field but do not declare the dependency.
```typescript
{
    displayName: 'Board',
    name: 'board',
    type: 'options',
    typeOptions: {
        loadOptionsMethod: 'getBoards',
        // Missing loadOptionsDependsOn -- n8n won't reload when project changes
    },
    default: '',
}
```

**CORRECT**: Declare dependencies so options reload when the parent field changes.
```typescript
{
    displayName: 'Board',
    name: 'board',
    type: 'options',
    typeOptions: {
        loadOptionsMethod: 'getBoards',
        loadOptionsDependsOn: ['project'],  // Reload when project changes
    },
    default: '',
}
```

**Why**: Without `loadOptionsDependsOn`, n8n loads options once on first open. When the user changes the parent field, stale/wrong options remain visible.

---

### AP-14: Putting Heavy Content in description Instead of references/

**WRONG**: A SKILL.md that is 800 lines long with every interface definition inline.

**CORRECT**: SKILL.md stays under 500 lines with essential quick references. Detailed interface definitions, complete examples, and anti-patterns go in `references/` files.

**Why**: Claude reads SKILL.md first. A bloated file wastes context window and slows down response. The references/ pattern keeps the main file scannable.
