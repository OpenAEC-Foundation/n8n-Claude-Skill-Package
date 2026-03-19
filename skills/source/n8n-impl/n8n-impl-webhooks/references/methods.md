# Webhook Methods Reference

## Webhook Node — Complete Options

### Core Parameters

| Parameter | Values | Description |
|-----------|--------|-------------|
| **HTTP Method** | GET, POST, PUT, PATCH, DELETE, HEAD | HTTP method to listen for |
| **Path** | String | URL path after `/webhook/` or `/webhook-test/` |
| **Authentication** | None, Basic Auth, Header Auth, JWT Auth | Authentication method |
| **Respond** | Immediately, When Last Node Finishes, Using 'Respond to Webhook' Node, Streaming | Response mode |

### Authentication Methods — Detailed

#### None
No authentication applied. NEVER use for public-facing webhooks that receive sensitive data.

#### Basic Auth
- Requires a **Basic Auth** credential in n8n
- Credential fields: `username`, `password`
- Client sends `Authorization: Basic <base64(user:pass)>` header
- n8n validates against stored credential
- Returns 401 Unauthorized on failure

#### Header Auth
- Requires a **Header Auth** credential in n8n
- Credential fields: `header name`, `header value`
- Client sends the custom header with the expected value
- Common pattern: `X-API-Key: <secret-value>`
- Returns 401 Unauthorized on mismatch

#### JWT Auth
- Requires a **JWT** credential in n8n
- Credential fields vary by algorithm:
  - **HMAC (HS256/HS384/HS512)**: shared secret
  - **RSA (RS256/RS384/RS512)**: public key (PEM format)
- n8n validates the JWT signature and expiration
- Decoded JWT payload is available in the workflow data
- Returns 401 Unauthorized on invalid/expired token

### Additional Options — Detailed

#### Binary Data
- **Default**: Disabled
- **When enabled**: Webhook accepts binary payloads (files, images, audio)
- **Multipart handling**: Each file field becomes a separate binary property
- **Access**: `$binary.<fieldName>` in expressions, `$input.first().binary` in Code node
- ALWAYS enable when the webhook will receive file uploads

#### IP Whitelist
- **Format**: Comma-separated IP addresses
- **Behavior**: Only listed IPs can call the webhook
- **Rejection**: Returns HTTP 403 Forbidden to unlisted IPs
- **IPv6**: Supported
- ALWAYS use when caller IPs are predictable

#### CORS (Cross-Origin Resource Sharing)
- **Default**: `*` (all origins)
- **Format**: Comma-separated origin URLs
- **Purpose**: Controls which browser-based applications can call the webhook
- **Production**: ALWAYS restrict to specific domains
- Does NOT affect server-to-server calls

#### Ignore Bots
- **Default**: Disabled
- **When enabled**: Filters out requests from known bot user agents
- ALWAYS enable for webhooks exposed to the public internet

#### Raw Body
- **Default**: Disabled
- **When enabled**: Passes the request body as-is without JSON parsing
- **Use for**: XML payloads, custom binary protocols, pre-signed data
- The raw body is available as a string in the output

#### Response Headers
- **Format**: Key-value pairs
- **Purpose**: Add custom headers to the webhook response
- **Common headers**:
  - `Content-Type: application/xml`
  - `Cache-Control: no-cache`
  - `X-Request-Id: {{$execution.id}}`

#### Response Code
- **Default**: 200
- **Purpose**: Override the HTTP status code returned to the caller
- **Common codes**: 200 (OK), 201 (Created), 202 (Accepted), 204 (No Content)

#### Property Name
- **Purpose**: Return only a specific key from the JSON data instead of the full object
- **Use for**: Filtering response data to return only what the caller needs

## Response Modes — Detailed

### Immediately
- Returns HTTP response immediately after receiving the request
- Response body: `{ "message": "Workflow got started" }`
- Status code: 200 (or custom via Response Code option)
- Workflow continues executing asynchronously
- ALWAYS use when the caller does not need the processing result

### When Last Node Finishes
- Waits for the entire workflow to complete before responding
- The response contains data from the last node in the workflow
- **Response Data sub-options**:
  - **All Entries**: Returns an array of all output items from the last node
  - **First Entry JSON**: Returns the JSON of the first output item
  - **First Entry Binary**: Returns a binary file from the first output item
  - **No Response Body**: Returns empty body with status code only
- NEVER use for long-running workflows (HTTP timeout risk)

### Using 'Respond to Webhook' Node
- Full control over response content and timing
- Requires a Respond to Webhook node in the workflow
- The response is sent when the Respond to Webhook node executes
- Workflow can continue executing AFTER the response is sent
- ALWAYS use when response must be crafted from intermediate processing results

### Streaming
- Returns data in real-time as it becomes available
- Requires streaming-capable nodes in the workflow (e.g., AI/LLM nodes)
- Uses chunked transfer encoding
- ALWAYS use for AI-powered workflows returning generated text

## Respond to Webhook Node — Response Types Detailed

### All Incoming Items
- Returns the complete array of JSON items from the input
- Wraps items in a JSON array
- Response `Content-Type`: `application/json`

### First Incoming Item
- Returns only the first item's JSON data
- Single JSON object (not array)
- Response `Content-Type`: `application/json`

### JSON
- Returns a custom JSON body defined in the **Response Body** field
- Use expressions to build dynamic responses
- Response `Content-Type`: `application/json`
- Example: `{ "status": "ok", "id": "{{ $json.id }}" }`

### Text
- Returns plain text or HTML content
- Default `Content-Type`: `text/html`
- Since v1.103.0: HTML is wrapped in a sandboxed `<iframe>`
- Override Content-Type via Response Headers if needed

### Binary File
- Returns a binary file from the input
- Set the **Input Field Name** to the binary property name
- Browser receives a file download
- Content-Type is auto-detected from the file's MIME type

### JWT Token
- Returns a signed JSON Web Token
- Configure: payload fields, secret/key, algorithm, expiration
- Response `Content-Type`: `application/json`
- Returns: `{ "token": "<signed-jwt>" }`

### Redirect
- Sends an HTTP redirect response (302)
- Set the **Redirect URL** field
- Browser/client follows the redirect automatically

### No Data
- Returns an empty response body
- Status code: 200 (or custom)
- Use for fire-and-forget acknowledgments

## Webhook Waiting Path

The `webhook-waiting` path prefix (`N8N_ENDPOINT_WEBHOOK_WAIT`) is used for workflows that use the **Wait** node:
- When a workflow pauses at a Wait node, it generates a waiting webhook URL
- External systems call this URL to resume the workflow
- Format: `<base>/webhook-waiting/<execution-id>`
- NEVER confuse with regular webhook paths

## Environment Variables Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `WEBHOOK_URL` | — | Public URL for webhook routing (reverse proxy) |
| `N8N_ENDPOINT_WEBHOOK` | `webhook` | Production webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_TEST` | `webhook-test` | Test webhook path prefix |
| `N8N_ENDPOINT_WEBHOOK_WAIT` | `webhook-waiting` | Waiting webhook path prefix |
| `N8N_PAYLOAD_SIZE_MAX` | 16MB | Maximum request payload size |
| `N8N_FORMDATA_FILE_SIZE_MAX` | — | Maximum form data file size |
| `N8N_DISABLE_PRODUCTION_MAIN_PROCESS` | `false` | Offload webhooks to separate process |
