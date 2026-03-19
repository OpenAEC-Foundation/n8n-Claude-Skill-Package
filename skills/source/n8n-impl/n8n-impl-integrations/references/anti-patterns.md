# API Integration Anti-Patterns

## AP-1: Making HTTP Requests in the Code Node

### Wrong
```javascript
// Code node
const response = await fetch('https://api.example.com/data');
const data = await response.json();
return [{ json: data }];
```

### Why It Fails
- The Code node runs in a sandboxed environment that blocks HTTP requests
- `fetch`, `axios`, and `http` are NOT available in the Code node
- Even if imported, requests bypass n8n's credential encryption and retry logic

### Correct
Use the **HTTP Request** node. ALWAYS use dedicated nodes for HTTP communication. The Code node is for data transformation only.

---

## AP-2: Hardcoding API Keys in Expressions

### Wrong
```
URL: https://api.example.com/data?api_key=sk-abc123xyz
Headers: Authorization: Bearer sk-abc123xyz
```

### Why It Fails
- API keys are visible in workflow JSON exports
- Keys are stored unencrypted in workflow definitions
- Keys appear in execution logs and debug output
- Sharing workflows exposes credentials

### Correct
ALWAYS use n8n credentials (Generic Credential or Predefined). Credentials are encrypted at rest and NEVER appear in workflow definitions or logs.

---

## AP-3: Pagination Without a Stop Condition

### Wrong
```
Pagination: Response Contains Next URL
  Next URL: {{ $response.body.next }}
  (no Maximum Pages set)
  (no Complete When expression)
```

### Why It Fails
- If the API returns a `next` URL on every response (including the last page), n8n fetches pages forever
- Consumes memory, API quota, and execution time until the workflow times out
- Can trigger API rate limits and IP bans

### Correct
ALWAYS set BOTH `Maximum Pages` AND a `Complete When` expression:
```
Maximum Pages: 100
Complete When: {{ $response.body.results.length === 0 }}
```

---

## AP-4: Sending All Items to a Rate-Limited API at Once

### Wrong
```
Get 1000 Items → HTTP Request (processes all 1000 in parallel)
```

### Why It Fails
- n8n sends one request per input item simultaneously
- APIs with rate limits return 429 (Too Many Requests) for most items
- Even with Retry On Fail, the burst pattern repeats on retry
- API provider may temporarily ban your IP or API key

### Correct
Use **Split In Batches** node before the HTTP Request:
```
Get 1000 Items → Split In Batches (size: 10) → HTTP Request → Wait (1s) → Loop back
```

---

## AP-5: Ignoring Error Responses

### Wrong
```
HTTP Request → Process Data (assumes success)
```

### Why It Fails
- API returns non-200 status codes for various reasons (auth expired, resource deleted, server error)
- Without error handling, the workflow either crashes or processes invalid data
- No visibility into which items failed

### Correct
ALWAYS configure error handling:
```
HTTP Request (Continue On Fail: Use Error Output)
  ├── Success → Process Data
  └── Error → Log/Notify/Retry
```

Enable **Retry On Fail** (Max Tries: 3, Wait: 1000ms) for transient errors.

---

## AP-6: Using $response Outside HTTP Request Node

### Wrong
```javascript
// In a Code node or Set node after HTTP Request
const status = $response.statusCode;
const headers = $response.headers;
```

### Why It Fails
- `$response` is ONLY available within the HTTP Request node's own context (parameter expressions and pagination)
- Outside the HTTP Request node, `$response` is `undefined`
- This causes runtime errors that crash the workflow

### Correct
To access response metadata in downstream nodes:
1. Enable **Include Response Headers And Status** in HTTP Request Options
2. Access via `$json.headers` and `$json.statusCode` in subsequent nodes

Or use expressions within the HTTP Request node itself for response-dependent logic.

---

## AP-7: Manually Managing OAuth2 Tokens

### Wrong
```
HTTP Request (GET token) → Set Node (store token) → HTTP Request (use token in header)
                         ↑ Check if expired, refresh ↑
```

