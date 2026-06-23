# Mobile Feature Checklist

Use this checklist before handing a backend feature to a mobile team or wiring it into an app screen.

## Design

- The feature has a clear domain owner plugin.
- Schema names are stable, readable, and action-oriented.
- Public, customer, seller, admin, and delivery behavior are separated.
- Mobile-specific schemas are used only when the mobile contract differs.
- Backward compatibility is considered for existing app versions.

## Routes

- Routes are registered in `plugins/<feature>/index.js`.
- Public routes use `API /` only when they are intentionally anonymous.
- Protected mobile routes use authenticated API routes.
- Admin routes are under `/admin/` and use admin schemas.
- Ownership/precondition checks are visible in route chains when reusable.
- Direct HTTP routes are limited to files, webhooks, confirmation links, health checks, or diagnostics.

## Actions

- Actions live in `plugins/<feature>/schemas/*.js`.
- `input`, `query`, and `params` declarations exist where practical.
- Required fields use `*` only when truly required.
- `$.params`, `$.query`, `model`, and `$.user` are used instead of raw request parsing.
- Errors use `$.invalid()` with either a status code or localized message.
- Success uses `$.success()` or `$.callback()` consistently.
- Long or repeated helper logic is moved to `FUNC` or a module.

## Auth And Authorization

- Anonymous schemas are added to the documented public allowlist if needed.
- Protected schemas return `401` for missing/invalid tokens.
- Authenticated users cannot mutate records they do not own.
- Seller/business membership checks happen server-side.
- Admin-only actions check admin identity or permissions.
- Optional personalization does not break anonymous discovery.
- Logout/session cleanup invalidates relevant caches.

## Data And Responses

- List response shape is documented as array or `{ items, count, page, limit }`.
- Empty lists return an empty list shape, not an error.
- Read of a missing record returns `404`.
- Public listings filter removed, disabled, archived, unpublished, or unapproved records.
- Admin listings intentionally expose broader states.
- Fields are allowlisted; secrets and internal columns are excluded.
- Media URLs are absolute or clearly documented.
- Compatibility aliases are intentional and documented.

## Uploads And Media

- Upload route and upload service URL are documented.
- Upload size limit is explicit.
- Upload auth token/header is scoped and safe for mobile if exposed.
- Download URLs are signed or otherwise protected from raw storage access.
- Domain actions validate that uploaded media can be attached by the current user.

## Operations

- Slow work is moved to `ON('service')`, background jobs, or external workers.
- Startup work in `ON('ready')` is idempotent and non-blocking where possible.
- Cache writes are paired with invalidation on update/remove/logout.
- Diagnostic routes are protected.
- Logs do not reveal tokens, passwords, or private integration data.

## Verification

- Syntax-check edited backend files:

```bash
cd backend-api
node --check path/to/edited-file.js
```

- Exercise important schemas with `.test.api` fixtures or manual API calls.
- If the mobile client changed, run:

```bash
cd mobileapp
npm run type-check
npm run lint
```

- Confirm the mobile anonymous allowlist, API types, and response normalizers match the backend contract.
