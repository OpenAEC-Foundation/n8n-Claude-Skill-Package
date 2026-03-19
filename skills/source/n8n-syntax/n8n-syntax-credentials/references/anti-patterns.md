# Credential Anti-Patterns

## AP-1: Missing password typeOption on sensitive fields

**WRONG:**
```typescript
{
  displayName: 'API Key',
  name: 'apiKey',
  type: 'string',
  default: '',
}
```

**RIGHT:**
```typescript
{
  displayName: 'API Key',
  name: 'apiKey',
  type: 'string',
  typeOptions: { password: true },
  default: '',
}
```

**Why**: Without `password: true`, the API key is visible in plain text in the GUI. ALWAYS use `typeOptions: { password: true }` for API keys, tokens, passwords, secrets, and private keys.

---

## AP-2: Missing expression prefix in authenticate

**WRONG:**
```typescript
authenticate: IAuthenticate = {
  type: 'generic',
  properties: {
    headers: {
      Authorization: 'Bearer {{$credentials.apiKey}}',  // Missing '=' prefix
    },
  },
};
```

**RIGHT:**
```typescript
authenticate: IAuthenticate = {
  type: 'generic',
  properties: {
    headers: {
      Authorization: '=Bearer {{$credentials.apiKey}}',  // '=' prefix required
    },
  },
};
```

**Why**: Without the `=` prefix, n8n treats the value as a literal string. The literal text `Bearer {{$credentials.apiKey}}` is sent instead of the resolved credential value. This causes authentication failures that are difficult to debug.

---

## AP-3: Pointing to .ts files in package.json

**WRONG:**
```json
{
  "n8n": {
    "credentials": [
      "credentials/MyApi.credentials.ts"
    ]
  }
}
```

**RIGHT:**
```json
{
  "n8n": {
    "credentials": [
      "dist/credentials/MyApi.credentials.js"
    ]
  }
}
```

**Why**: n8n loads compiled JavaScript at runtime, not TypeScript source. Pointing to `.ts` files causes n8n to fail silently -- the credential type is not registered and nodes that require it cannot authenticate.

---

## AP-4: Using authenticate AND manual credential injection

**WRONG:**
```typescript
// In credential file:
authenticate: IAuthenticate = {
  type: 'generic',
  properties: {
    headers: { Authorization: '=Bearer {{$credentials.apiKey}}' },
  },
};

// In node execute():
const credentials = await this.getCredentials('myApi');
const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'myApi', {
    method: 'GET',
    url: 'https://api.example.com/data',
    headers: {
      Authorization: `Bearer ${credentials.apiKey}`,  // DUPLICATE -- already injected
    },
  },
);
```

**RIGHT (Option A -- use authenticate):**
```typescript
const response = await this.helpers.httpRequestWithAuthentication.call(
  this, 'myApi', {
    method: 'GET',
    url: 'https://api.example.com/data',
    // No manual auth header -- authenticate handles it
  },
);
```

**RIGHT (Option B -- manual injection, no authenticate on credential):**
```typescript
const credentials = await this.getCredentials('myApi');
const response = await this.helpers.httpRequest({
  method: 'GET',
  url: 'https://api.example.com/data',
  headers: {
    Authorization: `Bearer ${credentials.apiKey}`,
  },
});
```

**Why**: Double injection can cause malformed headers or duplicate authentication parameters, leading to unpredictable API behavior.

---

## AP-5: Missing credential test endpoint

**WRONG:**
```typescript
export class MyApi implements ICredentialType {
  name = 'myApi';
  displayName = 'My API';
  properties: INodeProperties[] = [ /* ... */ ];
  authenticate: IAuthenticate = { /* ... */ };
  // No test property
}
```

**RIGHT:**
```typescript
export class MyApi implements ICredentialType {
  name = 'myApi';
  displayName = 'My API';
  properties: INodeProperties[] = [ /* ... */ ];
  authenticate: IAuthenticate = { /* ... */ };
  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials.baseUrl}}',
      url: '/api/v1/me',
    },
  };
}
```

**Why**: Without a `test` property, users cannot validate their credentials before using them in workflows. This leads to confusing runtime errors when credentials are misconfigured.

---

## AP-6: Using a data-modifying endpoint for credential testing

**WRONG:**
```typescript
test: ICredentialTestRequest = {
  request: {
    baseURL: '={{$credentials.baseUrl}}',
    url: '/api/v1/records',
    method: 'POST',
    body: { test: true },  // Creates a record every time user tests
  },
};
```

**RIGHT:**
```typescript
test: ICredentialTestRequest = {
  request: {
    baseURL: '={{$credentials.baseUrl}}',
    url: '/api/v1/me',        // Read-only endpoint
    method: 'GET',
  },
};
```

