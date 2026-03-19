# Credential Examples

## Example 1: API Key Credential (Header Injection)

The most common credential pattern. Injects an API key as a Bearer token in the Authorization header.

```typescript
import type {
  IAuthenticate,
  ICredentialTestRequest,
  ICredentialType,
  INodeProperties,
} from 'n8n-workflow';

export class ExampleApi implements ICredentialType {
  name = 'exampleApi';
  displayName = 'Example API';
  documentationUrl = 'https://docs.example.com/api/auth';

  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
      required: true,
      description: 'API key from your Example dashboard under Settings > API Keys',
    },
    {
      displayName: 'Base URL',
      name: 'baseUrl',
      type: 'string',
      default: 'https://api.example.com/v1',
      placeholder: 'https://api.example.com/v1',
      description: 'Base URL of the Example API',
    },
  ];

  authenticate: IAuthenticate = {
    type: 'generic',
    properties: {
      headers: {
        Authorization: '=Bearer {{$credentials.apiKey}}',
      },
    },
  };

  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials.baseUrl}}',
      url: '/me',
    },
  };
}
```

## Example 2: API Key in Query String

Some APIs require the key as a URL parameter instead of a header.

```typescript
export class WeatherApi implements ICredentialType {
  name = 'weatherApi';
  displayName = 'Weather API';

  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];

  authenticate: IAuthenticate = {
    type: 'generic',
    properties: {
      qs: {
        appid: '={{$credentials.apiKey}}',
      },
    },
  };

  test: ICredentialTestRequest = {
    request: {
      baseURL: 'https://api.openweathermap.org',
      url: '/data/2.5/weather',
      qs: {
        q: 'London',
      },
    },
  };
}
```

## Example 3: Basic Auth Credential

Username and password sent as HTTP Basic Authentication.

```typescript
export class JiraApi implements ICredentialType {
  name = 'jiraApi';
  displayName = 'Jira API';
  documentationUrl = 'https://docs.atlassian.com/jira/REST/latest/';

  properties: INodeProperties[] = [
    {
      displayName: 'Email',
      name: 'username',
      type: 'string',
      default: '',
      placeholder: 'user@company.com',
    },
    {
      displayName: 'API Token',
      name: 'password',
      type: 'string',
      typeOptions: { password: true },
      default: '',
      description: 'API token from https://id.atlassian.com/manage-profile/security/api-tokens',
    },
    {
      displayName: 'Domain',
      name: 'domain',
      type: 'string',
      default: '',
      placeholder: 'your-company.atlassian.net',
    },
  ];

  authenticate: IAuthenticate = {
    type: 'generic',
    properties: {
      auth: {
        username: '={{$credentials.username}}',
        password: '={{$credentials.password}}',
      },
    },
  };

  test: ICredentialTestRequest = {
    request: {
      baseURL: '=https://{{$credentials.domain}}',
      url: '/rest/api/3/myself',
    },
  };
}
```

## Example 4: Body Injection Credential

Credentials sent in the request body (common for legacy APIs and some login endpoints).

```typescript
export class LegacySystemApi implements ICredentialType {
  name = 'legacySystemApi';
  displayName = 'Legacy System API';

  properties: INodeProperties[] = [
    {
      displayName: 'Username',
      name: 'username',
      type: 'string',
      default: '',
    },
    {
      displayName: 'Password',
      name: 'password',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
    {
      displayName: 'Instance URL',
      name: 'instanceUrl',
      type: 'string',
      default: '',
      placeholder: 'https://legacy.company.com',
    },
  ];

  authenticate: IAuthenticate = {
    type: 'generic',
    properties: {
      body: {
        username: '={{$credentials.username}}',
        password: '={{$credentials.password}}',
      },
    },
  };

  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials.instanceUrl}}',
      url: '/api/auth/verify',
      method: 'POST',
    },
  };
}
```

## Example 5: OAuth2 Credential (Authorization Code Grant)

Full OAuth2 credential extending the built-in `oAuth2Api` base.

```typescript
import type { ICredentialType, INodeProperties } from 'n8n-workflow';

export class GoogleSheetsOAuth2Api implements ICredentialType {
  name = 'googleSheetsOAuth2Api';
  displayName = 'Google Sheets OAuth2 API';
  extends = ['oAuth2Api'];
  documentationUrl = 'https://docs.n8n.io/integrations/builtin/credentials/google/';

  properties: INodeProperties[] = [
    {
      displayName: 'Grant Type',
      name: 'grantType',
      type: 'hidden',
      default: 'authorizationCode',
    },
    {
      displayName: 'Authorization URL',
      name: 'authUrl',
      type: 'hidden',
      default: 'https://accounts.google.com/o/oauth2/v2/auth',
    },
    {
      displayName: 'Access Token URL',
      name: 'accessTokenUrl',
      type: 'hidden',
      default: 'https://oauth2.googleapis.com/token',
    },
    {
      displayName: 'Scope',
      name: 'scope',
      type: 'hidden',
      default: 'https://www.googleapis.com/auth/spreadsheets',
    },
    {
      displayName: 'Auth URI Query Parameters',
      name: 'authQueryParameters',
      type: 'hidden',
      default: 'access_type=offline&prompt=consent',
    },
    {
      displayName: 'Authentication',
      name: 'authentication',
      type: 'hidden',
      default: 'body',
    },
  ];
}
```

