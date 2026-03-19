# Extension Methods — Complete Reference Tables

> All n8n v1.x expression extension methods with full signatures, parameters, and return types.

## String Methods (19 methods)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `extractEmail` | `.extractEmail()` | `String \| undefined` | Extracts the first email address found in the string. Returns `undefined` if none found. |
| `extractUrl` | `.extractUrl()` | `String \| undefined` | Extracts the first URL (must start with `http`). Returns `undefined` if none found. |
| `extractDomain` | `.extractDomain()` | `String \| undefined` | Extracts the domain from an email address or URL. |
| `extractUrlPath` | `.extractUrlPath()` | `String \| undefined` | Extracts the URL path after the domain. |
| `isEmail` | `.isEmail()` | `Boolean` | Returns `true` if the entire string is a valid email address. |
| `isUrl` | `.isUrl()` | `Boolean` | Returns `true` if the entire string is a valid URL. |
| `isEmpty` | `.isEmpty()` | `Boolean` | Returns `true` if the string has no characters or is `null`. |
| `isNotEmpty` | `.isNotEmpty()` | `Boolean` | Returns `true` if the string contains at least one character. |
| `removeMarkdown` | `.removeMarkdown()` | `String` | Removes Markdown formatting syntax and HTML tags from the string. |
| `toDateTime` | `.toDateTime()` | `DateTime` | Converts string to Luxon DateTime. Supported formats: ISO 8601, HTTP, RFC2822, SQL, Unix milliseconds. |
| `toBoolean` | `.toBoolean()` | `Boolean` | Converts to boolean. `"0"`, `"false"`, `"no"` become `false` (case-insensitive). Everything else becomes `true`. |
| `toSentenceCase` | `.toSentenceCase()` | `String` | Capitalizes the first letter of each sentence. |
| `toSnakeCase` | `.toSnakeCase()` | `String` | Converts to snake_case: spaces and dashes become `_`, symbols removed, all lowercase. |
| `toTitleCase` | `.toTitleCase()` | `String` | Capitalizes the first letter of each word. |
| `urlEncode` | `.urlEncode(allChars?)` | `String` | URL-encodes the string. Pass `true` to also encode URI syntax characters (`/`, `?`, `&`, `=`). |
| `urlDecode` | `.urlDecode(allChars?)` | `String` | Decodes a URL-encoded string. Pass `true` to decode all characters. |
| `base64Encode` | `.base64Encode()` | `String` | Encodes the string as base64. |
| `base64Decode` | `.base64Decode()` | `String` | Decodes a base64-encoded string to plain text. |
| `hash` | `.hash(algo?)` | `String` | Hashes the string. Default algorithm: `md5`. Supported: `md5`, `base64`, `sha1`, `sha224`, `sha256`, `sha384`, `sha512`, `sha3`, `ripemd160`. |
| `quote` | `.quote(mark?)` | `String` | Wraps the string in quotation marks. Escapes existing quotes within the string. Optional: specify the quote character. |
| `replaceSpecialChars` | `.replaceSpecialChars()` | `String` | Replaces special/accented characters with their closest ASCII equivalent (e.g., `e` for `e`). |

