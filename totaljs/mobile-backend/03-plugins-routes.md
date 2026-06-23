# Plugins And Route Registration

Feature plugins are the main architecture unit. A plugin groups metadata, permissions, route registration, schemas, fixtures, and sometimes public/admin UI assets.

## Plugin Entry Point

Each plugin should expose metadata and an `install()` function:

```javascript
exports.icon = 'ti ti-box';
exports.name = '@(Products)';
exports.position = 5;
exports.visible = user => user.sa || user.permissions.includes('products');
exports.permissions = [
	{ id: 'products', name: '@(Products)' },
	{ id: 'products_create', name: '@(Create Products)' }
];

exports.install = function() {
	ROUTE('API     /       -products_smart_list --> Products/smart_query');
	ROUTE('API     /       -products_read/{id} --> Products/read');
	ROUTE('+API    /       +products_insert_mobile --> Products/insert_mobile');
	ROUTE('+API    /admin/ -admin_products --> Products/query');
};
```

Keep route registration in the plugin, not scattered through controllers. The controller should own gateway-level concerns; the plugin should own feature endpoints.

## Route Groups

This backend commonly separates route groups by path and schema prefix:

| Group | Route Pattern | Schema Pattern | Identity |
|-------|---------------|----------------|----------|
| public mobile | `API /` | `products_*`, `businesses_listing` | anonymous or optional user |
| customer mobile | `+API /` | `account_*`, `businesses_*` | customer session |
| admin dashboard | `+API /admin/` | `admin_*`, `admins_*` | admin session |
| delivery app | `+API /delivery/` | delivery-specific schemas | delivery session |
| special service | direct `GET`, `POST`, `FILE` | no API envelope | explicit token or file logic |

The route path helps `AUTH()` choose an identity type. Do not use one token table for every identity unless the project is intentionally single-role.

## Route Composition

Total.js routes can chain actions:

```javascript
ROUTE('+API / #products_publish/{id} --> Products/business_auth Products/publish (response)');
```

Use this for ownership and precondition checks:

1. `Products/business_auth` verifies the user can mutate the product.
2. `Products/publish` performs the mutation.
3. `(response)` returns the final action response.

This pattern keeps authorization reusable and visible at the route level while the actual business logic remains in schemas.

## Public Versus Protected

Do not ask the mobile client to parse route prefixes to decide auth. Instead, maintain a documented anonymous schema allowlist:

```text
account_login
account_login_mobile
account_create
account_create_mobile
products_smart_list
businesses_listing
businesses_read
businesses_products
explorer_nearby
mobile_home
```

Protected schemas should fail with `401` when the token is missing or invalid. Public schemas should not log the mobile user out if an old local token is stale.

## Mobile-Specific Routes

Use `_mobile` only when needed:

- login returns `{ token, user }` instead of only a token
- registration accepts mobile-first fields such as phone/country
- product create/update accepts mobile media arrays or simplified form fields
- password update has different confirmation or current-password behavior

Avoid `_mobile` for identical copies of web/admin actions. Contract duplication becomes expensive.

## HTTP Routes Beside API Routing

Some routes should remain normal HTTP routes:

```javascript
ROUTE('GET /businesses/{id}', business_read);
ROUTE('POST /upload/', upload, ['upload'], 1024 * 5);
ROUTE('FILE /download/*.*', files);
```

Use normal HTTP routes for:

- file upload/download
- health checks
- webhooks
- public confirmation links
- admin/static pages
- logs or service diagnostics with explicit tokens

Keep these rare. Most mobile app work should stay in API Routing.
