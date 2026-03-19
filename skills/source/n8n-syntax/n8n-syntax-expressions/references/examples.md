# n8n Expression Examples

## Basic Data Access

### Access Current Item Fields

```
{{ $json.name }}
{{ $json.address.city }}
{{ $json["field-with-dashes"] }}
{{ $json.items[0].name }}
```

### Access Binary Data Properties

```
{{ $binary.data.fileName }}
{{ $binary.data.mimeType }}
{{ $binary.data.fileSize }}
{{ $binary.attachment.fileExtension }}
```

### Access Other Node Output

```
// Linked item (expression context)
{{ $("HTTP Request").item.json.data.id }}

// First item from another node
{{ $("Get Config").first().json.apiEndpoint }}

// Last item
{{ $("Process Data").last().json.summary }}

// Count all items from a node
{{ $("Fetch Records").all().length }}

// Check if a node has executed
{{ $("Optional Step").isExecuted ? $("Optional Step").first().json.result : "skipped" }}
```

### Access All Items with Index

```
{{ $("Get Users").all()[0].json.email }}
{{ $("Get Users").all()[2].json.name }}
```

---

## Conditional Expressions

### Ternary Operator

```
{{ $json.status === "active" ? "Enabled" : "Disabled" }}
{{ $json.age >= 18 ? "Adult" : "Minor" }}
{{ $json.score > 90 ? "A" : $json.score > 80 ? "B" : "C" }}
```

### Null/Empty Safety with $ifEmpty

```
{{ $ifEmpty($json.nickname, $json.fullName) }}
{{ $ifEmpty($json.phone, "No phone provided") }}
{{ $ifEmpty($json.tags, ["untagged"]) }}
```

### Nullish Coalescing and Optional Chaining

```
{{ $json.address?.city ?? "Unknown" }}
{{ $json.metadata?.tags?.[0] ?? "no-tag" }}
```

---

## String Operations

### Template Literals

```
{{ `Hello, ${$json.firstName} ${$json.lastName}!` }}
{{ `Order #${$json.orderId} - Total: $${$json.total.toFixed(2)}` }}
```

### String Methods (n8n Extensions)

```
{{ $json.email.extractDomain() }}
{{ $json.text.removeMarkdown() }}
{{ $json.url.isUrl() ? "valid" : "invalid" }}
{{ $json.name.toSnakeCase() }}
{{ $json.html.base64Encode() }}
{{ $json.password.hash("sha256") }}
{{ $json.query.urlEncode() }}
{{ $json.name.toTitleCase() }}
```

---

## Number Operations

```
{{ $json.price.round(2) }}
{{ $json.value.abs() }}
{{ $json.amount.ceil() }}
{{ $json.score.floor() }}
{{ $json.count.format("en-US") }}
{{ ($json.price * 1.21).round(2) }}
```

---

## Array Operations (n8n Extensions)

```
// Extract specific field from array of objects
{{ $json.users.pluck("email") }}

// Remove duplicates
{{ $json.tags.unique() }}
{{ $json.items.removeDuplicates("id") }}

// Aggregation
{{ $json.scores.sum() }}
{{ $json.scores.average() }}
{{ $json.prices.max() }}
{{ $json.prices.min() }}

// Filtering and transformation
{{ $json.items.compact() }}
{{ $json.data.chunk(10) }}

// Set operations
{{ $json.listA.difference($json.listB) }}
{{ $json.listA.intersection($json.listB) }}
{{ $json.listA.union($json.listB) }}

// Checks
{{ $json.results.isEmpty() ? "No results" : "Has results" }}
{{ $json.items.first() }}
{{ $json.items.last() }}
```

---

## Object Operations (n8n Extensions)

```
{{ $json.data.keys() }}
{{ $json.data.values() }}
{{ $json.data.hasField("email") }}
{{ $json.data.compact() }}
{{ $json.data.removeField("password") }}
{{ $json.params.urlEncode() }}
{{ $json.data.toJsonString() }}
{{ Object.keys($json.data).length }}
```

---

## Date & Time (Luxon DateTime)

### Current Date/Time

```
{{ $now.format("yyyy-MM-dd HH:mm:ss") }}
{{ $now.toISO() }}
{{ $today.toISO() }}
{{ $now.year }}-{{ $now.month }}-{{ $now.day }}
```

### Date Arithmetic

```
{{ $today.minus({days: 7}).toISO() }}
{{ $now.plus({hours: 2}).format("HH:mm") }}
{{ $today.minus({months: 1}).startOf("month").toISO() }}
{{ $now.endOf("day").toISO() }}
```

### Parse Dates

```
{{ DateTime.fromISO("2024-06-23T00:00:00.000Z") }}
{{ DateTime.fromFormat("23-06-2024", "dd-MM-yyyy") }}
{{ $json.dateString.toDateTime() }}
{{ (1719100800000).toDateTime() }}
```

### Date Comparison

```
{{ $now.diff(DateTime.fromISO($json.createdAt), "days").days }}
{{ DateTime.fromISO($json.deadline).diffToNow("hours").hours }}
{{ $now.toRelative() }}
{{ $now.minus({hours: 3}).toRelative() }}
```

### Date Formatting

```
{{ $now.format("dd/MM/yyyy") }}
{{ $now.format("EEEE, MMMM d, yyyy") }}
{{ $now.toLocaleString() }}
{{ $now.monthLong }} {{ $now.year }}
{{ $now.weekdayLong }}
```

### Timezone Handling

```
{{ $now.setZone("America/New_York").format("HH:mm") }}
{{ $now.toUTC().toISO() }}
{{ $now.setZone("Europe/Amsterdam").format("yyyy-MM-dd HH:mm") }}
```

---

## JMESPath Queries

### Basic Field Extraction

```
// Extract names from array of objects
{{ $jmespath($json.people, "[*].name") }}
// Returns: ["Alice", "Bob", "Charlie"]