**Why**: Users may click "Test" multiple times during setup. A data-modifying endpoint creates side effects (duplicate records, sent messages, etc.) each time.

---

## AP-7: Hardcoding credential values in node code

**WRONG:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const response = await this.helpers.httpRequest({
    method: 'GET',
    url: 'https://api.example.com/data',
    headers: {
      Authorization: 'Bearer sk-hardcoded-api-key-12345',
    },
  });
  return [this.helpers.returnJsonArray(response)];
}
```

**RIGHT:**
```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const credentials = await this.getCredentials('exampleApi');
  const response = await this.helpers.httpRequest({
    method: 'GET',
    url: 'https://api.example.com/data',
    headers: {
      Authorization: `Bearer ${credentials.apiKey}`,
    },
  });
  return [this.helpers.returnJsonArray(response)];
}
```

**Why**: Hardcoded credentials are stored in plain text in the node source code, visible in version control, and impossible to rotate without code changes. n8n's credential system encrypts values and provides a GUI for management.

---

## AP-8: Wrong extends value for OAuth2

**WRONG:**
```typescript
export class MyOAuth2 implements ICredentialType {
  name = 'myOAuth2';
  extends = ['oauth2'];  // Wrong -- lowercase, not a valid base credential
}
```

**RIGHT:**
```typescript
export class MyOAuth2 implements ICredentialType {
  name = 'myOAuth2';
  extends = ['oAuth2Api'];  // Correct base credential name
}
```

**Why**: The base credential names are specific identifiers. Using `'oauth2'` instead of `'oAuth2Api'` causes n8n to fail to resolve the parent credential, and no OAuth2 properties are inherited.

Valid base credential names: `oAuth2Api`, `oAuth1Api`, `httpBasicAuth`, `httpHeaderAuth`, `httpQueryAuth`, `httpDigestAuth`.

---

## AP-9: Forgetting to set OAuth2 hidden properties

**WRONG:**
```typescript
export class MyOAuth2 implements ICredentialType {
  name = 'myOAuth2';
  extends = ['oAuth2Api'];
  properties: INodeProperties[] = [];  // No URL overrides
}
```

**RIGHT:**
```typescript
export class MyOAuth2 implements ICredentialType {
  name = 'myOAuth2';
  extends = ['oAuth2Api'];
  properties: INodeProperties[] = [
    {
      displayName: 'Authorization URL',
      name: 'authUrl',
      type: 'hidden',
      default: 'https://myservice.com/oauth/authorize',
    },
    {
      displayName: 'Access Token URL',
      name: 'accessTokenUrl',
      type: 'hidden',
      default: 'https://myservice.com/oauth/token',
    },
    {
      displayName: 'Scope',
      name: 'scope',
      type: 'hidden',
      default: 'read write',
    },
  ];
}
```

**Why**: The base `oAuth2Api` credential has empty defaults for authUrl and accessTokenUrl. Without overriding them, users must manually enter these URLs in the credential dialog, which is error-prone and defeats the purpose of a service-specific credential.

---

## AP-10: Credential name mismatch between credential and node

**WRONG:**
```typescript
// Credential file:
export class MyApi implements ICredentialType {
  name = 'myApi';  // Name is 'myApi'
}

// Node file:
credentials: [
  {
    name: 'myAPI',  // Case mismatch -- 'myAPI' vs 'myApi'
    required: true,
  },
],
```

**RIGHT:**
```typescript
// Credential file:
export class MyApi implements ICredentialType {
  name = 'myApi';
}

// Node file:
credentials: [
  {
    name: 'myApi',  // Exact match
    required: true,
  },
],
```

**Why**: The credential `name` is case-sensitive. A mismatch means n8n cannot find the credential type for the node, and the credentials dropdown in the GUI will be empty.

---

## AP-11: Using getCredentials with wrong credential name

**WRONG:**
```typescript
// Credential name is 'exampleApi'
const credentials = await this.getCredentials('ExampleApi');  // Wrong case
```

**RIGHT:**
```typescript
const credentials = await this.getCredentials('exampleApi');  // Exact match
```

**Why**: `getCredentials()` uses the credential's `name` property for lookup, which is case-sensitive. A wrong name throws a runtime error: "No credentials of type 'ExampleApi' exist."

---

## AP-12: Exposing credential values in node output

**WRONG:**
```typescript
const credentials = await this.getCredentials('myApi');
returnData.push({
  json: {
    apiKey: credentials.apiKey,  // Credential value in output data
    result: response.data,
  },
});
```

**RIGHT:**
```typescript
const credentials = await this.getCredentials('myApi');
returnData.push({
  json: {
    result: response.data,  // Only include API response data
  },
});
```

**Why**: Output data is visible in the n8n GUI, stored in execution history, and may be forwarded to other nodes. Including credential values in output exposes secrets in logs and the UI.
