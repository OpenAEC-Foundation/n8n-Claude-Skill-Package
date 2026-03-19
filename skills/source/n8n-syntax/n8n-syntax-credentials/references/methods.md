# ICredentialType Interface & Methods Reference

## ICredentialType Interface (Complete)

```typescript
import type { INodeProperties, Icon } from 'n8n-workflow';

export interface ICredentialType {
  // REQUIRED: Internal identifier used in code references
  name: string;

  // REQUIRED: Human-readable label shown in credentials GUI
  displayName: string;

  // REQUIRED: Array of input fields for credential configuration
  properties: INodeProperties[];

  // Optional: Link to documentation for setting up the credential
  documentationUrl?: string;

  // Optional: How credentials are injected into HTTP requests
  authenticate?: IAuthenticate;

  // Optional: API endpoint to validate credentials
  test?: ICredentialTestRequest;

  // Optional: Inherit properties from another credential type
  extends?: string[];

  // Optional: Icon displayed in the credentials list
  icon?: Icon;

  // Optional: URL-based icon
  iconUrl?: string;

  // Optional: Color for the icon
  iconColor?: string;
}
```

## IAuthenticate Interface

The `authenticate` property defines how credential values are automatically injected into HTTP requests made with `httpRequestWithAuthentication`.

```typescript
export interface IAuthenticate {
  // MUST be 'generic' -- no other value is supported
  type: 'generic';

  properties: {
    // Inject into URL query string parameters
    qs?: Record<string, string>;

    // Inject into HTTP headers
    headers?: Record<string, string>;

    // Inject into request body
    body?: Record<string, string>;

    // Inject as HTTP Basic Authentication (username + password)
    auth?: {
      username: string;
      password: string;
    };
  };
}
```

### Expression Syntax in IAuthenticate

All values in `authenticate.properties` support n8n credential expressions:

| Syntax | Example | Result |
|--------|---------|--------|
| `={{$credentials.fieldName}}` | `={{$credentials.apiKey}}` | Injects the apiKey value |
| `=Prefix {{$credentials.field}}` | `=Bearer {{$credentials.token}}` | Concatenates prefix + value |
| `={{$credentials?.field}}` | `={{$credentials?.domain}}` | Optional chaining (safe access) |

**Rule**: The `=` prefix is REQUIRED for expression evaluation. Without it, the string is treated as a literal.

## ICredentialTestRequest Interface

Defines an HTTP request that n8n executes when users click "Test" in the credentials dialog.

```typescript
export interface ICredentialTestRequest {
  request: {
    // Base URL for the test request
    baseURL?: string;

    // URL path (appended to baseURL)
    url: string;

    // HTTP method (defaults to GET)
    method?: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'HEAD';

    // Additional headers
    headers?: Record<string, string>;

    // Query string parameters
    qs?: Record<string, string>;

    // Request body
    body?: Record<string, unknown>;

    // Skip SSL certificate validation
    skipSslCertificateValidation?: boolean;

    // Request timeout in milliseconds
    timeout?: number;
  };
}
```

**Rule**: The test endpoint MUST be a lightweight, read-only endpoint (e.g., `/me`, `/verify`, `/ping`). NEVER use an endpoint that modifies data.

**Rule**: A 2xx HTTP response = credentials valid. Any other status code = credentials invalid.

**Rule**: The `authenticate` property is automatically applied to the test request -- you do NOT need to manually add auth headers to the test request.

## Credential Property Types (INodeProperties for Credentials)

Credential properties use the same `INodeProperties` interface as node properties, but a subset of types is commonly used:

### string (Most Common)

```typescript
{
  displayName: 'API Key',
  name: 'apiKey',
  type: 'string',
  typeOptions: { password: true },  // Masks input, encrypts at rest
  default: '',
  required: true,
  description: 'Your API key from the dashboard',
}
```

### string (URL field)

```typescript
{
  displayName: 'Base URL',
  name: 'baseUrl',
  type: 'string',
  default: 'https://api.example.com',
  placeholder: 'https://your-instance.example.com',
  description: 'The base URL of your instance',
}
```

### number

```typescript
{
  displayName: 'Port',
  name: 'port',
  type: 'number',
  default: 443,
  typeOptions: { minValue: 1, maxValue: 65535 },
}
```