// Nested field extraction
{{ $jmespath($json.data, "[*].address.city") }}
```

### Slice Projections

```
// First 3 items
{{ $jmespath($json.items, "[:3].name") }}

// Last 2 items
{{ $jmespath($json.items, "[-2:].name") }}
```

### Object Projections

```
// Extract all values from object properties
{{ $jmespath($json.servers, "*.status") }}
// Returns values of status field from all server objects
```

### Filter Expressions

```
// Filter by condition
{{ $jmespath($json.products, "[?price > `100`].name") }}

// Filter with equality
{{ $jmespath($json.users, "[?role == 'admin'].email") }}

// Filter on nested field
{{ $jmespath($json.orders, "[?status == 'pending'].items[*].name") }}
```

### Multiselect

```
// Select multiple fields into arrays
{{ $jmespath($json.people, "[].[firstName, lastName]") }}
// Returns: [["Alice","Smith"],["Bob","Jones"]]

// Select into objects
{{ $jmespath($json.people, "[].{name: firstName, email: emailAddress}") }}
```

### Using JMESPath with Node Output

```
// Filter items from another node
{{ $jmespath($("Code").all(), "[?json.active==`true`].json.id") }}

// Query against all items
{{ $jmespath($input.all(), "[?json.score > `80`].json.name") }}
```

### Flatten and Pipe

```
// Flatten nested arrays
{{ $jmespath($json.data, "[].items[]") }}

// Pipe to get length
{{ $jmespath($json.data, "[*].name | length(@)") }}

// Pipe to sort
{{ $jmespath($json.data, "sort_by([*], &age)") }}
```

---

## Static Workflow Data

### Global Static Data (Shared Across All Nodes)

```js
// In Code node — persists across executions
const staticData = $getWorkflowStaticData('global');

// Read previous value
const lastProcessedId = staticData.lastId || 0;

// Write new value (saved automatically on successful execution)
staticData.lastId = $json.id;
staticData.lastRun = $now.toISO();
```

### Node-Specific Static Data

```js
// Scoped to this specific node instance
const nodeData = $getWorkflowStaticData('node');

// Track processed items to avoid duplicates
nodeData.processedIds = nodeData.processedIds || [];
if (!nodeData.processedIds.includes($json.id)) {
    nodeData.processedIds.push($json.id);
}
```

### Static Data Deduplication Pattern

```js
const staticData = $getWorkflowStaticData('global');
const seen = new Set(staticData.processedIds || []);
const newItems = [];

for (const item of $input.all()) {
    if (!seen.has(item.json.id)) {
        seen.add(item.json.id);
        newItems.push(item);
    }
}

staticData.processedIds = Array.from(seen);
return newItems;
```

---

## IIFE (Immediately Invoked Function Expression)

For multi-statement logic inside expression fields (outside Code node):

### Basic IIFE

```
{{ (function() {
    const price = $json.price;
    const tax = price * 0.21;
    return (price + tax).toFixed(2);
})() }}
```

### IIFE with Conditional Logic

```
{{ (function() {
    const status = $json.status;
    const priority = $json.priority;
    if (status === "urgent" && priority > 5) return "critical";
    if (status === "urgent") return "high";
    if (priority > 5) return "medium";
    return "low";
})() }}
```

### Arrow Function IIFE

```
{{ (() => {
    const items = $input.all();
    return items.map(i => i.json.name).join(", ");
})() }}
```

---

## Paired Items / Item Linking

### Expression Context (Automatic)

```
// Access the linked item from a previous node
{{ $("Fetch Data").item.json.originalValue }}

// This resolves automatically per item during iteration
{{ $("Transform").item.json.processedName }}
```

### Code Node Context (Manual)

```js
// Use itemMatching in Code node
for (let i = 0; i < items.length; i++) {
    const original = $("Webhook").itemMatching(i);
    items[i].json.originalPayload = original.json.body;
}
return items;
```

---

## Execution Metadata

### Custom Data for Logging/Tracking

```js
// Set tracking data
$execution.customData.set("processed_count", String(items.length));
$execution.customData.set("source", "api_sync");
$execution.customData.setAll({
    "started_at": $now.toISO(),
    "workflow_version": "2.1"
});

// Read tracking data in a later node
const count = $execution.customData.get("processed_count");
```

### Execution Mode Branching

```
{{ $execution.mode === "test" ? "https://staging.api.com" : "https://api.com" }}
```

### Resume URL (Wait Node)

```
// Use in email/notification to allow manual continuation
{{ $execution.resumeUrl }}
```

---

## Environment Variables

### Instance Environment

```
{{ $env.API_BASE_URL }}
{{ $env.DEFAULT_TIMEOUT }}
```

### Workflow Variables

```
{{ $vars.companyName }}
{{ $vars.maxRetries }}
{{ $vars.defaultLocale }}
```

### Secrets (Expressions Only, NOT Code Node)

```
{{ $secrets.vault.apiKey }}
{{ $secrets.provider.databasePassword }}
```
