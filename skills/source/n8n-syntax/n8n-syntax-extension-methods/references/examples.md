# Extension Methods — Practical Examples

> Real-world patterns combining multiple extension methods in n8n v1.x expressions.

## String Method Combinations

### Extract and Validate Email from Text

```js
// Extract email and get its domain
{{ $json.body.extractEmail() }}
// → "john@example.com"

{{ $json.body.extractEmail().extractDomain() }}
// → "example.com"

// Safe extraction with fallback
{{ $ifEmpty($json.notes.extractEmail(), "no-email@unknown.com") }}
```

### Clean and Encode User Input

```js
// Prepare a search query for URL
{{ $json.searchTerm.replaceSpecialChars().urlEncode() }}

// Clean markdown from CMS content and convert to title case
{{ $json.rawContent.removeMarkdown().toTitleCase() }}

// Create a slug from a title
{{ $json.title.toSnakeCase() }}
// "My Blog Post!" → "my_blog_post"
```

### Hashing and Encoding

```js
// Create SHA256 hash of a payload for webhook verification
{{ $json.payload.hash("sha256") }}

// Base64 encode credentials for Basic Auth header
{{ ($json.username + ":" + $json.password).base64Encode() }}

// Decode a base64 message
{{ $json.encodedMessage.base64Decode() }}
```

### Type Conversion from Strings

```js
// Convert date string to DateTime for arithmetic
{{ $json.dateStr.toDateTime().plus(7, "days").toISO() }}

// Convert string flag to boolean
{{ $json.isActive.toBoolean() }}
// "yes" → true, "false" → false, "0" → false
```

## Array Method Combinations

### Extract and Aggregate from Object Arrays

```js
// Get all email addresses from contacts
{{ $json.contacts.pluck("email") }}
// → ["a@x.com", "b@y.com", "c@x.com"]

// Get unique domains from email list
{{ $json.contacts.pluck("email").unique() }}

// Calculate total order value
{{ $json.lineItems.pluck("price").sum() }}

// Get average rating
{{ $json.reviews.pluck("rating").average() }}

// Find min and max prices
{{ $json.products.pluck("price").min() }}
{{ $json.products.pluck("price").max() }}
```

### Deduplicate and Filter

```js
// Remove duplicate contacts by email
{{ $json.contacts.removeDuplicates("email") }}

// Remove null/empty entries from a list
{{ $json.tags.compact() }}
// ["n8n", "", null, "automation", undefined] → ["n8n", "automation"]

// Get unique items from merged sources
{{ $json.sourceA.union($json.sourceB) }}
```

### Set Operations

```js
// Find customers who exist in both lists
{{ $json.newsletterSubscribers.intersection($json.activeCustomers) }}

// Find unsubscribed users (in old list but not new)
{{ $json.previousSubscribers.difference($json.currentSubscribers) }}
```

### Restructuring Arrays

```js
// Batch API calls — split 100 items into groups of 10
{{ $json.allItems.chunk(10) }}
// → [[item1..item10], [item11..item20], ...]

// Convert array of key-value pairs to single object
{{ $json.settings.smartJoin("key", "value") }}
// [{key: "color", value: "red"}, {key: "size", value: "L"}]
// → {color: "red", size: "L"}

// Rename keys across all objects
{{ $json.apiResponse.renameKeys("first_name", "firstName") }}

// Add a computed element
{{ $json.items.append({name: "Total", value: $json.items.pluck("value").sum()}) }}
```

### Array to String

```js
// Serialize for API body
{{ $json.selectedIds.toJsonString() }}
// [1, 2, 3] → "[1,2,3]"

// Get first and last
{{ $json.results.first() }}
{{ $json.results.last() }}
```

## Number Method Combinations

### Format Currency and Numbers

```js
// Format as currency-style number
{{ $json.price.round(2).format("en-US") }}
// 1234.5 → "1,234.50" (with options for currency symbol)

// European number format
{{ $json.amount.format("de-DE") }}
// 1234.5 → "1.234,5"

// Round to 2 decimal places
{{ $json.tax.round(2) }}
// 15.6789 → 15.68
```

### Numeric Validation

```js
// Check before even/odd operations
{{ $json.value.isInteger() ? ($json.value.isEven() ? "even" : "odd") : "decimal" }}

// Absolute difference
{{ ($json.expected - $json.actual).abs() }}
```

### Timestamp Conversion

```js
// Unix timestamp (seconds) to formatted date
{{ $json.created_at.toDateTime("s").format("yyyy-MM-dd HH:mm") }}

// Unix timestamp (milliseconds) to ISO
{{ $json.timestamp.toDateTime().toISO() }}

// Excel serial date to readable format
{{ $json.excelDate.toDateTime("excel").format("dd MMM yyyy") }}
```

## Object Method Combinations

### Clean and Filter Objects

