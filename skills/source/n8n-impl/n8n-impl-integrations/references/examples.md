# API Integration Examples

## Example 1: REST API Integration (GET with API Key)

### Scenario
Fetch a list of users from a REST API using an API key in the header.

### Workflow Structure
```
Schedule Trigger → HTTP Request → Set Node (transform) → Google Sheets (store)
```

### HTTP Request Node Configuration
```
Method: GET
URL: https://api.example.com/v1/users
Authentication: Generic Credential → Header Auth
  Header Name: X-API-Key
  Header Value: (stored in credential)
Send Query Parameters: true
  status: active
  limit: 100
Response Format: JSON
```

### Output
Each user object becomes a separate n8n item:
```json
[
  { "json": { "id": 1, "name": "Alice", "email": "alice@example.com" } },
  { "json": { "id": 2, "name": "Bob", "email": "bob@example.com" } }
]
```

---

## Example 2: POST Request with JSON Body

### Scenario
Create a new contact in a CRM API.

### HTTP Request Node Configuration
```
Method: POST
URL: https://api.crm.com/v2/contacts
Authentication: Predefined Credential Type → CRM API
Send Headers: true
  Accept: application/json
Send Body: true
  Body Content Type: JSON
  Specify Body: Using Fields Below
    first_name: {{ $json.firstName }}
    last_name: {{ $json.lastName }}
    email: {{ $json.email }}
    company: {{ $json.company }}
```

### Key Points
- ALWAYS use expressions to map input data to body fields
- NEVER hardcode values that come from previous nodes
- The response (created contact with ID) flows to the next node

---

## Example 3: OAuth2 Flow (Google Sheets API)

### Scenario
Read data from a Google Sheet using OAuth2 authentication.

### Step 1: Create OAuth2 Credential
```
Credential Type: Google Sheets OAuth2 API (predefined)
  Client ID: (from Google Cloud Console)
  Client Secret: (from Google Cloud Console)
  Scope: https://www.googleapis.com/auth/spreadsheets.readonly
```

### Step 2: Register Callback URL in Google Cloud Console
```
Authorized redirect URI: https://your-n8n.example.com/rest/oauth2-credential/callback
```

### Step 3: Authorize in n8n
1. Click "Connect my account" in credential settings
2. Google login page opens
3. Grant access to Google Sheets
4. Redirect back to n8n — credential saved

### Step 4: Use in HTTP Request (if no dedicated node)
```
Method: GET
URL: https://sheets.googleapis.com/v4/spreadsheets/{{ $json.spreadsheetId }}/values/Sheet1!A1:Z1000
Authentication: Predefined Credential Type → Google Sheets OAuth2 API
Response Format: JSON
```

### Better Alternative
ALWAYS use the dedicated Google Sheets node instead of HTTP Request when available. The dedicated node handles pagination, formatting, and error handling automatically.

---

## Example 4: Generic OAuth2 for Custom API

### Scenario
Integrate with a custom SaaS API that uses OAuth2 but has no predefined n8n credential.

### Create Generic OAuth2 Credential
```
Credential Type: OAuth2 API (generic)
  Grant Type: Authorization Code
  Authorization URL: https://auth.customsaas.com/oauth/authorize
  Access Token URL: https://auth.customsaas.com/oauth/token
  Client ID: app_abc123
  Client Secret: (stored securely)
  Scope: read:data write:data
  Authentication: Send In Header (default)
```

### Use in HTTP Request Node
```
Method: GET
URL: https://api.customsaas.com/v2/data
Authentication: Predefined Credential Type → OAuth2 API
Response Format: JSON
```

n8n automatically includes the `Authorization: Bearer <token>` header and refreshes the token when expired.

---

## Example 5: Paginated Fetching (Offset-Based)

### Scenario
Fetch all products from an API that returns 50 items per page with offset pagination.

### HTTP Request Node Configuration
```
Method: GET
URL: https://api.store.com/v1/products
Authentication: Generic Credential → Header Auth
Send Query Parameters: true
  limit: 50
Response Format: JSON

Options:
  Pagination: Specify How Each Page Is Determined
    Parameters:
      Type: Query
      Name: offset
      Value: {{ $pageCount * 50 }}
    Page Size: 50
    Maximum Pages: 20
    Complete When: {{ $response.body.products.length < 50 }}
```

### Result
All pages are fetched and merged. If 3 pages exist (150 products), the output contains 150 items.

---