Key points:
- `extends = ['oAuth2Api']` inherits clientId, clientSecret, and token management
- Override specific properties (URLs, scopes) using `type: 'hidden'`
- n8n handles the entire OAuth2 flow (redirect, token exchange, refresh)
- The callback URL is provided by n8n automatically

## Example 6: Credential Test with SSL Skip

For self-hosted services that may use self-signed certificates.

```typescript
test: ICredentialTestRequest = {
  request: {
    baseURL: '={{$credentials?.baseUrl}}',
    url: '/api/v1/health',
    method: 'GET',
    skipSslCertificateValidation:
      '={{$credentials?.allowUnauthorizedCerts}}' as unknown as boolean,
  },
};
```

## Example 7: Custom Header Name Credential

Some APIs use custom header names instead of standard Authorization.

```typescript
export class CustomHeaderApi implements ICredentialType {
  name = 'customHeaderApi';
  displayName = 'Custom Header API';

  properties: INodeProperties[] = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
    },
  ];

  authenticate: IAuthenticate = {
    type: 'generic',
    properties: {
      headers: {
        'X-Api-Key': '={{$credentials.apiKey}}',
      },
    },
  };
}
```

## Example 8: Multiple Authentication Methods in One Node

A node that supports both API key and OAuth2 authentication.

```typescript
// In the NODE description (not credential):
{
  displayName: 'Authentication',
  name: 'authentication',
  type: 'options',
  options: [
    { name: 'API Key', value: 'apiKey' },
    { name: 'OAuth2', value: 'oAuth2' },
  ],
  default: 'apiKey',
}

// Credential bindings:
credentials: [
  {
    name: 'myServiceApi',
    required: true,
    displayOptions: {
      show: { authentication: ['apiKey'] },
    },
  },
  {
    name: 'myServiceOAuth2Api',
    required: true,
    displayOptions: {
      show: { authentication: ['oAuth2'] },
    },
  },
],
```

## Example 9: Credential with Conditional Fields

Show different fields based on authentication method selection within the credential itself.

```typescript
export class FlexibleApi implements ICredentialType {
  name = 'flexibleApi';
  displayName = 'Flexible API';

  properties: INodeProperties[] = [
    {
      displayName: 'Auth Method',
      name: 'authMethod',
      type: 'options',
      options: [
        { name: 'API Key', value: 'apiKey' },
        { name: 'Service Account', value: 'serviceAccount' },
      ],
      default: 'apiKey',
    },
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string',
      typeOptions: { password: true },
      default: '',
      displayOptions: {
        show: { authMethod: ['apiKey'] },
      },
    },
    {
      displayName: 'Service Account JSON',
      name: 'serviceAccountJson',
      type: 'string',
      typeOptions: { password: true },
      default: '',
      displayOptions: {
        show: { authMethod: ['serviceAccount'] },
      },
    },
  ];
}
```

## Example 10: Using Credentials in Node Execute Method

```typescript
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const items = this.getInputData();
  const returnData: INodeExecutionData[] = [];

  // Option A: Use httpRequestWithAuthentication (preferred when authenticate is defined)
  for (let i = 0; i < items.length; i++) {
    try {
      const response = await this.helpers.httpRequestWithAuthentication.call(
        this,
        'exampleApi',
        {
          method: 'GET',
          url: 'https://api.example.com/data',
          qs: { page: i + 1 },
        },
      );
      returnData.push({ json: response as IDataObject });
    } catch (error) {
      if (this.continueOnFail()) {
        returnData.push({ json: { error: (error as Error).message }, pairedItem: { item: i } });
        continue;
      }
      throw error;
    }
  }

  return [returnData];
}

// Option B: Manual credential access (when authenticate is NOT defined)
async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
  const credentials = await this.getCredentials('exampleApi');

  const response = await this.helpers.httpRequest({
    method: 'GET',
    url: `${credentials.baseUrl}/data`,
    headers: {
      Authorization: `Bearer ${credentials.apiKey}`,
    },
  });

  return [this.helpers.returnJsonArray(response as IDataObject[])];
}
```

## Example 11: Complete package.json Registration

```json
{
  "name": "n8n-nodes-myservice",
  "version": "1.0.0",
  "description": "n8n nodes for MyService integration",
  "license": "MIT",
  "keywords": ["n8n-community-node-package"],
  "n8n": {
    "n8nNodesApiVersion": 1,
    "strict": true,
    "credentials": [
      "dist/credentials/MyServiceApi.credentials.js",
      "dist/credentials/MyServiceOAuth2Api.credentials.js"
    ],
    "nodes": [
      "dist/nodes/MyService/MyService.node.js"
    ]
  },
  "files": ["dist"],
  "scripts": {
    "build": "n8n-node build",
    "dev": "n8n-node dev",
    "lint": "n8n-node lint"
  },
  "devDependencies": {
    "@n8n/node-cli": "*",
    "typescript": "5.9.2"
  },
  "peerDependencies": {
    "n8n-workflow": "*"
  }
}
```
