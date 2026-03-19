# Webhook Examples

## Example 1: Basic POST Webhook

**Use case**: Receive JSON data from an external service.

### Webhook Node Configuration
```
HTTP Method: POST
Path: incoming/orders
Authentication: None
Respond: When Last Node Finishes
Response Data: First Entry JSON
```

### Test
```bash
# Test URL (workflow in test mode)
curl -X POST https://n8n.example.com/webhook-test/incoming/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "12345", "amount": 99.99}'

# Production URL (workflow activated)
curl -X POST https://n8n.example.com/webhook/incoming/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "12345", "amount": 99.99}'
```

### Accessing Data in Subsequent Nodes
```javascript
// Expression
{{ $json.body.orderId }}   // "12345"
{{ $json.body.amount }}    // 99.99

// Code node
const orderId = $input.first().json.body.orderId;
```

---

## Example 2: Authenticated Webhook with Header Auth

**Use case**: Secure webhook for a payment provider callback.

### Step 1: Create Header Auth Credential
```
Name: Payment Provider API Key
Header Name: X-Payment-Secret
Header Value: sk_live_abc123xyz
```

### Step 2: Webhook Node Configuration
```
HTTP Method: POST
Path: payments/callback
Authentication: Header Auth
Credential: Payment Provider API Key
Respond: When Last Node Finishes
```

### Step 3: Additional Options
```
IP Whitelist: 203.0.113.10,203.0.113.11
Response Code: 200
```

### Caller Request
```bash
curl -X POST https://n8n.example.com/webhook/payments/callback \
  -H "Content-Type: application/json" \
  -H "X-Payment-Secret: sk_live_abc123xyz" \
  -d '{"event": "payment.completed", "paymentId": "pay_001"}'
```

### Rejection Scenarios
- Missing header: HTTP 401 Unauthorized
- Wrong header value: HTTP 401 Unauthorized
- Unlisted IP: HTTP 403 Forbidden

---

## Example 3: Basic Auth Webhook

**Use case**: Legacy system integration requiring username/password.

### Step 1: Create Basic Auth Credential
```
Name: Legacy System Auth
Username: webhook_user
Password: s3cur3_p4ss
```

### Step 2: Webhook Node Configuration
```
HTTP Method: POST
Path: legacy/data-sync
Authentication: Basic Auth
Credential: Legacy System Auth
Respond: Immediately
```

### Caller Request
```bash
curl -X POST https://n8n.example.com/webhook/legacy/data-sync \
  -u "webhook_user:s3cur3_p4ss" \
  -H "Content-Type: application/json" \
  -d '{"records": [{"id": 1, "name": "Item A"}]}'
```

---

## Example 4: File Upload Webhook

**Use case**: Receive file uploads via multipart form data.

### Webhook Node Configuration
```
HTTP Method: POST
Path: uploads/documents
Authentication: Header Auth
Respond: Using 'Respond to Webhook' Node
```

### Additional Options
```
Binary Data: Enabled
```

### Workflow Design
```
Webhook --> Move Binary Data --> Write Binary File --> Respond to Webhook
```

### Accessing Binary Data (Code Node)
```javascript
// Access the uploaded file
const items = $input.all();
for (const item of items) {
  if (item.binary) {
    for (const [key, binaryData] of Object.entries(item.binary)) {
      const fileName = binaryData.fileName;
      const mimeType = binaryData.mimeType;
      const fileSize = binaryData.fileSize;
      // Process file...
    }
  }
}
return items;
```

### Respond to Webhook Configuration
```
Respond With: JSON
Response Body: { "status": "uploaded", "message": "File received successfully" }
Response Code: 201
```

### Caller Request
```bash
curl -X POST https://n8n.example.com/webhook/uploads/documents \
  -H "X-API-Key: upload_secret_123" \
  -F "file=@invoice.pdf"
```

---

## Example 5: Custom JSON Response with Respond to Webhook

**Use case**: Process data and return a crafted API response.

### Webhook Node Configuration
```
HTTP Method: POST
Path: api/v1/process
Authentication: Header Auth
Respond: Using 'Respond to Webhook' Node
```

### Workflow Design
```
Webhook --> Code (process data) --> Respond to Webhook
```

### Code Node (Processing)
```javascript
const input = $input.first().json.body;
const result = {
  processedAt: new Date().toISOString(),
  inputCount: input.items?.length || 0,
  status: "completed",
  results: input.items?.map(item => ({
    id: item.id,
    processed: true
  })) || []
};
return [{ json: result }];
```