## Array Methods (21 methods)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `append` | `.append(elem1, ..., elemN)` | `Array` | Adds one or more elements to the end of the array. |
| `average` | `.average()` | `Number` | Returns the arithmetic mean of all elements. Throws error if array contains non-numbers. |
| `chunk` | `.chunk(length)` | `Array<Array>` | Splits the array into sub-arrays of the specified length. Last chunk may be smaller. |
| `compact` | `.compact()` | `Array` | Removes `null`, `""` (empty string), and `undefined` values from the array. |
| `difference` | `.difference(otherArray)` | `Array` | Returns elements present in the base array but NOT in `otherArray`. |
| `intersection` | `.intersection(otherArray)` | `Array` | Returns elements present in BOTH the base array and `otherArray`. |
| `isEmpty` | `.isEmpty()` | `Boolean` | Returns `true` if the array has no elements or is `null`. |
| `isNotEmpty` | `.isNotEmpty()` | `Boolean` | Returns `true` if the array contains at least one element. |
| `first` | `.first()` | `any` | Returns the first element of the array. |
| `last` | `.last()` | `any` | Returns the last element of the array. |
| `max` | `.max()` | `Number` | Returns the largest number in the array. |
| `min` | `.min()` | `Number` | Returns the smallest number in the array. |
| `pluck` | `.pluck(field1?, field2?, ...)` | `Array` | Extracts values for the specified fields from each object in the array. Single field returns flat array; multiple fields return array of sub-objects. |
| `randomItem` | `.randomItem()` | `any` | Returns a random element from the array. |
| `removeDuplicates` | `.removeDuplicates(keys?)` | `Array` | Removes duplicate entries. For object arrays, pass key name(s) to deduplicate by specific fields. |
| `renameKeys` | `.renameKeys(from, to)` | `Array` | Renames object field names across all objects in the array. |
| `smartJoin` | `.smartJoin(keyField, nameField)` | `Object` | Transforms array of objects into a single object using one field as keys and another as values. |
| `sum` | `.sum()` | `Number` | Returns the sum of all numeric elements. Throws error on non-numbers. |
| `toJsonString` | `.toJsonString()` | `String` | Converts the array to a JSON string (equivalent to `JSON.stringify()`). |
| `union` | `.union(otherArray)` | `Array` | Concatenates both arrays and removes duplicate entries. |
| `unique` | `.unique()` | `Array` | Removes duplicate elements from the array (same as `removeDuplicates()` without keys). |

## Number Methods (11 methods)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `abs` | `.abs()` | `Number` | Returns the absolute value. |
| `ceil` | `.ceil()` | `Number` | Rounds up to the next whole number. |
| `floor` | `.floor()` | `Number` | Rounds down to the nearest whole number. |
| `format` | `.format(locale?, options?)` | `String` | Formats as localized string using `Intl.NumberFormat`. Example: `(1234.5).format("de-DE")` produces `"1.234,5"`. |
| `isEmpty` | `.isEmpty()` | `Boolean` | Returns `false` for ALL numbers. Returns `true` only for `null`. |
| `isEven` | `.isEven()` | `Boolean` | Returns `true` if even. Throws error for non-integer values. |
| `isInteger` | `.isInteger()` | `Boolean` | Returns `true` if the number is a whole number (no decimal part). |
| `isOdd` | `.isOdd()` | `Boolean` | Returns `true` if odd. Throws error for non-integer values. |
| `round` | `.round(decimalPlaces?)` | `Number` | Rounds to nearest whole number, or to specified decimal places. Example: `(3.456).round(2)` returns `3.46`. |
| `toBoolean` | `.toBoolean()` | `Boolean` | `0` becomes `false`. All other numbers become `true`. |
| `toDateTime` | `.toDateTime(format?)` | `DateTime` | Converts numeric timestamp to DateTime. Default: Unix milliseconds. Formats: `"s"` (seconds), `"ms"` (milliseconds), `"excel"` (Excel serial date). |

## Object Methods (12 methods)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `compact` | `.compact()` | `Object` | Removes fields with `null` or `""` values. Does NOT remove `0` or `false`. |
| `hasField` | `.hasField(name)` | `Boolean` | Returns `true` if the top-level key exists. Does NOT check nested fields. |
| `isEmpty` | `.isEmpty()` | `Boolean` | Returns `true` if the object has no keys or is `null`. |
| `isNotEmpty` | `.isNotEmpty()` | `Boolean` | Returns `true` if the object has at least one key. |
| `keepFieldsContaining` | `.keepFieldsContaining(value)` | `Object` | Keeps only fields whose values contain the specified string (partial match). |
| `keys` | `.keys()` | `Array` | Returns an array of all top-level field names. |
| `merge` | `.merge(otherObject)` | `Object` | Merges two objects. Base object values take precedence over `otherObject` values for duplicate keys. |
| `removeField` | `.removeField(key)` | `Object` | Returns a new object with the specified field removed. |
| `removeFieldsContaining` | `.removeFieldsContaining(value)` | `Object` | Removes fields whose values contain the specified string (partial match). |
| `toJsonString` | `.toJsonString()` | `String` | Converts the object to a JSON string. |
| `urlEncode` | `.urlEncode()` | `String` | Converts the object to a URL query parameter string: `key1=value1&key2=value2`. |
| `values` | `.values()` | `Array` | Returns an array of all field values. |

