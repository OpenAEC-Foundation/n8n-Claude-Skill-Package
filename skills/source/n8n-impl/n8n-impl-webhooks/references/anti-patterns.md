# Webhook Anti-Patterns

## AP-01: Using Test URL in Production

**NEVER** give external services the test URL (`/webhook-test/`).

| Aspect | Problem |
|--------|---------|
| **What happens** | The test URL only works when "Listen for Test Event" is active in the editor |
| **Impact** | External services get HTTP 404 when no one has the editor open |
| **Root cause** | Confusing test URL with production URL |

**Fix**: ALWAYS use the production URL (`/webhook/`) for external integrations. ALWAYS activate the workflow before sharing the URL.

---

## AP-02: Not Setting WEBHOOK_URL Behind Reverse Proxy

**NEVER** omit `WEBHOOK_URL` when n8n runs behind a reverse proxy (Traefik, nginx, Caddy).

| Aspect | Problem |
|--------|---------|
| **What happens** | n8n displays internal URLs (e.g., `http://localhost:5678/webhook/...`) |
| **Impact** | External services cannot reach the webhook |
| **Root cause** | n8n generates URLs from its own hostname without knowing the public URL |

**Fix**: ALWAYS set `WEBHOOK_URL=https://your-public-domain.com/` in the environment.

---

## AP-03: Using GET for Sensitive Data

**NEVER** use the GET method for webhooks that receive sensitive information.

| Aspect | Problem |
|--------|---------|
| **What happens** | Sensitive data appears in URL query parameters |
| **Impact** | Data logged in server access logs, browser history, proxy logs |
| **Root cause** | GET parameters are part of the URL, not the body |

**Fix**: ALWAYS use POST (or PUT/PATCH) for sensitive payloads. Data in the request body is not logged in URLs.

---

## AP-04: No Authentication on Public Webhooks

**NEVER** expose webhooks without authentication when they trigger actions with side effects.

| Aspect | Problem |
|--------|---------|
| **What happens** | Anyone who discovers the URL can trigger the workflow |
| **Impact** | Unauthorized data injection, resource abuse, security breach |
| **Root cause** | Relying on URL obscurity instead of proper authentication |

**Fix**: ALWAYS configure at least one authentication method (Basic Auth, Header Auth, or JWT Auth). Combine with IP Whitelist when possible.

---

## AP-05: Using "When Last Node Finishes" for Long Workflows

**NEVER** use the "When Last Node Finishes" response mode for workflows that take more than 30 seconds.

| Aspect | Problem |
|--------|---------|
| **What happens** | The HTTP connection times out before the workflow completes |
| **Impact** | Caller receives a timeout error; workflow may still complete but response is lost |
| **Root cause** | HTTP clients and proxies have default timeouts (30-120 seconds) |

**Fix**: Use "Immediately" for long-running workflows (caller gets instant acknowledgment). Use a callback pattern or polling endpoint if the caller needs the result.

---

## AP-06: Multiple Respond to Webhook Nodes on Same Path

**NEVER** place multiple Respond to Webhook nodes in the same execution path.

| Aspect | Problem |
|--------|---------|
| **What happens** | Only the FIRST Respond to Webhook node executes; subsequent ones are ignored |
| **Impact** | Unexpected response content, silent data loss |
| **Root cause** | HTTP allows only one response per request |

**Fix**: Use IF/Switch nodes to route to exactly ONE Respond to Webhook node per execution path. Place success and error responses on separate branches.

---

## AP-07: Forgetting to Enable Binary Data for File Uploads

**NEVER** accept file uploads without enabling the Binary Data option.

| Aspect | Problem |
|--------|---------|
| **What happens** | File data is lost or corrupted; only text/JSON metadata is received |
| **Impact** | Missing file content, broken file processing workflows |
| **Root cause** | Binary Data option is disabled by default |

**Fix**: ALWAYS enable "Binary Data" in Additional Options when the webhook receives file uploads (multipart/form-data).

