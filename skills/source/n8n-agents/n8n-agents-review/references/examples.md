# n8n Review Examples

> Good code, bad code, and fixes for each review area.

---

## 1. Workflow JSON — Node Name Uniqueness

### BAD: Duplicate node names

```json
{
  "nodes": [
    { "id": "1", "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest", "typeVersion": 4.2, "position": [250, 300], "parameters": {} },
    { "id": "2", "name": "HTTP Request", "type": "n8n-nodes-base.httpRequest", "typeVersion": 4.2, "position": [450, 300], "parameters": {} }
  ],
  "connections": {
    "HTTP Request": { "main": [[{ "node": "HTTP Request", "type": "main", "index": 0 }]] }
  }
}
```

**Problem**: Both nodes named "HTTP Request" — connections cannot distinguish between them.

### GOOD: Unique node names

```json
{
  "nodes": [
    { "id": "1", "name": "Fetch Users", "type": "n8n-nodes-base.httpRequest", "typeVersion": 4.2, "position": [250, 300], "parameters": {} },
    { "id": "2", "name": "Fetch Orders", "type": "n8n-nodes-base.httpRequest", "typeVersion": 4.2, "position": [450, 300], "parameters": {} }
  ],
  "connections": {
    "Fetch Users": { "main": [[{ "node": "Fetch Orders", "type": "main", "index": 0 }]] }
  }
}
```

---

## 2. Connection Wiring — IF Node Outputs

### BAD: IF node with single output array

```json
{
  "IF": {
    "main": [
      [{ "node": "Process Data", "type": "main", "index": 0 }]
    ]
  }
}
```

**Problem**: IF node MUST have 2 output arrays (true branch at index 0, false branch at index 1).

### GOOD: IF node with both branches

```json
{
  "IF": {
    "main": [
      [{ "node": "Process Active", "type": "main", "index": 0 }],
      [{ "node": "Process Inactive", "type": "main", "index": 0 }]
    ]
  }
}
```

---

## 3. Connection Wiring — Stale Reference

### BAD: Connection to deleted node

```json
{
  "nodes": [
    { "id": "1", "name": "Trigger", "type": "n8n-nodes-base.scheduleTrigger", "typeVersion": 1.2, "position": [250, 300], "parameters": {} },
    { "id": "2", "name": "Transform", "type": "n8n-nodes-base.set", "typeVersion": 3, "position": [450, 300], "parameters": {} }
  ],
  "connections": {
    "Trigger": { "main": [[{ "node": "Deleted Node", "type": "main", "index": 0 }]] }
  }
}
```

**Problem**: "Deleted Node" does not exist in `nodes[]`.

### FIX: Update connection to existing node

```json
{
  "connections": {
    "Trigger": { "main": [[{ "node": "Transform", "type": "main", "index": 0 }]] }
  }
}
```

---

## 4. Expression — Wrong JMESPath Argument Order

### BAD: Reversed arguments

```javascript
{{ $jmespath("[*].name", $json.items) }}
```

**Problem**: n8n's `$jmespath` takes `(object, searchString)` — object FIRST.

### GOOD: Correct argument order

```javascript
{{ $jmespath($json.items, "[*].name") }}
```

---

## 5. Expression — Using $itemIndex in Code Node

### BAD: Restricted variable in Code node

```javascript
// Code node — Run Once for All Items
for (let i = 0; i < items.length; i++) {
  items[i].json.position = $itemIndex; // ERROR: $itemIndex is not defined
}
return items;
```

### GOOD: Use loop variable instead

```javascript
// Code node — Run Once for All Items
for (let i = 0; i < items.length; i++) {
  items[i].json.position = i;
}
return items;
```

---

## 6. Expression — Using new Date()

### BAD: JavaScript Date constructor

```javascript
{{ new Date().toISOString() }}
```

**Problem**: `new Date()` ignores workflow timezone settings.

### GOOD: Luxon DateTime

```javascript
{{ $now.toISO() }}
```

Or for formatted output:

```javascript
{{ $now.format("yyyy-MM-dd HH:mm:ss") }}
```

---

## 7. Custom Node — Wrong Execute Return Type

### BAD: Single array return

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  for (let i = 0; i < items.length; i++) {
    returnData.push({ json: { processed: true } });
  }

  return returnData; // ERROR: returns INodeExecutionData[], not INodeExecutionData[][]
}
```

### GOOD: Double-wrapped array return

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  for (let i = 0; i < items.length; i++) {
    returnData.push({ json: { processed: true } });
  }

  return [returnData]; // Correct: wrapped in outer array
}
```

---

## 8. Custom Node — Missing continueOnFail

### BAD: No error handling in item loop

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  for (let i = 0; i < items.length; i++) {
    const userId = this.getNodeParameter('userId', i) as string;
    const response = await this.helpers.httpRequest({
      method: 'GET',
      url: `https://api.example.com/users/${userId}`,
    });
    returnData.push({ json: response as IDataObject });
  }

  return [returnData];
}
```

**Problem**: One failed HTTP request stops the entire batch.

### GOOD: continueOnFail with pairedItem

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  for (let i = 0; i < items.length; i++) {
    try {
      const userId = this.getNodeParameter('userId', i) as string;
      const response = await this.helpers.httpRequest({
        method: 'GET',
        url: `https://api.example.com/users/${userId}`,
      });
      returnData.push({ json: response as IDataObject, pairedItem: { item: i } });
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

