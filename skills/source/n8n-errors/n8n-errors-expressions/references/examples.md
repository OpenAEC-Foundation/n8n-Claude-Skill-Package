# Common Expression Errors with Fixes

> Before/after code examples for the most frequent n8n expression errors.
> Each example shows the broken code and the corrected version.

---

## 1. Undefined $json Reference

**Scenario:** Accessing a field that does not exist on the current item.

```js
// BROKEN — field "customer_name" does not exist, actual field is "customerName"
{{ $json.customer_name }}
// Returns: undefined

// FIXED — use correct field name
{{ $json.customerName }}

// DEFENSIVE — provide fallback for optional fields
{{ $ifEmpty($json.customerName, "Unknown Customer") }}
```

**Debugging tip:** Use `{{ Object.keys($json) }}` to list all available field names.

---

## 2. Nested Property Access on Undefined Parent

**Scenario:** Accessing `$json.address.city` when `address` is null or undefined.

```js
// BROKEN — throws TypeError if address is null/undefined
{{ $json.address.city }}

// FIXED — optional chaining
{{ $json.address?.city }}

// FIXED — with fallback
{{ $ifEmpty($json.address?.city, "No City") }}
```

---

## 3. $itemIndex in Code Node

**Scenario:** Using `$itemIndex` inside a Code node, which is not available there.

```js
// BROKEN — ReferenceError: $itemIndex is not defined
const index = $itemIndex;
return { json: { position: index } };

// FIXED — Run Once for All Items mode, use loop index
const results = [];
for (let i = 0; i < items.length; i++) {
  results.push({
    json: { position: i, value: items[i].json.name }
  });
}
return results;

// FIXED — Run Once for Each Item mode, use items array
return {
  json: { position: items.indexOf($input.item), value: $input.item.json.name }
};
```

---

## 4. $secrets in Code Node

**Scenario:** Trying to access external secrets inside a Code node.

```js
// BROKEN — ReferenceError: $secrets is not defined
const apiKey = $secrets.myProvider.apiKey;

// FIXED — use environment variables instead
const apiKey = $env.MY_API_KEY;

// ALTERNATIVE — pass secret via Set node before Code node
// In Set node expression field: {{ $secrets.myProvider.apiKey }}
// Then in Code node:
const apiKey = $json.apiKey; // received from Set node
```

---

## 5. JMESPath Parameter Order

**Scenario:** Swapping the parameter order in `$jmespath()`.

```js
// BROKEN — parameters in wrong order (JMESPath spec order)
{{ $jmespath("[*].name", $json.data) }}
// Returns: unexpected result or error

// FIXED — n8n order: object first, search string second
{{ $jmespath($json.data, "[*].name") }}
```

```js
// BROKEN — filtering with swapped params
{{ $jmespath("[?age > `30`].name", $json.people) }}

// FIXED
{{ $jmespath($json.people, "[?age > `30`].name") }}
```

---

## 6. Paired Item Error

**Scenario:** Using `$("<Node>").item` when item linking is broken.

```js
// BROKEN — "Paired item information is missing"
// Occurs after nodes that change item count (aggregation, split, filter)
{{ $("Customer Lookup").item.json.email }}

// FIXED — use explicit first item access
{{ $("Customer Lookup").first().json.email }}

// FIXED — use all items with index
{{ $("Customer Lookup").all()[0].json.email }}

// FIXED — in Code node, use itemMatching
const matchedItem = $("Customer Lookup").itemMatching(0);
return { json: { email: matchedItem.json.email } };
```

---

## 7. $response Outside HTTP Request Node

**Scenario:** Trying to use `$response` in a node other than HTTP Request.

```js
// BROKEN — in a Set node after HTTP Request
{{ $response.statusCode }}
// ReferenceError: $response is not defined

// FIXED — HTTP Request outputs response as $json for downstream nodes
{{ $json.statusCode }}

// For headers: configure HTTP Request to include response headers in output
// Then access via $json.headers
```

---

## 8. String vs Number Type Mismatch

**Scenario:** API returns a number as a string, causing comparison failure.

```js
// BROKEN — string "100" compared to number, unexpected behavior
{{ $json.price > 50 }}
// May return true because "100" > 50 uses type coercion unpredictably

// FIXED — explicit type conversion
{{ Number($json.price) > 50 }}

// FIXED — for integer parsing
{{ parseInt($json.price, 10) > 50 }}

// FIXED — for decimal parsing
{{ parseFloat($json.price) > 50.00 }}
```