---

## AP-08: Wildcard CORS in Production

**NEVER** use `*` (wildcard) CORS in production webhooks that handle sensitive data.

| Aspect | Problem |
|--------|---------|
| **What happens** | Any website can make cross-origin requests to your webhook |
| **Impact** | Cross-site request forgery potential, data exfiltration risk |
| **Root cause** | Default CORS setting is `*` (all origins allowed) |

**Fix**: ALWAYS set specific origin domains: `https://app.example.com,https://admin.example.com`. Wildcard is acceptable only for truly public, read-only endpoints.

---

## AP-09: Ignoring the 16MB Payload Limit

**NEVER** send payloads larger than 16MB without adjusting the limit.

| Aspect | Problem |
|--------|---------|
| **What happens** | n8n rejects the request with a payload-too-large error |
| **Impact** | Large file uploads or bulk data transfers fail silently |
| **Root cause** | Default `N8N_PAYLOAD_SIZE_MAX` is 16MB |

**Fix**: For large payloads, increase `N8N_PAYLOAD_SIZE_MAX` in the environment. For very large files, use a storage service (S3, GCS) and send only the file URL via the webhook.

---

## AP-10: Not Testing Both Test and Production URLs

**NEVER** assume the production URL works just because the test URL works.

| Aspect | Problem |
|--------|---------|
| **What happens** | Test URL works in the editor, but production URL returns 404 |
| **Impact** | Webhook integration fails after deployment |
| **Root cause** | Workflow not activated, or `WEBHOOK_URL` not configured |

**Fix**: ALWAYS test BOTH URLs. After activating the workflow, verify the production URL with a curl request:
```bash
curl -I https://n8n.example.com/webhook/<path>
```

---

## AP-11: Hardcoding Webhook URLs in External Services

**NEVER** hardcode the full webhook URL including the host in external service configurations without using `WEBHOOK_URL`.

| Aspect | Problem |
|--------|---------|
| **What happens** | When n8n moves to a different domain or port, all integrations break |
| **Impact** | Widespread webhook failures requiring manual updates in every external service |
| **Root cause** | Tight coupling to infrastructure details |

**Fix**: ALWAYS set `WEBHOOK_URL` centrally. When migrating, update only the env var. Document the webhook URL in a central configuration registry.

---

## AP-12: Not Handling Duplicate Webhook Calls

**NEVER** assume webhooks are called exactly once.

| Aspect | Problem |
|--------|---------|
| **What happens** | External services may retry failed calls, or send duplicate events |
| **Impact** | Duplicate processing, duplicate records, duplicate notifications |
| **Root cause** | Network timeouts, retry logic in calling services |

**Fix**: Implement idempotency in the workflow: check for duplicate event IDs before processing, use upsert operations instead of inserts, and return 200 even for already-processed events.

---

## AP-13: Using Webhook Path as Security

**NEVER** rely on an obscure webhook path (e.g., `/webhook/a8f3b2c1d4e5f6`) as the only security measure.

| Aspect | Problem |
|--------|---------|
| **What happens** | URLs can be discovered through logs, referrer headers, or brute force |
| **Impact** | Unauthorized access to the webhook |
| **Root cause** | Security through obscurity is not real security |

**Fix**: ALWAYS combine obscure paths with proper authentication (Header Auth, Basic Auth, or JWT Auth) AND IP Whitelist when possible.

---

## AP-14: Skipping the Respond to Webhook Node Unintentionally

**NEVER** design workflows where the Respond to Webhook node can be bypassed by an IF/Switch without a fallback response.

| Aspect | Problem |
|--------|---------|
| **What happens** | When the node is skipped, n8n returns a generic 200 with a standard message |
| **Impact** | Caller receives an unexpected response format, breaking their integration |
| **Root cause** | Missing response branch for edge cases |

**Fix**: ALWAYS ensure every possible execution path through an IF/Switch leads to a Respond to Webhook node. Add a default/fallback branch that returns an appropriate error response.
