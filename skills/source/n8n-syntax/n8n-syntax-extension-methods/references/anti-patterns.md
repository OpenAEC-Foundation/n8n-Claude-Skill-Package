# Extension Methods — Anti-Patterns

> Common mistakes when using n8n v1.x extension methods and how to fix them.

## String Method Anti-Patterns

### AP-S01: Not handling undefined from extraction methods

```js
// WRONG — extractEmail() returns undefined if no email found
{{ $json.text.extractEmail().extractDomain() }}
// TypeError: Cannot read property 'extractDomain' of undefined

// CORRECT — use $ifEmpty() for safe fallback
{{ $ifEmpty($json.text.extractEmail(), "unknown@example.com").extractDomain() }}

// CORRECT — check with ternary
{{ $json.text.extractEmail() ? $json.text.extractEmail().extractDomain() : "no-domain" }}
```

### AP-S02: Using .hash() without specifying algorithm

```js
// WRONG — relies on implicit md5 default, unclear intent
{{ $json.payload.hash() }}

// CORRECT — ALWAYS specify the algorithm explicitly
{{ $json.payload.hash("sha256") }}
```

### AP-S03: Using .urlEncode() without allChars for path segments

```js
// WRONG — does not encode slashes and special URI characters
{{ "path/with spaces?query=value".urlEncode() }}
// → "path/with%20spaces?query=value" (slashes and ? preserved)

// CORRECT — pass true to encode ALL characters when encoding a value, not a URL
{{ $json.fileName.urlEncode(true) }}
```

### AP-S04: Assuming .toBoolean() works like JavaScript truthy

```js
// WRONG assumption — "0" is truthy in JavaScript but false in n8n
{{ "0".toBoolean() }}
// Returns false (n8n treats "0", "false", "no" as false)

// This is CORRECT behavior in n8n — but be aware of the difference from JS
// ALWAYS check the n8n conversion rules, not JavaScript truthy/falsy rules
```

### AP-S05: Using .toDateTime() on non-date strings

```js
// WRONG — unparseable string throws error
{{ "not a date".toDateTime() }}
// Error: Invalid DateTime

// CORRECT — validate or use conditional logic
{{ $json.value.isEmpty() ? $now : $json.value.toDateTime() }}
```

## Array Method Anti-Patterns

### AP-A01: Using .sum() or .average() on mixed-type arrays

```js
// WRONG — throws error if array contains non-numbers
{{ $json.values.sum() }}
// Error when values = [10, "twenty", 30]

// CORRECT — pluck numeric field first, or ensure numeric data
{{ $json.items.pluck("price").sum() }}

// CORRECT — compact first to remove nulls
{{ $json.amounts.compact().sum() }}
```

### AP-A02: Using .removeDuplicates() without keys for objects

```js
// WRONG — object reference comparison, rarely removes anything
{{ $json.contacts.removeDuplicates() }}
// Does NOT deduplicate objects with same field values

// CORRECT — specify the key to deduplicate by
{{ $json.contacts.removeDuplicates("email") }}
```

### AP-A03: Confusing .unique() with .removeDuplicates()

```js
// .unique() works well for primitive arrays
{{ [1, 2, 2, 3, 3].unique() }}
// → [1, 2, 3]

// For object arrays, ALWAYS use .removeDuplicates("key")
{{ $json.users.removeDuplicates("id") }}
// .unique() on objects compares references, not values
```

### AP-A04: Calling .isEven()/.isOdd() on array length without checking integer

```js
// WRONG — .isEven() is a Number method, not available on arrays
{{ $json.items.length.isEven() }}
// This actually works since length is always integer, but...

// ...the real mistake is calling .isEven() on non-integers elsewhere
{{ $json.ratio.isEven() }}
// Error if ratio = 3.14

// CORRECT — check isInteger first
{{ $json.ratio.isInteger() ? $json.ratio.isEven() : false }}
```

### AP-A05: Using .smartJoin() with wrong field order

```js
// WRONG — parameters reversed: (keyField, valueField)
{{ $json.items.smartJoin("value", "key") }}
// Uses "value" as keys and "key" as values — backwards

// CORRECT — first param is the key field, second is the value field
{{ $json.items.smartJoin("key", "value") }}
// [{key: "color", value: "red"}] → {color: "red"}
```