---

## 9. Custom Node — Mutating Input Data

### BAD: Direct mutation of input items

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();

  for (let i = 0; i < items.length; i++) {
    items[i].json.newField = 'value'; // Mutates shared reference
  }

  return [items];
}
```

**Problem**: Modifying `getInputData()` directly can corrupt data for other nodes.

### GOOD: Create new items

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  for (let i = 0; i < items.length; i++) {
    returnData.push({
      json: { ...items[i].json, newField: 'value' },
      pairedItem: { item: i },
    });
  }

  return [returnData];
}
```

---

## 10. Credential — Missing Authenticate Method

### BAD: No authenticate property

```typescript
export class MyApi implements ICredentialType {
  name = 'myApi';
  displayName = 'My API';
  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];
  // Missing authenticate — credentials never injected into requests
}
```

### GOOD: Complete credential with authenticate and test

```typescript
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
      baseURL: 'https://api.example.com',
      url: '/me',
    },
  };
}
```

---

## 11. Code Node — Wrong Return Format

### BAD: Missing json wrapper

```javascript
// Code node — Run Once for All Items
return items.map(item => ({
  name: item.json.name,
  email: item.json.email,
}));
```

**Problem**: Each item must have a `json` key.

### GOOD: Proper json wrapper

```javascript
// Code node — Run Once for All Items
return items.map(item => ({
  json: {
    name: item.json.name,
    email: item.json.email,
  },
}));
```

---

## 12. Code Node — Python Dot Notation

### BAD: Dot notation in Python

```python
# Code node — Python
for item in _items:
    name = item.json.name  # AttributeError
return _items
```

### GOOD: Bracket notation in Python

```python
# Code node — Python
for item in _items:
    name = item["json"]["name"]
return _items
```

---

## 13. Deployment — Missing Encryption Key

### BAD: No explicit encryption key

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_PORT=5678
      # No N8N_ENCRYPTION_KEY set — auto-generated
```

**Problem**: If the container is rebuilt, a new key is generated and all existing credentials become undecryptable.

### GOOD: Explicit encryption key

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_PORT=5678
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_RUNNERS_ENABLED=true
      - NODE_ENV=production
```

---

## 14. Deployment — SQLite with Queue Mode

### BAD: SQLite in queue mode

```yaml
environment:
  - EXECUTIONS_MODE=queue
  - DB_TYPE=sqlite
  - QUEUE_BULL_REDIS_HOST=redis
```

**Problem**: SQLite does NOT support queue mode — it cannot handle concurrent access from multiple workers.

### GOOD: PostgreSQL with queue mode

```yaml
environment:
  - EXECUTIONS_MODE=queue
  - DB_TYPE=postgresdb
  - DB_POSTGRESDB_HOST=postgres
  - DB_POSTGRESDB_DATABASE=n8n
  - DB_POSTGRESDB_USER=n8n
  - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
  - QUEUE_BULL_REDIS_HOST=redis
  - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
```

---

## 15. Security — Hardcoded API Key

### BAD: API key in node parameters

```json
{
  "name": "HTTP Request",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "url": "https://api.example.com/data",
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "Bearer sk-1234567890abcdef" }
      ]
    }
  }
}
```

**Problem**: API key is visible in workflow JSON, logs, and execution history.

### GOOD: Credential reference

```json
{
  "name": "HTTP Request",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "url": "https://api.example.com/data",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth"
  },
  "credentials": {
    "httpHeaderAuth": {
      "id": "cred_abc123",
      "name": "Example API Key"
    }
  }
}
```

---

## 16. Trigger Node — Missing Manual Mode

### BAD: No manual trigger function

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
  const callback = () => {
    this.emit([[{ json: { event: 'data' } }]]);
  };

  // Only registers for production — manual "Test" hangs forever
  someService.on('event', callback);

  return {
    closeFunction: async () => {
      someService.off('event', callback);
    },
  };
}
```

### GOOD: Manual mode handled

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
  const callback = () => {
    this.emit([[{ json: { event: 'data' } }]]);
  };

  if (this.getMode() === 'manual') {
    const manualTriggerFunction = async () => {
      callback();
    };
    return { manualTriggerFunction };
  }

  someService.on('event', callback);

  return {
    closeFunction: async () => {
      someService.off('event', callback);
    },
  };
}
```

---

## 17. Package.json — Wrong File Paths

### BAD: Pointing to TypeScript source files

```json
{
  "n8n": {
    "nodes": ["nodes/MyNode/MyNode.node.ts"],
    "credentials": ["credentials/MyApi.credentials.ts"]
  }
}
```

**Problem**: n8n loads compiled JavaScript — TypeScript files are not recognized.

### GOOD: Pointing to compiled dist files

```json
{
  "n8n": {
    "nodes": ["dist/nodes/MyNode/MyNode.node.js"],
    "credentials": ["dist/credentials/MyApi.credentials.js"]
  }
}
```