## Boolean Methods (3 methods)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `isEmpty` | `.isEmpty()` | `Boolean` | Returns `false` for ALL boolean values (`true` and `false`). Returns `true` only for `null`. |
| `toNumber` | `.toNumber()` | `Number` | `true` becomes `1`, `false` becomes `0`. |
| `toString` | `.toString()` | `String` | `true` becomes `"true"`, `false` becomes `"false"`. |

## DateTime Methods (Luxon-based)

### Property Accessors (read-only)

| Property | Returns | Description |
|----------|---------|-------------|
| `.day` | `Number` | Day of the month (1-31). |
| `.hour` | `Number` | Hour of the day (0-23). |
| `.minute` | `Number` | Minute (0-59). |
| `.second` | `Number` | Second (0-59). |
| `.millisecond` | `Number` | Millisecond (0-999). |
| `.month` | `Number` | Month (1-12). Note: 1-based, NOT 0-based. |
| `.monthLong` | `String` | Full month name (e.g., `"March"`). |
| `.monthShort` | `String` | Abbreviated month name (e.g., `"Mar"`). |
| `.weekday` | `Number` | Day of week (1=Monday, 7=Sunday). ISO standard. |
| `.weekdayLong` | `String` | Full weekday name (e.g., `"Wednesday"`). |
| `.weekdayShort` | `String` | Abbreviated weekday name (e.g., `"Wed"`). |
| `.weekNumber` | `Number` | ISO week number (1-52). |
| `.quarter` | `Number` | Quarter of the year (1-4). |
| `.year` | `Number` | Full year (e.g., `2026`). |
| `.locale` | `String` | Current locale string (e.g., `"en-US"`). |
| `.zone` | `Object` | Time zone object with `.name` and `.isValid` properties. |
| `.isInDST` | `Boolean` | `true` if the DateTime is in daylight saving time. |

### Transformation Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `plus` | `.plus(n, unit?)` or `.plus({units})` | `DateTime` | Adds a duration. Positional: `.plus(5, "days")`. Object: `.plus({days: 5, hours: 3})`. Units: `years`, `months`, `weeks`, `days`, `hours`, `minutes`, `seconds`, `milliseconds`. |
| `minus` | `.minus(n, unit?)` or `.minus({units})` | `DateTime` | Subtracts a duration. Same syntax as `.plus()`. |
| `set` | `.set(values)` | `DateTime` | Sets specific components. Example: `.set({hour: 9, minute: 0})`. Keys: `year`, `month`, `day`, `hour`, `minute`, `second`, `millisecond`. |
| `startOf` | `.startOf(unit)` | `DateTime` | Rounds down to the start of the given unit. Example: `.startOf("month")` sets to first day at 00:00:00. |
| `endOf` | `.endOf(unit)` | `DateTime` | Rounds up to the end of the given unit. Example: `.endOf("day")` sets to 23:59:59.999. |
| `setZone` | `.setZone(zone)` | `DateTime` | Converts to specified timezone. Examples: `"America/New_York"`, `"UTC+3"`, `"Europe/Amsterdam"`. |
| `setLocale` | `.setLocale(locale)` | `DateTime` | Sets locale for name formatting. Example: `.setLocale("nl")` makes `.monthLong` return `"maart"`. |
| `toLocal` | `.toLocal()` | `DateTime` | Converts to the workflow's local timezone. |
| `toUTC` | `.toUTC(offset?)` | `DateTime` | Converts to UTC. Optional offset in minutes. |

### Comparison & Analysis Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `equals` | `.equals(other)` | `Boolean` | Returns `true` if both represent the identical moment AND timezone. |
| `hasSame` | `.hasSame(other, unit)` | `Boolean` | Returns `true` if both share the same value at the specified unit (ignores timezone). Example: `.hasSame(otherDate, "day")`. |
| `isBetween` | `.isBetween(date1, date2)` | `Boolean` | Returns `true` if the DateTime falls between `date1` and `date2` (exclusive on both bounds). |
| `diffTo` | `.diffTo(other, unit)` | `Duration` | Returns the difference as a Luxon Duration object. Access the value via `.days`, `.hours`, etc. |
| `diffToNow` | `.diffToNow(unit)` | `Duration` | Returns the difference between the DateTime and the current moment. |
| `extract` | `.extract(unit?)` | `Number` | Extracts a part of the date. Units: `year`, `month`, `week`, `day`, `hour`, `minute`, `second`. |