## Example 6: Paginated Fetching (Cursor-Based)

### Scenario
Fetch records from an API using cursor-based pagination.

### HTTP Request Node Configuration
```
Method: GET
URL: https://api.service.com/v1/records
Authentication: Generic Credential → Header Auth
Send Query Parameters: true
  limit: 100
Response Format: JSON

Options:
  Pagination: Specify How Each Page Is Determined
    Parameters:
      Type: Query
      Name: cursor
      Value: {{ $response.body.nextCursor }}
    Page Size: 100
    Maximum Pages: 50
    Complete When: {{ !$response.body.nextCursor }}
```

---

## Example 7: Paginated Fetching (Link Header / Next URL)

### Scenario
API returns the next page URL in the response body or Link header.

### HTTP Request Node Configuration (Next URL in Body)
```
Method: GET
URL: https://api.github.com/repos/n8n-io/n8n/issues
Authentication: Predefined Credential Type → GitHub API
Response Format: JSON

Options:
  Pagination: Response Contains Next URL
    Next URL: {{ $response.body.next }}
    Maximum Pages: 10
```

### HTTP Request Node Configuration (Next URL in Link Header)
```
Options:
  Pagination: Response Contains Next URL
    Next URL: {{ $response.headers.link?.match(/<([^>]+)>;\s*rel="next"/)?.[1] }}
    Maximum Pages: 10
    Complete When: {{ !$response.headers.link?.includes('rel="next"') }}
```

---

## Example 8: Batch Requests with Rate Limiting

### Scenario
Send 500 items to an API that allows 10 requests per second.

### Workflow Structure
```
Get Data (500 items) → Split In Batches → HTTP Request → Wait → Loop back to Split In Batches
```

### Split In Batches Node
```
Batch Size: 10
```

### HTTP Request Node
```
Method: POST
URL: https://api.service.com/v1/records
Authentication: Generic Credential → Header Auth
Send Body: true
  Body Content Type: JSON
  Specify Body: Using Fields Below
    name: {{ $json.name }}
    value: {{ $json.value }}
```

### Wait Node
```
Resume: After Time Interval
Amount: 1
Unit: Seconds
```

### How It Works
1. Split In Batches sends 10 items at a time
2. HTTP Request processes each of the 10 items (10 parallel calls)
3. Wait node pauses 1 second for rate limit compliance
4. Loop continues with next batch of 10
5. After all 500 items processed, flow exits the loop

---

## Example 9: File Download

### Scenario
Download a PDF report from an API.

### HTTP Request Node Configuration
```
Method: GET
URL: https://api.reports.com/v1/reports/{{ $json.reportId }}/download
Authentication: Predefined Credential Type → Reports API
Response Format: File
Output Binary Property: reportFile
```

### Follow-Up: Save to Disk
```
Write Binary File Node:
  File Name: {{ $json.reportId }}.pdf
  Property Name: reportFile
```

### Follow-Up: Send as Email Attachment
```
Send Email Node:
  Attachments: reportFile
```

---

## Example 10: Error Handling with Branching

### Scenario
Call an API and handle different response codes with appropriate logic.

### Workflow Structure
```
HTTP Request (Continue On Fail: Use Error Output)
  ├── Success output → Process Data → Store Results
  └── Error output → IF Node
                       ├── statusCode 429 → Wait → Retry (loop back)
                       ├── statusCode 404 → Log "Not Found" → Skip
                       └── Other errors → Error Notification (Slack/Email)
```

### HTTP Request Node Settings
```
Continue On Fail: Use Error Output
Retry On Fail: true
Max Tries: 3
Wait Between Tries: 2000
```

### IF Node (on error branch)
```
Condition: {{ $json.statusCode === 429 }}
→ True: Wait 60 seconds, then retry
→ False: Check for 404 or send error notification
```

---

## Example 11: Multipart File Upload

### Scenario
Upload a file to an API endpoint along with metadata.

### Workflow Structure
```
Read Binary File → HTTP Request (POST multipart)
```

### HTTP Request Node Configuration
```
Method: POST
URL: https://api.storage.com/v1/upload
Authentication: Generic Credential → Header Auth
Send Body: true
  Body Content Type: Form-Data (Multipart)
  Parameters:
    - Name: file, Parameter Type: File, Input Binary Field: data
    - Name: description, Parameter Type: String, Value: {{ $json.description }}
    - Name: folder_id, Parameter Type: String, Value: {{ $json.folderId }}
```
