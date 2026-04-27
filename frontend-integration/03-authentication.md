# Authentication & Authorization

## Model

Total.js API Routing uses **session token authentication**. After a successful login or registration, the server returns an opaque token string. The client stores this token and injects it into every subsequent request via the `x-token` header.

There are no cookies, no JWTs with decodable claims, no refresh tokens in the standard setup — just one opaque token the server validates.

---

## Token lifecycle

```
1.  POST /api/ { schema: "account_login", data: { email, password } }
           ↓
2.  Server returns session token
           ↓
3.  Client stores token (localStorage / SecureStore / flutter_secure_storage)
           ↓
4.  Every subsequent request:  x-token: <token>
           ↓
5.  Server responds with data (authorized) or HTTP 401 (expired/invalid)
           ↓
6.  On 401 → clear stored token → redirect to login screen
           ↓
7.  On logout: POST /api/ { schema: "account_logout" } → clear stored token
```

The 401 handler lives in the HTTP client interceptor, runs globally, and requires no per-call handling.

---

## Login

### Request

```json
POST /api/
{
  "schema": "account_login",
  "data": {
    "email": "user@example.com",
    "password": "hunter2"
  }
}
```

### Success response

Login may return an **array** with one object. Always handle both array and plain-object shapes:

```json
[
  {
    "success": true,
    "token": "9f4a2c1e8b...",
    "value": {
      "id": "usr_abc",
      "name": "Jane Doe",
      "email": "user@example.com",
      "role": "admin"
    }
  }
]
```

Store `token` (or `response[0].token`) as the session token. The user object is in `value`.

### Failure response

```json
[{ "success": false, "error": "Invalid credentials" }]
```

### Client logic

```
item = Array.isArray(response) ? response[0] : response

if item.success === false  → show item.error
if item.success === true   → store item.token, set user = item.value, navigate to app
```

---

## Registration

### Request

```json
POST /api/
{
  "schema": "account_create",
  "data": {
    "name": "Jane Doe",
    "email": "user@example.com",
    "password": "hunter2"
  }
}
```

### Success response

```json
{
  "success": true,
  "value": "<session_token>"
}
```

`value` is the session token string directly. Store it and consider the user authenticated. Then call `account` to hydrate the user object.

---

## Get profile / validate token

Use this on app startup to check whether a stored token is still valid and to populate the user state.

```json
POST /api/
x-token: <stored_token>

{ "schema": "account" }
```

### Success response

```json
{
  "success": true,
  "value": {
    "id": "usr_abc",
    "name": "Jane Doe",
    "email": "user@example.com",
    "role": "admin",
    "sa": false
  }
}
```

If this returns HTTP `401`, the token is expired. Clear it and redirect to login.

---

## Logout

```json
POST /api/
x-token: <token>

{ "schema": "account_logout" }
```

Always clear the stored token client-side regardless of the response. If the server call fails (network error), the token is still cleared — the user is logged out locally.

---

## Startup initialization flow

This is the correct pattern on every platform:

```
App starts
  ↓
Read token from storage
  ├── No token → show login screen, done
  └── Token present → POST /api/ { schema: "account" }
                            ↓
                       success?
                         ├── Yes → set user in state, show app
                         └── No (401 or error) → clear token, show login screen
```

The loading/splash state during this check prevents a flash of the unauthenticated UI.

---

## Password management

### Change password (authenticated)

```json
POST /api/
x-token: <token>

{
  "schema": "account_password",
  "data": {
    "current_password": "hunter2",
    "new_password": "c0rrect-horse"
  }
}
```

### Request a password reset (unauthenticated)

```json
POST /api/
{
  "schema": "account_reset",
  "data": { "email": "user@example.com" }
}
```

The server emails a reset link with a time-limited token.

### Verify email address

```json
POST /api/
{
  "schema": "account_verify",
  "data": { "token": "<email_token_from_link>" }
}
```

---

## OAuth (Google, GitHub, etc.)

### Browser-based flow (web apps)

```
Step 1 — Get OAuth redirect URL from backend:
  POST /api/ { "schema": "account_google?page=dashboard" }
  Response: { "success": true, "value": "https://accounts.google.com/o/oauth2/..." }

Step 2 — Redirect browser to the OAuth provider URL

Step 3 — Provider redirects back to your app with a session ID

Step 4 — Exchange session ID for a backend session token:
  POST /api/ { "schema": "account_oauth", "data": { "sessionid": "<id_from_callback>" } }
  Response: { "success": true, "token": "..." }
```

### Native/mobile flow

When the mobile app obtains the OAuth ID token directly using the platform SDK (Google Sign-In, GitHub app), exchange it directly:

```json
POST /api/
{
  "schema": "account_login_google",
  "data": { "token": "<google_id_token>" }
}
```

```json
POST /api/
{
  "schema": "account_login_github",
  "data": { "token": "<github_access_token>" }
}
```

Response format mirrors standard login — store the returned token.

---

## Two-Factor Authentication (TOTP)

Standard Time-based One-Time Password (TOTP), compatible with Google Authenticator, Authy, 1Password, etc.

### Enable 2FA

**Step 1 — Generate secret:**
```json
POST /api/  (x-token required)
{ "schema": "account_2fa_generate" }
```
Response contains a `qr_uri` (to render as a QR code) and a `secret` (for manual entry). Show both to the user.

**Step 2 — Confirm with a TOTP code:**
```json
POST /api/  (x-token required)
{
  "schema": "account_2fa_enable",
  "data": { "token": "123456" }
}
```

### Disable 2FA

```json
POST /api/  (x-token required)
{ "schema": "account_2fa_disable" }
```

### Verify TOTP during login (if 2FA is enabled)

After the initial `account_login` succeeds and you have a session token, the session may be pending 2FA verification. Verify it:

```json
POST /api/  (x-token: pending session token)
{
  "schema": "account_2fa_verify",
  "data": { "token": "123456" }
}
```

---

## Authorization — roles and permissions

Authorization is **entirely server-side**. The client does not decode the token or evaluate permissions locally.

The server either:
- Returns the requested data → user is authorized
- Returns HTTP `401` → not authenticated
- Returns HTTP `403` or `{ success: false, error: "..." }` → authenticated but not authorized

The user object often contains a role or a boolean flag (e.g., `sa: true` for super-admin). Use these **only for UI decisions** — showing or hiding admin sections, for example. Never use a client-side flag as a security gate.

If a call to a privileged schema returns `401` or `{ success: false }`, show an error or redirect. Never retry without re-authentication.