## Number Method Anti-Patterns

### AP-N01: Using .toDateTime() without specifying format

```js
// WRONG — ambiguous: is 1710850000 in seconds or milliseconds?
{{ $json.timestamp.toDateTime() }}
// Default is milliseconds, but Unix timestamps are often in seconds

// CORRECT — ALWAYS specify the format
{{ $json.timestamp.toDateTime("s") }}    // Unix seconds
{{ $json.timestamp.toDateTime("ms") }}   // Unix milliseconds
{{ $json.timestamp.toDateTime("excel") }} // Excel serial date
```

### AP-N02: Formatting without locale

```js
// WRONG — uses system default locale, inconsistent across environments
{{ $json.price.format() }}

// CORRECT — ALWAYS specify locale for predictable output
{{ $json.price.format("en-US") }}  // "1,234.50"
{{ $json.price.format("de-DE") }}  // "1.234,50"
{{ $json.price.format("nl-NL") }}  // "1.234,50"
```

### AP-N03: Using .round() and expecting string output

```js
// WRONG — .round(2) returns a number, may lose trailing zero
{{ $json.price.round(2) }}
// 10.50 → 10.5 (number), not "10.50" (string)

// CORRECT — chain with .format() for display with fixed decimals
{{ $json.price.round(2).format("en-US") }}
```

## Object Method Anti-Patterns

### AP-O01: Expecting .merge() to overwrite with argument values

```js
// WRONG assumption — merge gives precedence to BASE object
{{ $json.userInput.merge($json.defaults) }}
// userInput values WIN for duplicate keys, not defaults

// If you want defaults to override, swap the order:
{{ $json.defaults.merge($json.userInput) }}
// Now defaults WIN for duplicate keys

// CORRECT for "defaults with overrides" pattern:
// Put defaults as base, user input as argument
{{ $json.defaults.merge($json.userInput) }}
// Wait — this makes defaults win. That is backwards for most use cases.

// The CORRECT pattern: user input is base, defaults fill in gaps
{{ $json.userInput.merge($json.defaults) }}
// userInput values preserved, defaults fill missing keys only
```

### AP-O02: Using .hasField() to check nested fields

```js
// WRONG — hasField only checks top-level keys
{{ $json.data.hasField("address.city") }}
// Returns false even if $json.data.address.city exists

// CORRECT — check each level
{{ $json.data.hasField("address") && $json.data.address.hasField("city") }}
```

### AP-O03: Using .compact() and expecting 0 or false to be removed

```js
// .compact() only removes null and "" (empty string)
{{ {a: 0, b: false, c: null, d: "", e: "value"}.compact() }}
// → {a: 0, b: false, e: "value"}
// 0 and false are KEPT — only null and "" are removed

// If you need to remove falsy values, use .removeFieldsContaining() or Code node
```

### AP-O04: Using .keepFieldsContaining() with non-string values

```js
// WRONG — matching is string-based, numbers may not match as expected
{{ $json.data.keepFieldsContaining(42) }}

// CORRECT — use string matching
{{ $json.data.keepFieldsContaining("42") }}
```

## DateTime Anti-Patterns

### AP-D01: Using new Date() instead of Luxon

```js
// WRONG — JavaScript Date does not respect workflow timezone
{{ new Date().toISOString() }}

// CORRECT — use $now (respects workflow timezone)
{{ $now.toISO() }}

// CORRECT — use $today for midnight
{{ $today.toISO() }}
```

### AP-D02: Using moment.js format tokens with Luxon

```js
// WRONG — moment.js tokens do not work with Luxon .format()
{{ $now.format("YYYY-MM-DD") }}
// Produces "YYYY-03-DD" (literal Y and D characters)

// CORRECT — use Luxon tokens (lowercase yyyy, dd)
{{ $now.format("yyyy-MM-dd") }}
// → "2026-03-19"
```

**Common token mistakes:**