### boolean

```typescript
{
  displayName: 'Allow Unauthorized SSL',
  name: 'allowUnauthorizedCerts',
  type: 'boolean',
  default: false,
  description: 'Whether to allow connections to servers with self-signed certificates',
}
```

### options (Dropdown)

```typescript
{
  displayName: 'Region',
  name: 'region',
  type: 'options',
  options: [
    { name: 'US East', value: 'us-east-1' },
    { name: 'EU West', value: 'eu-west-1' },
    { name: 'Asia Pacific', value: 'ap-southeast-1' },
  ],
  default: 'us-east-1',
}
```

### hidden (Internal Values)

```typescript
{
  displayName: 'Grant Type',
  name: 'grantType',
  type: 'hidden',
  default: 'authorizationCode',
}
```

Used in OAuth2 credentials to set fixed values that users should NOT modify. The `hidden` type with `typeOptions: { expirable: true }` marks tokens that n8n should auto-refresh.

```typescript
{
  displayName: 'Access Token',
  name: 'accessToken',
  type: 'hidden',
  typeOptions: { expirable: true },
  default: '',
}
```

## displayOptions for Conditional Credential Fields

Show or hide credential fields based on other field values:

```typescript
{
  displayName: 'Private Key',
  name: 'privateKey',
  type: 'string',
  typeOptions: { password: true },
  default: '',
  displayOptions: {
    show: {
      authMethod: ['serviceAccount'],
    },
  },
}
```

## httpRequestWithAuthentication Method

Used in nodes to make authenticated requests using the `authenticate` configuration:

```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this,
  'credentialTypeName',  // Must match the credential's `name` property
  {
    method: 'GET',
    url: 'https://api.example.com/data',
    qs: { limit: 10 },
  },
);
```

This method:
1. Reads the credential values from encrypted storage
2. Applies the `authenticate` configuration (injects qs, headers, body, or auth)
3. Executes the HTTP request
4. Returns the response

## getCredentials Method

Used in nodes to access raw credential values (when NOT using `authenticate`):

```typescript
const credentials = await this.getCredentials('credentialTypeName');
// credentials is a plain object: { apiKey: '...', baseUrl: '...', ... }
```

## Credential Inheritance (extends)

When a credential extends another, it inherits ALL properties from the parent. Common base credentials:

| Base Credential | Purpose | Inherited Properties |
|----------------|---------|---------------------|
| `oAuth2Api` | OAuth2 flow | clientId, clientSecret, accessTokenUrl, authUrl, scope, authentication, grantType |
| `oAuth1Api` | OAuth1 flow | consumerKey, consumerSecret, requestTokenUrl, authorizationUrl, accessTokenUrl, signatureMethod |
| `httpBasicAuth` | Basic auth | user, password |
| `httpHeaderAuth` | Header auth | name, value |
| `httpQueryAuth` | Query auth | name, value |
| `httpDigestAuth` | Digest auth | user, password |

```typescript
// Extending oAuth2Api -- only define ADDITIONAL properties
export class SlackOAuth2Api implements ICredentialType {
  name = 'slackOAuth2Api';
  displayName = 'Slack OAuth2 API';
  extends = ['oAuth2Api'];
  properties: INodeProperties[] = [
    {
      displayName: 'Authorization URL',
      name: 'authUrl',
      type: 'hidden',
      default: 'https://slack.com/oauth/v2/authorize',
    },
    {
      displayName: 'Access Token URL',
      name: 'accessTokenUrl',
      type: 'hidden',
      default: 'https://slack.com/api/oauth.v2.access',
    },
    {
      displayName: 'Scope',
      name: 'scope',
      type: 'hidden',
      default: 'chat:write channels:read',
    },
  ];
}
```

## genericCredentialRequest Pattern

For nodes that support multiple credential types via the HTTP Request node's "Generic Credential Type" feature:

```typescript
// In node description
credentials: [
  {
    name: 'httpHeaderAuth',
    displayOptions: {
      show: { authentication: ['headerAuth'] },
    },
  },
  {
    name: 'httpQueryAuth',
    displayOptions: {
      show: { authentication: ['queryAuth'] },
    },
  },
],
```

This allows users to select different authentication methods in the node UI, each mapping to a different credential type.