---

## 9. Empty Expression Result

**Scenario:** Expression evaluates but returns empty, null, or undefined.

```js
// BROKEN — field exists but contains null
{{ $json.middleName }}
// Returns: null (renders as empty in most nodes)

// FIXED — provide default value
{{ $ifEmpty($json.middleName, "N/A") }}

// FIXED — conditional with ternary
{{ $json.middleName ? $json.middleName : "Not Provided" }}
```

```js
// BROKEN — array field is empty
{{ $json.tags.first() }}
// Returns: undefined (no first element)

// FIXED — check array before accessing
{{ $json.tags?.isNotEmpty() ? $json.tags.first() : "no-tag" }}
```

---

## 10. Static Data Not Available During Testing

**Scenario:** `$getWorkflowStaticData()` returns empty during manual test.

```js
// BROKEN — always empty during manual "Test Workflow" execution
const staticData = $getWorkflowStaticData('global');
const lastRun = staticData.lastExecution;
// lastRun is undefined during testing

// FIXED — add fallback for testing scenario
const staticData = $getWorkflowStaticData('global');
const lastRun = staticData.lastExecution || new Date(0).toISOString();
// Uses epoch as default when no static data exists

// IMPORTANT: Static data is ONLY populated during production execution
// (trigger/webhook activation). NEVER rely on it during manual tests.
```

---

## 11. Python Dot Notation Error

**Scenario:** Using JavaScript-style dot notation in Python Code node.

```python
# BROKEN — AttributeError in Python
name = item.json.name

# FIXED — ALWAYS use bracket notation in Python
name = item["json"]["name"]

# BROKEN — accessing nested data
city = item.json.address.city

# FIXED
city = item["json"]["address"]["city"]

# DEFENSIVE — with default value
city = item["json"].get("address", {}).get("city", "Unknown")
```

---

## 12. Date Comparison Failure

**Scenario:** Comparing date strings instead of DateTime objects.

```js
// BROKEN — string comparison gives wrong results
{{ $json.createdAt > $json.updatedAt }}
// "2024-01-15" > "2024-02-01" returns true (string comparison, "1" > "0")

// FIXED — convert to DateTime then compare
{{ $json.createdAt.toDateTime() > $json.updatedAt.toDateTime() }}

// FIXED — check if date is within range
{{ $json.eventDate.toDateTime().isBetween(
  $today.minus({days: 7}),
  $today.plus({days: 7})
) }}
```

---

## 13. Wrong Extension Method for Type

**Scenario:** Calling `.average()` on a non-array value.

```js
// BROKEN — $json.score is a single number, not an array
{{ $json.score.average() }}
// TypeError: $json.score.average is not a function

// FIXED — .average() is for arrays only
{{ $json.scores.average() }}

// If you have a single value, no averaging needed
{{ $json.score }}
```

```js
// BROKEN — .extractEmail() on a number
{{ $json.phone.extractEmail() }}
// TypeError

// FIXED — .extractEmail() is for strings only
{{ $json.emailField.extractEmail() }}
```

---

## 14. Object Displayed as [object Object]

**Scenario:** Using an object in a string context.

```js
// BROKEN — concatenating object into string
{{ "Data: " + $json.metadata }}
// Returns: "Data: [object Object]"

// FIXED — convert to JSON string
{{ "Data: " + $json.metadata.toJsonString() }}

// FIXED — access specific property instead
{{ "Data: " + $json.metadata.type }}
```

---

## 15. IIFE Syntax for Complex Logic

**Scenario:** Need multi-statement logic in an expression field.

```js
// BROKEN — multiple statements in expression without IIFE
{{ const x = $json.price * 1.21; return x; }}
// SyntaxError

// FIXED — wrap in IIFE (Immediately Invoked Function Expression)
{{ (function() {
  const price = $json.price || 0;
  const tax = price * 0.21;
  return (price + tax).toFixed(2);
})() }}
```

---

## 16. Loop Over Items Context Variable

**Scenario:** Using `$("<Node>").context` outside a Loop Over Items node.

```js
// BROKEN — used in a node that is not inside a Loop Over Items
{{ $("Loop Over Items").context.currentRunIndex }}
// Returns: undefined

// FIXED — ONLY use .context inside the Loop Over Items sub-workflow
// Verify the node is connected within the Loop Over Items execution path
```
