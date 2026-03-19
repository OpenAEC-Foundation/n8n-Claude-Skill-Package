# Expression Error Types & Variable Availability

> Reference for n8n-errors-expressions skill.
> Covers all expression error categories and detailed variable availability rules.

---

## Expression Error Categories

### 1. Reference Errors

Reference errors occur when an expression uses a variable or function that does not exist in the current context.

| Error Pattern | Trigger Condition | Resolution |
|---------------|-------------------|------------|
| `ReferenceError: $itemIndex is not defined` | Used `$itemIndex` inside Code node | ALWAYS use loop index variable or `items.indexOf(item)` |
| `ReferenceError: $secrets is not defined` | Used `$secrets` inside Code node | ALWAYS use `$env.SECRET_NAME` or pass secret via Set node |
| `ReferenceError: $response is not defined` | Used `$response` outside HTTP Request node | `$response` is ONLY available in HTTP Request node fields |
| `ReferenceError: $pageCount is not defined` | Used `$pageCount` outside HTTP Request pagination | `$pageCount` is ONLY available in HTTP Request node |
| `ReferenceError: X is not defined` | Misspelled variable name or used non-existent variable | Check spelling; verify variable exists in current context |

### 2. Type Errors

Type errors occur when an operation is performed on an incompatible data type.

| Error Pattern | Trigger Condition | Resolution |
|---------------|-------------------|------------|
| `TypeError: Cannot read properties of undefined (reading 'X')` | Accessing nested property where parent is undefined | Use optional chaining: `$json.parent?.child` |
| `TypeError: Cannot read properties of null (reading 'X')` | Accessing property on null value | Add null check or use `$ifEmpty()` |
| `TypeError: X is not a function` | Calling method on wrong type (e.g., `.extractEmail()` on number) | Verify data type before calling extension methods |
| `TypeError: X.toDateTime is not a function` | Calling `.toDateTime()` on non-string, non-number type | Ensure value is string or number before conversion |
| `TypeError: Cannot convert undefined or null to object` | Using `Object.keys()` or `Object.values()` on undefined | Check value exists before using Object methods |

### 3. Syntax Errors

Syntax errors occur when the expression contains invalid JavaScript syntax.

| Error Pattern | Trigger Condition | Resolution |
|---------------|-------------------|------------|
| `SyntaxError: Unexpected token` | Malformed expression (missing brackets, quotes) | Check matching `{{ }}` and balanced brackets/quotes |
| `SyntaxError: Invalid left-hand side in assignment` | Using `=` assignment inside expression | Expressions are read-only; use Code node for assignments |
| `SyntaxError: Unexpected end of input` | Incomplete expression | Ensure all parentheses, brackets, and quotes are closed |

### 4. Paired Item Errors

Paired item errors occur when n8n cannot trace the link between the current item and a referenced node's output.

| Error Pattern | Trigger Condition | Resolution |
|---------------|-------------------|------------|
| `Paired item information is missing` | Item link chain broken by intermediate node | Use `.first()`, `.all()`, or `.itemMatching()` |
| `$("<Node>").item` returns undefined | No matching paired item found | Use explicit access: `.first()` or `.all()[index]` |
| `$("<Node>").item` returns wrong data | Multiple items with ambiguous linking | Use `.itemMatching(currentIndex)` for precise matching |

### 5. Empty/Null Result Errors

These are not runtime errors but produce unexpected empty or null values.

| Symptom | Trigger Condition | Resolution |
|---------|-------------------|------------|
| Expression returns empty string | Field value is `""`, `null`, or `undefined` | Use `$ifEmpty($json.field, "fallback")` |
| Expression returns `undefined` | Field does not exist on the item | Check field name with `Object.keys($json)` |
| Expression returns `NaN` | Numeric operation on non-numeric string | Convert explicitly: `Number($json.field)` or validate first |
| Expression returns `[object Object]` | Object used in string context | Use `.toJsonString()` or access specific properties |

---

## Variable Availability: Detailed Rules

### Variables NOT Available in Code Node

| Variable | Why Restricted | Alternative |
|----------|---------------|-------------|
| `$itemIndex` | Code node manages its own item iteration | Use `items.indexOf(item)` or loop counter variable |
| `$secrets` | Security restriction in sandboxed environment | Use `$env.SECRET_NAME` to access environment variables |

### Variables ONLY in HTTP Request Node

| Variable | Purpose | Why Restricted |
|----------|---------|---------------|
| `$response.body` | Access response body in subsequent request config | Only meaningful within HTTP Request node chain |
| `$response.headers` | Access response headers | Only meaningful within HTTP Request node chain |
| `$response.statusCode` | Access HTTP status code | Only meaningful within HTTP Request node chain |
| `$response.statusMessage` | Access status message | Only meaningful within HTTP Request node chain |
| `$pageCount` | Track pagination page number | Only meaningful during HTTP Request pagination |