| moment.js (WRONG) | Luxon (CORRECT) | Output |
|-------------------|-----------------|--------|
| `YYYY` | `yyyy` | `2026` |
| `DD` | `dd` | `19` |
| `Do` | Not supported | Use `d` + ordinal logic |
| `dddd` | `EEEE` | `Wednesday` |
| `ddd` | `EEE` | `Wed` |
| `A` | `a` | `PM` |

### AP-D03: Treating .diffTo() result as a plain number

```js
// WRONG — diffTo returns a Duration object, not a number
{{ $json.deadline.toDateTime().diffTo($now, "days") }}
// Returns Duration object, not a number

// CORRECT — access the specific unit property
{{ $json.deadline.toDateTime().diffTo($now, "days").days }}
// → 14.5 (number)

// CORRECT — round for clean display
{{ $json.deadline.toDateTime().diffTo($now, "days").days.round() }}
// → 15
```

### AP-D04: Assuming .isBetween() is inclusive

```js
// WRONG assumption — isBetween is EXCLUSIVE on both bounds
{{ $now.isBetween($now, $now.plus(1, "days")) }}
// Returns false — the start bound is excluded

// Be aware: exact boundary values return false
// Use .equals() for boundary checks if needed
```

### AP-D05: Chaining .setZone() and expecting time values to change

```js
// .setZone() changes the timezone representation, not the moment in time
{{ $now.setZone("America/New_York").hour }}
// Returns the hour in New York timezone — the MOMENT is the same,
// only the representation changes

// This is CORRECT behavior — just be aware that:
// .setZone() does NOT add/subtract hours
// It converts the SAME instant to a different timezone representation
```

### AP-D06: Using string concatenation for dates instead of .format()

```js
// WRONG — manual concatenation is error-prone and ignores padding
{{ $now.year + "-" + $now.month + "-" + $now.day }}
// → "2026-3-9" (no zero-padding)

// CORRECT — use .format() for consistent output
{{ $now.format("yyyy-MM-dd") }}
// → "2026-03-09"
```

## BinaryFile Anti-Patterns

### AP-B01: Using .fileSize as a number without conversion

```js
// WRONG — fileSize is a STRING, math operations produce wrong results
{{ $binary.data.fileSize > 1000000 }}
// String comparison, not numeric: "9999" > "1000000" is true

// CORRECT — ALWAYS convert to number first
{{ parseInt($binary.data.fileSize) > 1000000 }}
```

### AP-B02: Assuming .directory is always set

```js
// WRONG — directory is empty when binary data is stored in database
{{ $binary.data.directory + "/" + $binary.data.fileName }}
// → "undefined/report.pdf" or "/report.pdf"

// CORRECT — check before using
{{ $binary.data.directory ? $binary.data.directory + "/" + $binary.data.fileName : $binary.data.fileName }}
```

### AP-B03: Using .fileType when you need the full MIME type

```js
// WRONG — fileType only returns the first part of MIME type
{{ $binary.data.fileType }}
// → "image" (not "image/jpeg")

// CORRECT — use .mimeType for the full MIME type
{{ $binary.data.mimeType }}
// → "image/jpeg"
```

## General Anti-Patterns

### AP-G01: Chaining without understanding return types

```js
// WRONG — .sum() returns a Number, not an Array
{{ $json.items.pluck("price").sum().round() }}
// This actually works because .round() is a Number method

// But THIS is wrong:
{{ $json.items.pluck("price").sum().pluck("value") }}
// Error: .pluck() is not a function (sum returns Number, not Array)

// ALWAYS check: what type does the previous method return?
```

### AP-G02: Using JavaScript methods when n8n extensions exist

```js
// WRONG — JavaScript Array methods lack n8n null-safety
{{ JSON.stringify($json.data) }}

// CORRECT — use n8n extension methods
{{ $json.data.toJsonString() }}

// WRONG — manual deduplication
{{ [...new Set($json.items)] }}

// CORRECT — use built-in method
{{ $json.items.unique() }}
```

### AP-G03: Not using $ifEmpty() for null-safe access

```js
// WRONG — throws error if field is null/undefined
{{ $json.optionalField.toTitleCase() }}

// CORRECT — provide fallback
{{ $ifEmpty($json.optionalField, "default value").toTitleCase() }}
```
