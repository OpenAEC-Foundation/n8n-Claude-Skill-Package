# HTTP Request Node Methods Reference

## HTTP Request Node — Complete Options

### Authentication Section

| Option | Values | Description |
|--------|--------|-------------|
| Authentication | None, Predefined Credential Type, Generic Credential | How the request authenticates |
| Credential Type | 400+ service-specific types | Select when using Predefined Credential |
| Generic Credential Type | Header Auth, Query Auth, Basic Auth, Digest Auth, OAuth1 API, OAuth2 API, Custom Auth | Select when using Generic Credential |

### Request Configuration

| Parameter | Description |
|-----------|-------------|
| Method | GET, POST, PUT, PATCH, DELETE, HEAD |
| URL | Full endpoint URL (supports expressions) |
| Send Headers | Enable to add custom request headers |
| Send Query Parameters | Enable to add URL query parameters |
| Send Body | Enable to configure request body (not available for GET/HEAD) |

### Body Content Types

#### JSON
```
Content-Type: application/json
```
- **Using Fields Below**: Add key-value pairs via UI
- **Using JSON**: Provide raw JSON string or expression
- Supports nested objects and arrays

#### Form URL-Encoded
```
Content-Type: application/x-www-form-urlencoded
```
- Key-value pairs sent as URL-encoded form data
- Used for traditional web form submissions

#### Form-Data (Multipart)
```
Content-Type: multipart/form-data
```
- Supports file uploads alongside text fields
- Set parameter type to "File" for binary uploads
- Reference binary data from previous nodes

#### Binary
- Sends raw binary data as request body
- Input Binary Field: name of binary property to send
- Content-Type set from binary data MIME type

#### Raw
- Send arbitrary content (XML, GraphQL, plain text)
- ALWAYS set Content-Type header manually
- Useful for SOAP APIs or GraphQL endpoints

#### n8n Binary File
- Sends binary file from workflow data
- Automatically sets Content-Type from file MIME type

### Response Configuration

| Parameter | Values | Description |
|-----------|--------|-------------|
| Response Format | Autodetect, File, JSON, Text | How to parse the response |
| Output Binary Property | (string) | Property name for File response format |
| Put Output in Field | (string) | Custom field name for response data |

### Options (Advanced)

| Option | Default | Description |
|--------|---------|-------------|
| Batching | Off | Process items in batches with configurable size and delay |
| Ignore SSL Issues | false | Skip SSL certificate validation |
| Redirects | Follow (default) | Follow All Redirects, or Do Not Follow |
| Response Headers | Excluded | Include response headers in output |
| Proxy | (none) | HTTP proxy URL |
| Timeout | 300000ms | Request timeout in milliseconds |
| Pagination | Off | Enable automatic pagination |

### Retry Configuration (Node Settings Tab)

| Setting | Default | Description |
|---------|---------|-------------|
| Retry On Fail | false | Enable automatic retries |
| Max Tries | 3 | Number of retry attempts |
| Wait Between Tries | 1000ms | Delay between retries |

### Continue On Fail (Node Settings Tab)

| Setting | Default | Description |
|---------|---------|-------------|
| Continue On Fail | Off | Options: Off, Use Error Output, Use Default Value |

---

## OAuth2 Flow — Step by Step

### Step 1: Register Application with API Provider

1. Go to the API provider's developer portal
2. Create a new application/project
3. Note the **Client ID** and **Client Secret**
4. Set the redirect/callback URI to: `https://<your-n8n-host>/rest/oauth2-credential/callback`

### Step 2: Create OAuth2 Credential in n8n

1. Go to **Credentials** in n8n sidebar
2. Click **Add Credential**
3. Search for "OAuth2 API" (generic) or the specific service name
4. Fill in:

| Field | Value |
|-------|-------|
| Client ID | From API provider |
| Client Secret | From API provider |
| Authorization URL | Provider's `/authorize` endpoint |
| Access Token URL | Provider's `/token` endpoint |
| Scope | Space-separated permissions (e.g., `read write`) |
| Auth URI Query Parameters | Additional params if required by provider |
| Authentication | Header (default) or Body |
| Grant Type | Authorization Code (default) or Client Credentials |

### Step 3: Authorize

1. Click **Connect my account** button in the credential form
2. Browser redirects to the API provider's authorization page
3. Grant access to the requested scopes
4. Provider redirects back to n8n's callback URL
5. n8n stores the access token and refresh token (encrypted)

### Step 4: Token Lifecycle

- n8n automatically refreshes expired access tokens using the stored refresh token
- If the refresh token expires, re-authorize by clicking **Connect my account** again
- NEVER manually manage token storage or refresh logic in workflows

### Client Credentials Grant

For server-to-server authentication (no user interaction):

1. Set **Grant Type** to "Client Credentials"
2. Provide Client ID, Client Secret, and Access Token URL
3. No authorization URL or callback needed
4. n8n fetches tokens automatically

---

## Pagination Options — Complete Reference

### Mode: Response Contains Next URL

| Setting | Description |
|---------|-------------|
| Next URL | Expression to extract next page URL from response (e.g., `{{ $response.body.next }}`) |
| Maximum Pages | Hard limit on pages to fetch (ALWAYS set this) |
| Complete When | Expression that returns true when pagination should stop |

**How it works:**
1. n8n sends initial request
2. Extracts next URL from response using your expression
3. Sends request to next URL
4. Repeats until Complete When is true or Maximum Pages reached
5. All results are merged into a single output

### Mode: Specify How Each Page Is Determined

| Setting | Description |
|---------|-------------|
| Parameters | Key-value pairs sent with each request, with update expressions |
| Page Size | Number of results per page |
| Maximum Pages | Hard limit on pages to fetch |
| Complete When | Stop condition expression |

**Parameter update expressions** use `$response` and `$pageCount`:
- Offset-based: `{{ $response.body.offset + $response.body.limit }}`
- Page number: `{{ $pageCount + 1 }}`
- Cursor-based: `{{ $response.body.cursor }}`

### Available Variables in Pagination Expressions

| Variable | Description |
|----------|-------------|
| `$response.body` | Parsed response body of current page |
| `$response.headers` | Response headers of current page |
| `$response.statusCode` | HTTP status code of current page |
| `$pageCount` | Number of pages fetched so far (starts at 0) |

---

## Generic Credential Types — Configuration

### Header Auth

| Field | Description |
|-------|-------------|
| Name | Header name (e.g., `X-API-Key`, `Authorization`) |
| Value | Header value (e.g., `Bearer sk-abc123`) |

### Query Auth

| Field | Description |
|-------|-------------|
| Name | Query parameter name (e.g., `api_key`, `token`) |
| Value | Query parameter value |

### Basic Auth

| Field | Description |
|-------|-------------|
| Username | HTTP Basic username |
| Password | HTTP Basic password |

### Custom Auth

| Field | Description |
|-------|-------------|
| Headers | Custom key-value header pairs |
| Query String | Custom key-value query parameters |
| Body | Custom key-value body fields |

Custom Auth allows combining multiple injection points in a single credential.

---

## $response Variable Reference

Available ONLY within the HTTP Request node context (including pagination expressions):

```
$response.body          → Parsed response body (object or string)
$response.headers       → Response headers (object)
$response.statusCode    → HTTP status code (number)
$response.statusMessage → Status message (string, optional)
```

NEVER use `$response` outside the HTTP Request node — it is undefined in other node contexts.
