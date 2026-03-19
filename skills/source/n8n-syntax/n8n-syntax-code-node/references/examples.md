# Code Node — JavaScript & Python Examples

## Item Manipulation

### Transform All Items (All-Items Mode)

```javascript
// JavaScript — add computed field to every item
return items.map(item => ({
  json: {
    ...item.json,
    fullName: `${item.json.firstName} ${item.json.lastName}`,
    processedAt: $now.toISO()
  }
}));
```

```python
# Python — add computed field to every item
result = []
for item in _items:
    data = item["json"]
    result.append({
        "json": {
            **data,
            "fullName": f"{data['firstName']} {data['lastName']}",
            "processedAt": str(_now)
        }
    })
return result
```

### Transform Single Item (Each-Item Mode)

```javascript
// JavaScript — enrich current item
const data = $input.item.json;
return {
  json: {
    ...data,
    fullName: `${data.firstName} ${data.lastName}`,
    processedAt: $now.toISO()
  }
};
```

```python
# Python — enrich current item
data = _item["json"]
return {
    "json": {
        **data,
        "fullName": f"{data['firstName']} {data['lastName']}",
        "processedAt": str(_now)
    }
}
```

### Filter Items

```javascript
// JavaScript — keep only active users (all-items mode)
return items
  .filter(item => item.json.status === 'active')
  .map(item => ({ json: item.json }));
```

```python
# Python — keep only active users (all-items mode)
return [{"json": item["json"]} for item in _items if item["json"]["status"] == "active"]
```

### Split One Item into Multiple

```javascript
// JavaScript — split comma-separated tags into individual items
const results = [];
for (const item of items) {
  const tags = item.json.tags.split(',');
  for (const tag of tags) {
    results.push({
      json: { ...item.json, tag: tag.trim() }
    });
  }
}
return results;
```

### Merge Multiple Items into One

```javascript
// JavaScript — combine all items into a single summary
const names = items.map(item => item.json.name);
const total = items.reduce((sum, item) => sum + item.json.amount, 0);
return [{
  json: {
    count: items.length,
    names: names,
    totalAmount: total
  }
}];
```

## Aggregation Patterns

### Group By Field

```javascript
// JavaScript — group items by category
const groups = {};
for (const item of items) {
  const key = item.json.category;
  if (!groups[key]) groups[key] = [];
  groups[key].push(item.json);
}

return Object.entries(groups).map(([category, groupItems]) => ({
  json: {
    category,
    count: groupItems.length,
    items: groupItems
  }
}));
```

```python
# Python — group items by category
groups = {}
for item in _items:
    key = item["json"]["category"]
    if key not in groups:
        groups[key] = []
    groups[key].append(item["json"])

return [
    {"json": {"category": cat, "count": len(grp), "items": grp}}
    for cat, grp in groups.items()
]
```

### Deduplicate by Field

```javascript
// JavaScript — remove duplicate items by email
const seen = new Set();
const unique = [];
for (const item of items) {
  const email = item.json.email;
  if (!seen.has(email)) {
    seen.add(email);
    unique.push({ json: item.json });
  }
}
return unique;
```

### Running Totals

```javascript
// JavaScript — add cumulative sum to each item
let runningTotal = 0;
return items.map(item => {
  runningTotal += item.json.amount;
  return {
    json: {
      ...item.json,
      runningTotal
    }
  };
});
```

## Binary Data Handling

### Read Binary File Content

```javascript
// JavaScript — read binary data as string (all-items mode)
const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');
const content = buffer.toString('utf-8');
return [{
  json: {
    content,
    size: buffer.length
  }
}];
```

### Process CSV from Binary

```javascript
// JavaScript — parse CSV binary into structured items
const buffer = await this.helpers.getBinaryDataBuffer(0, 'data');
const csv = buffer.toString('utf-8');
const lines = csv.split('\n');
const headers = lines[0].split(',').map(h => h.trim());

return lines.slice(1).filter(line => line.trim()).map(line => {
  const values = line.split(',');
  const obj = {};
  headers.forEach((header, i) => {
    obj[header] = values[i]?.trim() || '';
  });
  return { json: obj };
});
```

### Create Binary Output

```javascript
// JavaScript — generate a text file as binary output
const content = items.map(item =>
  `${item.json.name}: ${item.json.value}`
).join('\n');

return [{
  json: { fileName: 'report.txt', lineCount: items.length },
  binary: {
    data: await this.helpers.prepareBinaryData(
      Buffer.from(content, 'utf-8'),
      'report.txt',
      'text/plain'
    )
  }
}];
```

### Pass Through Binary Data

```javascript
// JavaScript — modify json but keep binary intact
return items.map(item => ({
  json: { ...item.json, processed: true },
  binary: item.binary
}));
```

## Cross-Node Data Access

### Read Data from Previous Nodes

```javascript
// JavaScript — access output from a specific named node
const httpResults = $("HTTP Request").all();
const firstResult = $("HTTP Request").first().json;

return [{
  json: {
    totalResults: httpResults.length,
    firstUrl: firstResult.url
  }
}];
```

### Item Matching (Tracing)

```javascript
// JavaScript — trace back to matching item in another node
const results = [];
for (let i = 0; i < items.length; i++) {
  const original = $("Spreadsheet").itemMatching(i);
  results.push({
    json: {
      processed: items[i].json.output,
      originalId: original.json.id
    }
  });
}
return results;
```

## JMESPath Queries

```javascript
// JavaScript — extract nested data with JMESPath
const allNames = $jmespath(items.map(i => i.json), "[*].name");
const filtered = $jmespath(items.map(i => i.json), "[?age > `18`].name");
return [{
  json: { names: allNames, adults: filtered }
}];
```

```python
# Python — JMESPath with underscore prefix
all_names = _jmespath([i["json"] for i in _items], "[*].name")
filtered = _jmespath([i["json"] for i in _items], "[?age > `18`].name")
return [{"json": {"names": all_names, "adults": filtered}}]
```

## Execution Metadata & Static Data

### Custom Execution Data

```javascript
// JavaScript — tag execution with metadata for monitoring
$execution.customData.set("processedItems", items.length);
$execution.customData.set("source", $prevNode.name);

return items.map(item => ({ json: item.json }));
```

### Workflow Static Data (Persistent)

```javascript
// JavaScript — track processed IDs across executions
const staticData = $getWorkflowStaticData('global');
staticData.processedIds = staticData.processedIds || [];

const newItems = items.filter(item =>
  !staticData.processedIds.includes(item.json.id)
);

// Update tracking list (keep last 1000)
const newIds = newItems.map(item => item.json.id);
staticData.processedIds = [...staticData.processedIds, ...newIds].slice(-1000);

return newItems.map(item => ({ json: item.json }));
```

## Error Handling Patterns

### Try-Catch with Graceful Degradation

```javascript
// JavaScript — handle errors per item (all-items mode)
const results = [];
for (const item of items) {
  try {
    const value = JSON.parse(item.json.rawData);
    results.push({ json: { parsed: value, error: null } });
  } catch (error) {
    results.push({ json: { parsed: null, error: error.message } });
  }
}
return results;
```

### Conditional Empty Check

```javascript
// JavaScript — handle empty input gracefully
if (items.length === 0 || Object.keys(items[0].json).length === 0) {
  return [{ json: { results: 0, message: "No input data" } }];
}
return items.map(item => ({ json: item.json }));
```

## Console Logging (Debug)

```javascript
// JavaScript — log to browser developer console
console.log("Item count:", items.length);
console.log("First item:", JSON.stringify(items[0].json));
// Open browser DevTools (F12) to see output
```