### String Conversion Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `format` | `.format(fmt)` | `String` | Formats using Luxon token syntax. Example: `.format("yyyy-MM-dd HH:mm:ss")`. See Luxon Format Tokens below. |
| `toLocaleString` | `.toLocaleString(options?)` | `String` | Returns a locale-specific date string. |
| `toISO` | `.toISO()` | `String` | Returns ISO 8601 string: `"2026-03-19T14:05:09.000+01:00"`. |
| `toString` | `.toString()` | `String` | Returns ISO-formatted string (same as `.toISO()`). |
| `toRelative` | `.toRelative(options?)` | `String` | Returns human-readable relative time string (e.g., `"2 days ago"`, `"in 3 hours"`). |
| `toMillis` | `.toMillis()` | `Number` | Returns Unix timestamp in milliseconds. |
| `toSeconds` | `.toSeconds()` | `Number` | Returns Unix timestamp in seconds. |

### Luxon Format Tokens — Complete Reference

| Token | Meaning | Example Output |
|-------|---------|----------------|
| `yyyy` | 4-digit year | `2026` |
| `yy` | 2-digit year | `26` |
| `MMMM` | Full month name | `March` |
| `MMM` | Short month name | `Mar` |
| `MM` | 2-digit month | `03` |
| `M` | Month (no padding) | `3` |
| `dd` | 2-digit day | `09` |
| `d` | Day (no padding) | `9` |
| `EEEE` | Full weekday | `Wednesday` |
| `EEE` | Short weekday | `Wed` |
| `HH` | 24-hour hour (padded) | `14` |
| `H` | 24-hour hour | `4` |
| `hh` | 12-hour hour (padded) | `02` |
| `h` | 12-hour hour | `2` |
| `mm` | Minutes (padded) | `05` |
| `m` | Minutes | `5` |
| `ss` | Seconds (padded) | `09` |
| `s` | Seconds | `9` |
| `SSS` | Milliseconds | `123` |
| `a` | AM/PM | `PM` |
| `Z` | Short UTC offset | `+01:00` |
| `ZZ` | Tech UTC offset | `+0100` |
| `ZZZZ` | Timezone name | `Central European Time` |
| `z` | Short timezone | `CET` |

### Common Format Patterns

| Pattern | Token String | Output Example |
|---------|-------------|----------------|
| ISO date | `yyyy-MM-dd` | `2026-03-19` |
| ISO datetime | `yyyy-MM-dd'T'HH:mm:ss` | `2026-03-19T14:05:09` |
| European date | `dd-MM-yyyy` | `19-03-2026` |
| European with slashes | `dd/MM/yyyy` | `19/03/2026` |
| US date | `MM/dd/yyyy` | `03/19/2026` |
| Human-readable | `EEEE, MMMM dd yyyy` | `Wednesday, March 19 2026` |
| Time (24h) | `HH:mm:ss` | `14:05:09` |
| Time (12h) | `hh:mm a` | `02:05 PM` |
| Date + time | `dd MMM yyyy HH:mm` | `19 Mar 2026 14:05` |
| Filename-safe | `yyyy-MM-dd_HH-mm-ss` | `2026-03-19_14-05-09` |

## BinaryFile Properties (7 properties)

| Property | Returns | Description |
|----------|---------|-------------|
| `.directory` | `String` | Folder path where the file is stored. NOT set when binary data is stored in the database. |
| `.fileExtension` | `String` | File extension without dot (e.g., `"pdf"`, `"jpg"`, `"txt"`). |
| `.fileName` | `String` | Complete filename including extension (e.g., `"report.pdf"`). |
| `.fileSize` | `String` | File size in bytes as a STRING. ALWAYS convert with `parseInt()` for math operations. |
| `.fileType` | `String` | First part of MIME type only (e.g., `"image"`, `"video"`, `"application"`, `"text"`). |
| `.id` | `String` | Unique identifier for the binary data in disk or S3 storage. |
| `.mimeType` | `String` | Full MIME type (e.g., `"image/jpeg"`, `"text/plain"`, `"application/pdf"`). |

### Access Pattern

```js
// Access binary metadata for the current item
{{ $binary.data.fileName }}
{{ $binary.data.mimeType }}
{{ $binary.data.fileSize }}

// Access from a specific node
{{ $("Read File").item.binary.data.fileName }}

// Multiple binary properties on same item
{{ $binary.attachment.fileName }}
{{ $binary.image.mimeType }}
```