```js
// Remove all null/empty fields before API call
{{ $json.formData.compact() }}
// {name: "John", phone: "", email: null, age: 30}
// → {name: "John", age: 30}

// Keep only fields containing a search term
{{ $json.metadata.keepFieldsContaining("production") }}
// {env: "production", url: "production.api.com", debug: "false"}
// → {env: "production", url: "production.api.com"}

// Remove sensitive fields
{{ $json.user.removeField("password").removeField("ssn") }}
```

### Object Inspection

```js
// Check if field exists before accessing
{{ $json.response.hasField("error") ? $json.response.error : "No error" }}

// Get all field names
{{ $json.record.keys() }}
// → ["id", "name", "email", "created"]

// Get all values
{{ $json.scores.values() }}
// → [95, 87, 92]
```

### Merge and Serialize

```js
// Merge defaults with user input (user input = base, takes precedence)
{{ $json.userInput.merge($json.defaults) }}

// Convert to query string for URL
{{ {page: 1, limit: 50, sort: "name"}.urlEncode() }}
// → "page=1&limit=50&sort=name"

// Serialize for logging
{{ $json.payload.toJsonString() }}
```

## DateTime Combinations

### Date Arithmetic

```js
// One week ago at start of day
{{ $now.minus({days: 7}).startOf("day").toISO() }}
// → "2026-03-12T00:00:00.000+01:00"

// End of current month
{{ $now.endOf("month").format("yyyy-MM-dd") }}
// → "2026-03-31"

// Start of current quarter
{{ $now.startOf("quarter").toISO() }}

// 90 days from now
{{ $today.plus(90, "days").format("yyyy-MM-dd") }}

// Set specific time on today's date
{{ $today.set({hour: 9, minute: 30}).toISO() }}
```

### Date Comparison

```js
// Check if a date is within a range
{{ $json.eventDate.toDateTime().isBetween($today, $today.plus(30, "days")) }}

// Check if two dates are the same day (ignoring time)
{{ $json.dateA.toDateTime().hasSame($json.dateB.toDateTime(), "day") }}

// Days until deadline
{{ $today.diffTo($json.deadline.toDateTime(), "days").days }}

// Time since last update
{{ $json.lastModified.toDateTime().diffToNow("hours").hours.round() }}
```

### Date Formatting

```js
// Human-readable relative time
{{ $json.createdAt.toDateTime().toRelative() }}
// → "3 days ago"

// Format for European audience
{{ $now.format("dd-MM-yyyy HH:mm") }}
// → "19-03-2026 14:05"

// Format for filename
{{ $now.format("yyyy-MM-dd_HH-mm-ss") }}
// → "2026-03-19_14-05-09"

// ISO date only (no time)
{{ $now.format("yyyy-MM-dd") }}
// → "2026-03-19"

// Full human-readable
{{ $now.format("EEEE, MMMM dd yyyy 'at' HH:mm") }}
// → "Wednesday, March 19 2026 at 14:05"
```

### Timezone Handling

```js
// Convert to specific timezone
{{ $now.setZone("America/New_York").format("HH:mm z") }}
// → "09:05 EST"

// Convert to UTC for API calls
{{ $now.toUTC().toISO() }}

// Convert to local workflow timezone
{{ $json.utcTimestamp.toDateTime().toLocal().format("yyyy-MM-dd HH:mm") }}
```

### Extract Date Components

```js
// Get day of week for scheduling logic
{{ $now.weekday }}
// 1=Monday ... 7=Sunday

// Get month name for reporting
{{ $now.monthLong }}
// → "March"

// Quarter-based routing
{{ $now.quarter }}
// → 1 (Q1: Jan-Mar)

// Week number for weekly reports
{{ $now.weekNumber }}
```

## Mixed Type Chains

### String to DateTime to Formatted Output

```js
// Parse date string, add time, format for display
{{ $json.dateString.toDateTime().plus(2, "hours").format("dd MMM yyyy HH:mm") }}
```

### Array to Aggregated Number to Formatted String

```js
// Calculate and format total
{{ $json.orders.pluck("amount").sum().round(2).format("en-US") }}
// → "12,345.67"
```

### Object Keys to Filtered Array

```js
// Get field names and check for required fields
{{ ["name", "email"].difference($json.formData.keys()) }}
// Returns missing required fields
```

## BinaryFile Metadata Patterns

```js
// Route by file type
{{ $binary.data.fileType }}
// "image" → send to image processing
// "application" → send to document parser

// Build output filename
{{ "processed_" + $binary.data.fileName }}
// → "processed_report.pdf"

// Check file size (convert string to number first)
{{ parseInt($binary.data.fileSize) > 5000000 ? "large" : "small" }}

// Get MIME type for Content-Type header
{{ $binary.data.mimeType }}
// → "application/pdf"
```