### Why It Fails
- Token refresh logic is error-prone and duplicated across workflows
- Tokens stored in workflow variables are not encrypted
- Race conditions when multiple executions refresh simultaneously
- n8n's built-in OAuth2 handles all of this automatically

### Correct
Use an **OAuth2 API credential** (generic or predefined). n8n:
- Stores tokens encrypted
- Refreshes expired tokens automatically
- Handles concurrent access safely
- Provides a one-click authorization flow

---

## AP-8: Not Setting Content-Type for Raw Body

### Wrong
```
Method: POST
Body Content Type: Raw
Body: <soap:Envelope>...</soap:Envelope>
(no Content-Type header set)
```

### Why It Fails
- Without `Content-Type`, the server cannot parse the request body
- Default Content-Type may be `application/octet-stream` or nothing
- SOAP APIs require `text/xml` or `application/soap+xml`
- GraphQL APIs require `application/json`

### Correct
ALWAYS set the Content-Type header explicitly when using Raw body:
```
Headers:
  Content-Type: text/xml; charset=utf-8
  SOAPAction: "http://example.com/action"
Body Content Type: Raw
Body: <soap:Envelope>...</soap:Envelope>
```

---

## AP-9: Using GET Requests with a Body

### Wrong
```
Method: GET
Send Body: true
  Body Content Type: JSON
  { "filter": "active" }
```

### Why It Fails
- The HTTP specification states GET requests SHOULD NOT have a body
- Many servers and proxies strip or ignore GET request bodies
- n8n may not send the body correctly for GET requests
- Causes unpredictable behavior across different API implementations

### Correct
Use **query parameters** for GET request filters:
```
Method: GET
Send Query Parameters: true
  filter: active
```

If the API requires a body for data retrieval, use POST (some APIs use `POST /search` for complex queries).

---

## AP-10: Not Handling Binary Response as File

### Wrong
```
Method: GET
URL: https://api.example.com/reports/download
Response Format: JSON
```

### Why It Fails
- Binary responses (PDF, images, ZIP) cannot be parsed as JSON
- The node throws a JSON parse error
- The binary data is lost

### Correct
Set **Response Format** to **File** for binary downloads:
```
Method: GET
URL: https://api.example.com/reports/download
Response Format: File
Output Binary Property: reportFile
```

Then access via `$binary.reportFile` in subsequent nodes.

---

## AP-11: Duplicating Authentication Across HTTP Request Nodes

### Wrong
```
HTTP Request 1: Headers → Authorization: Bearer {{ $json.token }}
HTTP Request 2: Headers → Authorization: Bearer {{ $json.token }}
HTTP Request 3: Headers → Authorization: Bearer {{ $json.token }}
```

### Why It Fails
- Token value must be passed through the workflow data chain
- If the token source node changes, all HTTP Request nodes need updating
- No centralized credential management
- Tokens in expressions are visible in workflow JSON

### Correct
Create ONE credential and reference it in all HTTP Request nodes:
```
HTTP Request 1: Authentication → Predefined Credential Type → My API Credential
HTTP Request 2: Authentication → Predefined Credential Type → My API Credential
HTTP Request 3: Authentication → Predefined Credential Type → My API Credential
```

Change the credential once, all nodes update automatically.

---

## AP-12: Ignoring Pagination for List Endpoints

### Wrong
```
Method: GET
URL: https://api.example.com/v1/contacts
(no pagination configured)
```

### Why It Fails
- Most APIs return only 20-100 items per request by default
- Without pagination, you only get the first page of results
- Workflows silently process incomplete data
- Data integrity issues downstream (missing records)

### Correct
ALWAYS check API documentation for pagination on list endpoints. Configure pagination:
```
Options:
  Pagination: Response Contains Next URL
    Next URL: {{ $response.body.pagination.nextUrl }}
    Maximum Pages: 50
    Complete When: {{ !$response.body.pagination.nextUrl }}
```
