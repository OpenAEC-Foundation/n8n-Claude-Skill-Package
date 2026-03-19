# Custom Node Development: Anti-Patterns

## AP-1: Pointing n8n Field to Source Files

**WRONG:**
```json
{
    "n8n": {
        "nodes": ["nodes/MyNode/MyNode.node.ts"],
        "credentials": ["credentials/MyApi.credentials.ts"]
    }
}
```

**CORRECT:**
```json
{
    "n8n": {
        "nodes": ["dist/nodes/MyNode/MyNode.node.js"],
        "credentials": ["dist/credentials/MyApi.credentials.js"]
    }
}
```

**Why:** n8n loads compiled JavaScript at runtime. Pointing to `.ts` source files causes a silent failure — the node simply does not appear in the editor with no error message.

---

## AP-2: Missing n8n-community-node-package Keyword

**WRONG:**
```json
{
    "keywords": ["n8n", "automation", "workflow"]
}
```

**CORRECT:**
```json
{
    "keywords": ["n8n-community-node-package"]
}
```

**Why:** n8n's community node installer searches npm specifically for the `n8n-community-node-package` keyword. Without it, users CANNOT install your package through the n8n UI (Settings > Community Nodes).

---

## AP-3: n8n-workflow as Regular Dependency

**WRONG:**
```json
{
    "dependencies": {
        "n8n-workflow": "^1.0.0"
    }
}
```

**CORRECT:**
```json
{
    "peerDependencies": {
        "n8n-workflow": "*"
    }
}
```

**Why:** `n8n-workflow` is provided by the n8n runtime. Including it as a regular dependency causes version conflicts, duplicate type definitions, and increased package size. ALWAYS use `peerDependencies` with `"*"` to accept whatever version n8n provides.

---

## AP-4: Not Iterating Over Input Items

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const userId = this.getNodeParameter('userId', 0) as string;
    // Only processes the first item!
    const response = await this.helpers.httpRequest({ /* ... */ });
    return [[{ json: response }]];
}
```

**CORRECT:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
        const userId = this.getNodeParameter('userId', i) as string;
        const response = await this.helpers.httpRequest({ /* ... */ });
        returnData.push({ json: response as IDataObject });
    }

    return [returnData];
}
```

**Why:** n8n processes data as arrays of items. If a previous node outputs 10 items, your node receives all 10. ALWAYS iterate over `this.getInputData()` and pass the correct item index to `getNodeParameter()` — expressions like `{{ $json.id }}` resolve differently per item.

---

## AP-5: Missing continueOnFail Handling

**WRONG:**
```typescript
for (let i = 0; i < items.length; i++) {
    const response = await this.helpers.httpRequest({ /* ... */ });
    returnData.push({ json: response });
}
```

**CORRECT:**
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

**Why:** Users expect the "Continue On Fail" setting to work. Without this pattern, any API error crashes the entire workflow. ALWAYS wrap item processing in try/catch and check `this.continueOnFail()`.

---

## AP-6: Missing pairedItem in Error Output

**WRONG:**
```typescript
if (this.continueOnFail()) {
    returnData.push({ json: { error: error.message } });
    continue;
}
```

**CORRECT:**
```typescript
if (this.continueOnFail()) {
    returnData.push({
        json: { error: (error as Error).message },
        pairedItem: { item: i },
    });
    continue;
}
```

**Why:** `pairedItem` links the output item back to its input item. Without it, n8n cannot track data lineage and the "Output Mapping" feature breaks. ALWAYS include `pairedItem: { item: i }` when pushing items to returnData.

---

## AP-7: Hardcoding Connection Types

**WRONG:**
```typescript
description: INodeTypeDescription = {
    inputs: ['main'],
    outputs: ['main'],
};
```

**CORRECT:**
```typescript
import { NodeConnectionTypes } from 'n8n-workflow';

description: INodeTypeDescription = {
    inputs: [NodeConnectionTypes.Main],
    outputs: [NodeConnectionTypes.Main],
};
```