### Variables with Conditional Availability

| Variable | Condition | Behavior When Unavailable |
|----------|-----------|--------------------------|
| `$getWorkflowStaticData()` | ONLY during production execution (trigger/webhook) | Returns empty object `{}` during manual test |
| `$("<Node>").context` | ONLY when working with Loop Over Items node | Returns undefined outside loop context |
| `$("<Node>").item` | Requires intact paired item chain | Returns undefined or wrong data if chain broken |
| `$execution.resumeUrl` | ONLY after Wait node in workflow | Returns undefined if no Wait node present |
| `$execution.resumeFormUrl` | ONLY after Wait node with form | Returns undefined if no form Wait node |

### Python Code Node Variable Mapping

Python Code node uses `_` prefix instead of `$`. ALWAYS translate when moving expressions to Python:

| JavaScript | Python | Notes |
|------------|--------|-------|
| `$json` | `_json` | Available as shorthand |
| `$input.item` | `_item` | Each-item mode only |
| `items` | `_items` | All-items mode only |
| `$env` | `_env` | Same behavior |
| `$vars` | `_vars` | Same behavior |
| `$execution` | `_execution` | Same behavior |
| `$workflow` | `_workflow` | Same behavior |
| `$jmespath()` | `_jmespath()` | Same parameter order |
| `$getWorkflowStaticData()` | `_getWorkflowStaticData()` | Same behavior |
| `$("<Node>")` | `_("<Node>")` | Same methods available |

### Python-Specific Restrictions

| Rule | Reason |
|------|--------|
| ALWAYS use bracket notation: `item["json"]["field"]` | Dot notation causes `AttributeError` in Python |
| NEVER import external libraries on n8n Cloud | Only standard library available on Cloud |
| NEVER use `items.length` | Python uses `len(_items)` |
| NEVER use `getBinaryDataBuffer()` | Not supported in Python Code node |

---

## Extension Method Type Requirements

ALWAYS verify the data type before calling n8n extension methods. Using a method on the wrong type causes `TypeError`.

### String-Only Methods

| Method | Expected Type | Error If Wrong |
|--------|--------------|----------------|
| `.extractEmail()` | String | `TypeError: X.extractEmail is not a function` |
| `.extractUrl()` | String | `TypeError: X.extractUrl is not a function` |
| `.extractDomain()` | String | `TypeError: X.extractDomain is not a function` |
| `.isEmail()` | String | `TypeError: X.isEmail is not a function` |
| `.isUrl()` | String | `TypeError: X.isUrl is not a function` |
| `.removeMarkdown()` | String | `TypeError: X.removeMarkdown is not a function` |
| `.toSentenceCase()` | String | `TypeError` |
| `.toSnakeCase()` | String | `TypeError` |
| `.toTitleCase()` | String | `TypeError` |
| `.urlEncode()` | String or Object | Different behavior per type |
| `.base64Encode()` | String | `TypeError` |
| `.hash()` | String | `TypeError` |

### Array-Only Methods

| Method | Expected Type | Error If Wrong |
|--------|--------------|----------------|
| `.average()` | Array (of numbers) | `TypeError` or `NaN` |
| `.chunk()` | Array | `TypeError` |
| `.compact()` | Array or Object | Different behavior per type |
| `.difference()` | Array | `TypeError` |
| `.intersection()` | Array | `TypeError` |
| `.pluck()` | Array (of objects) | `TypeError` |
| `.sum()` | Array (of numbers) | `TypeError` or `NaN` |
| `.union()` | Array | `TypeError` |
| `.unique()` | Array | `TypeError` |

### Number-Only Methods

| Method | Expected Type | Error If Wrong |
|--------|--------------|----------------|
| `.abs()` | Number | `TypeError` |
| `.ceil()` | Number | `TypeError` |
| `.floor()` | Number | `TypeError` |
| `.isEven()` | Number (integer) | Error if non-integer |
| `.isOdd()` | Number (integer) | Error if non-integer |
| `.round()` | Number | `TypeError` |

### Shared Methods (Multiple Types)

| Method | Supported Types | Notes |
|--------|----------------|-------|
| `.isEmpty()` | String, Array, Object, Number, Boolean | Returns `true` for null on all types |
| `.isNotEmpty()` | String, Array, Object, Number, Boolean | Inverse of `.isEmpty()` |
| `.toBoolean()` | String, Number | String: `"false"`/`"0"`/`"no"` = false; Number: 0 = false |
| `.toDateTime()` | String, Number | String: ISO/RFC/SQL format; Number: Unix ms or seconds |
| `.toJsonString()` | Array, Object | Equivalent to `JSON.stringify()` |