### Respond to Webhook Configuration
```
Respond With: First Incoming Item
Response Code: 200
Response Headers:
  X-Processing-Time: {{ $now.toISO() }}
  Content-Type: application/json
```

---

## Example 6: JWT Authentication Webhook

**Use case**: Validate incoming JWTs from an OAuth-protected service.

### Step 1: Create JWT Credential
```
Name: Service JWT Validator
Algorithm: RS256
Public Key: -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhki...
-----END PUBLIC KEY-----
```

### Step 2: Webhook Node Configuration
```
HTTP Method: POST
Path: secure/events
Authentication: JWT Auth
Credential: Service JWT Validator
Respond: When Last Node Finishes
```

### Caller Request
```bash
curl -X POST https://n8n.example.com/webhook/secure/events \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{"event": "user.created", "userId": "usr_123"}'
```

### Accessing JWT Claims
```javascript
// The decoded JWT payload is available in the webhook output
const userId = $json.body.userId;
// JWT claims are available depending on n8n version
```

---

## Example 7: Dynamic Path Parameters

**Use case**: Multi-tenant webhook with resource identification.

### Webhook Node Configuration
```
HTTP Method: POST
Path: :tenantId/events/:eventType
Authentication: Header Auth
Respond: When Last Node Finishes
```

### Caller Request
```bash
curl -X POST https://n8n.example.com/webhook/acme-corp/events/order-placed \
  -H "X-API-Key: tenant_secret" \
  -H "Content-Type: application/json" \
  -d '{"orderId": "ORD-789"}'
```

### Accessing Path Parameters
```javascript
// In expressions
{{ $json.params.tenantId }}    // "acme-corp"
{{ $json.params.eventType }}   // "order-placed"

// In Code node
const tenantId = $input.first().json.params.tenantId;
const eventType = $input.first().json.params.eventType;
```

---

## Example 8: Respond with JWT Token

**Use case**: Custom authentication endpoint that issues JWTs.

### Webhook Node Configuration
```
HTTP Method: POST
Path: auth/token
Authentication: None
Respond: Using 'Respond to Webhook' Node
```

### Workflow Design
```
Webhook --> Code (validate credentials) --> IF (valid)
  --> Respond to Webhook (JWT Token)
  --> Respond to Webhook (Error: 401)
```

### Respond to Webhook Configuration (Success Path)
```
Respond With: JWT Token
JWT Secret: your-signing-secret
Algorithm: HS256
Expiration: 3600
Payload:
  userId: {{ $json.userId }}
  role: {{ $json.role }}
```

### Respond to Webhook Configuration (Error Path)
```
Respond With: JSON
Response Body: { "error": "Invalid credentials" }
Response Code: 401
```

**IMPORTANT**: Only the FIRST Respond to Webhook node that executes will send a response. Place error and success responses on separate branches.

---

## Example 9: Webhook with CORS for Browser Clients

**Use case**: Frontend application calling webhook via JavaScript.

### Webhook Node Configuration
```
HTTP Method: POST
Path: api/submit-form
Authentication: None
Respond: Using 'Respond to Webhook' Node
```

### Additional Options
```
CORS: https://app.example.com,https://staging.example.com
Response Headers:
  Access-Control-Allow-Headers: Content-Type,Authorization
```

### Frontend Code (JavaScript)
```javascript
const response = await fetch('https://n8n.example.com/webhook/api/submit-form', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: 'John', email: 'john@example.com' })
});
const result = await response.json();
```

---

## Example 10: GET Webhook for Health Check

**Use case**: External monitoring service checking webhook availability.

### Webhook Node Configuration
```
HTTP Method: GET
Path: health/webhook-status
Authentication: None
Respond: When Last Node Finishes
Response Data: First Entry JSON
```

### Workflow Design
```
Webhook --> Code (return status) --> (response sent automatically)
```

### Code Node
```javascript
return [{
  json: {
    status: "healthy",
    timestamp: new Date().toISOString(),
    version: "1.0.0"
  }
}];
```

### Caller Request
```bash
curl https://n8n.example.com/webhook/health/webhook-status
# Response: {"status":"healthy","timestamp":"2025-01-15T10:30:00Z","version":"1.0.0"}
```