**Why:** Hardcoding strings bypasses TypeScript type checking and can break if n8n changes internal type values. ALWAYS use `NodeConnectionTypes` constants from `n8n-workflow`.

---

## AP-8: Wrong Package Name Prefix

**WRONG:**
```json
{
    "name": "my-n8n-nodes"
}
```

**CORRECT:**
```json
{
    "name": "n8n-nodes-myservice"
}
```

**Why:** n8n requires community node packages to start with `n8n-nodes-`. This convention is enforced by the community node installer and npm search. Packages without this prefix are not recognized.

---

## AP-9: Declaring execute() on Declarative Nodes

**WRONG:**
```typescript
export class MyApiNode implements INodeType {
    description: INodeTypeDescription = {
        requestDefaults: { baseURL: 'https://api.example.com' },
        properties: [
            { /* with routing config */ },
        ],
    };

    async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
        // Manually making HTTP requests
    }
}
```

**CORRECT:**
```typescript
export class MyApiNode implements INodeType {
    description: INodeTypeDescription = {
        requestDefaults: { baseURL: 'https://api.example.com' },
        properties: [
            { /* with routing config */ },
        ],
    };

    // No execute() method — n8n handles requests via routing
}
```

**Why:** When both `execute()` and `requestDefaults`/routing are present, `execute()` takes precedence and the routing config is ignored. Choose one approach: programmatic (with `execute()`) OR declarative (with routing). NEVER mix them.

---

## AP-10: Forgetting to Build Before Link/Publish

**WRONG:**
```bash
npm link            # Links source files, node not found
npm publish         # Publishes without compiled dist/
```

**CORRECT:**
```bash
npm run build       # Compiles TypeScript to dist/
npm link            # Links compiled package
# or
npm run build
npm publish         # Publishes with compiled dist/
```

**Why:** The `n8n` field in package.json points to `dist/` files. Without building first, those files do not exist. The node silently fails to load.

---

## AP-11: Not Including dist in files Field

**WRONG:**
```json
{
    "files": ["nodes", "credentials"]
}
```

**CORRECT:**
```json
{
    "files": ["dist"]
}
```

**Why:** npm only publishes files listed in the `files` field. Since `n8n.nodes` and `n8n.credentials` point to `dist/`, you MUST include `dist` in the `files` array. Otherwise the published package is empty.

---

## AP-12: Returning Wrong Array Shape from execute()

**WRONG:**
```typescript
// Returns flat array — TypeScript error
return returnData;

// Returns 3D array — runtime error
return [[returnData]];
```

**CORRECT:**
```typescript
// Single output: wrap in one array
return [returnData];

// Two outputs (like IF node): one array per output
return [trueItems, falseItems];
```

**Why:** `execute()` returns `INodeExecutionData[][]` — a 2D array where the first dimension is the output index. For a single-output node, ALWAYS return `[returnData]`. Each additional output adds another element to the outer array.

---

## AP-13: Using Sensitive Fields Without Password Flag

**WRONG:**
```typescript
{
    displayName: 'API Key',
    name: 'apiKey',
    type: 'string',
    default: '',
}
```

**CORRECT:**
```typescript
{
    displayName: 'API Key',
    name: 'apiKey',
    type: 'string',
    typeOptions: { password: true },
    default: '',
}
```

**Why:** Without `password: true`, the API key is displayed as plain text in the n8n UI and may appear in logs. ALWAYS use `typeOptions: { password: true }` for API keys, tokens, passwords, and any other sensitive credential fields.

---

## AP-14: Not Setting noDataExpression on Resource/Operation

**WRONG:**
```typescript
{
    displayName: 'Resource',
    name: 'resource',
    type: 'options',
    options: [/* ... */],
    default: 'user',
}
```

**CORRECT:**
```typescript
{
    displayName: 'Resource',
    name: 'resource',
    type: 'options',
    noDataExpression: true,
    options: [/* ... */],
    default: 'user',
}
```

**Why:** Resource and Operation selectors should be static configuration, not dynamic expressions. Without `noDataExpression: true`, the UI shows an expression toggle that confuses users. ALWAYS set `noDataExpression: true` on Resource and Operation properties.
