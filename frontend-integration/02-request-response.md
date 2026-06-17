# Request & Response Contract

## Request

### HTTP method and URL

```
POST https://totaljsbackend.com/api/
```

Always use one API endpoint. The path is project-specific:

- Generic examples often use `POST /api/`
- Projects with `ROUTE('API / ...')` use `POST /`

Keep the path configurable, for example with `EXPO_PUBLIC_API_PATH`.

### Headers

| Header | Required | Value |
|--------|----------|-------|
| `Content-Type` | Always | `application/json` |
| `x-token` | For protected schemas | Session token obtained at login |

`x-token` is the portable baseline. Some projects also accept `token: <session_token>` and `Authorization: Bearer <session_token>` for compatibility with middleware or file services. It is safe for a mobile client to send all three headers for protected schemas when the backend supports them.

Do not attach tokens to known public schemas if the app can avoid it. Keep a client-side anonymous allowlist based on the backend team's public schema contract and verified route behavior.

### Body envelope

```json
{
  "schema": "<schema_string>",
  "data":   { }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schema` | string | Always | The operation address. May include `/{id}` and `?query=params`. |
| `data` | object | When the operation needs input | Omit entirely for operations that take no parameters (e.g. list, read). |

### Examples

**List — no data:**
```json
{ "schema": "posts_list" }
```

**List with pagination — params in schema string:**
```json
{ "schema": "posts_list?page=2&limit=20&status=published" }
```

**Read one — ID in schema string:**
```json
{ "schema": "posts_read/abc123" }
```

**Create — payload in data:**
```json
{
  "schema": "posts_create",
  "data": {
    "title": "Hello World",
    "body": "Content here",
    "status": "draft"
  }
}
```

**Update — ID in schema, fields in data:**
```json
{
  "schema": "posts_update/abc123",
  "data": { "title": "Updated Title", "status": "published" }
}
```

**Delete — no data:**
```json
{ "schema": "posts_remove/abc123" }
```

**Search — params in schema, query in data:**
```json
{
  "schema": "posts_search?limit=10&mode=semantic",
  "data": { "query": "how to get started" }
}
```

### Optional GET helper

Some Total.js projects support `GET /?schema=<schema_string>` for read-only helper calls. Treat this as project-specific convenience. The durable contract remains the JSON envelope:

```text
GET /?schema=products_smart_list%3Flimit%3D20
```

---

## Response

### HTTP status codes

| Status | Meaning |
|--------|---------|
| `200` | Request was processed. Check `success` field to know if the operation succeeded. |
| `401` | Token missing, expired, or invalid. Clear the stored token and go to login. |
| `403` | Authenticated but not authorized for this schema. |
| `404` | Endpoint not found (usually misconfigured base URL, not a missing resource). |
| `500` | Server error. |

> Total.js returns HTTP `200` even for business-level failures. Always check `success: true/false` in the body.

### Success response — object format

```json
{
  "success": true,
  "value": { }
}
```

### Success response — array format

Some schemas (particularly login) return an array:

```json
[
  {
    "success": true,
    "value": { },
    "token": "<session_token>"
  }
]
```

**Always handle both.** Write a normalizer that collapses `response[0]` when the response is an array:

```typescript
const item = Array.isArray(response) ? response[0] : response;
```

Production clients should centralize this normalization:

- Collapse one-item transport arrays.
- Throw when `success === false` or `error` exists.
- Return `value` when present.
- Preserve a root `token` from login responses.
- Accept raw arrays and `{ items: [] }` for list helpers.

### `value` by operation type

| Operation | `value` |
|-----------|---------|
| `*_list` | Array of records |
| `*_read/{id}` | Single record object |
| `*_create` / `*_insert` | Created record or its ID |
| `*_update/{id}` | Updated record |
| `*_remove/{id}` | `true` or `null` |
| `account_login` | Session token string (or nested in response root as `token`) |
| `account` (profile) | Current user object |

### Paginated response

```json
{
  "success": true,
  "value": [ ],
  "meta": {
    "total": 250,
    "page": 2,
    "limit": 20
  }
}
```

---

## Error response

### Business-level error (HTTP 200)

```json
{
  "success": false,
  "error": "Title is required",
  "code": "validation_error"
}
```

Or in array format:
```json
[{ "success": false, "error": "Invalid credentials" }]
```

### Error fields

| Field | Type | Description |
|-------|------|-------------|
| `success` | boolean | Always `false` on error |
| `error` | string | Human-readable error message |
| `code` | string | Optional machine-readable error code |

---

## Unified error extractor

Because responses can be objects or arrays, and errors can come from the HTTP layer or the application layer, you need a single normalizer. This is the reference implementation — adapt it to your language:

```typescript
function extractErrorMessage(error: unknown): string {
  // Total.js array response: [{ "error": "..." }]
  if (Array.isArray(error)) {
    const first = (error as any[])[0];
    return first?.error || first?.value || 'An error occurred';
  }

  // Axios/fetch wrapping a Total.js array body
  if ((error as any)?.response?.data) {
    const d = (error as any).response.data;
    if (Array.isArray(d)) return d[0]?.error || 'An error occurred';
    return d?.error || 'An error occurred';
  }

  // Standard Error object
  if ((error as any)?.message) return (error as any).message;

  // Plain string thrown
  if (typeof error === 'string') return error;

  return 'An unexpected error occurred';
}
```

---

## File uploads

File uploads do not go through `/api/`. Total.js projects typically use a separate **file transfer service** (often running on a subdomain like `fs.totaljsbackend.com`).

The two-step pattern:

```
Step 1 — Upload file:
  POST https://fs.totaljsbackend.com/upload/{bucket}/
      ?token=<upload_service_token>&hostname=1
  Content-Type: multipart/form-data
  Body: file field

Step 2 — Register in the app:
  POST https://totaljsbackend.com/api/
  { "schema": "documents_create", "data": { id, url, name, size, type } }
```

### Upload response shape

```json
{
  "id": "file_abc123",
  "url": "https://fs.totaljsbackend.com/file/...",
  "name": "report.pdf",
  "size": 204800,
  "type": "application/pdf"
}
```

The upload service token is separate from the session token — it is a static credential provided by the backend team, scoped to the upload service only.

React Native upload clients should support these project variants:

- Upload URL contains `{id}` or `{0}` placeholder: replace it with active business id, user id, or `anonymous`.
- Upload URL has no placeholder: append the bucket id as the final path segment.
- Upload token with no configured header: add `?token=<upload_token>` unless already present.
- Upload token with configured header: send it in that header, using `Bearer <token>` when the header is `Authorization`.
- No upload token: send the user session token as `token`, `x-token`, and `Authorization`.

Normalize relative upload response URLs into absolute URLs using the file service origin.

---

## WebSocket (real-time)

When the backend exposes real-time events, they come through a WebSocket endpoint. The URL convention varies by project, but the auth pattern is consistent — pass the session token as a query parameter:

```
wss://totaljsbackend.com/ws/
    ?token=<session_token>&...
```

Or with a dedicated API key (for public/webhook-style integrations):

```
wss://totaljsbackend.com/ws/{resource_id}
    ?apikey=<api_key>&token=<session_token>
```

Incoming messages are JSON objects. The shape is project-defined, but always expect a `type` discriminator field:

```json
{ "type": "message", "data": { ... } }
{ "type": "status",  "state": "connected" }
{ "type": "error",   "message": "..." }
```

Always reconnect with exponential backoff on unexpected close. See the platform integration guides for complete WebSocket hook implementations.
