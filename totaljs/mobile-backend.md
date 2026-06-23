# Total.js Mobile App Backend Guide

This guide describes how to build a Total.js backend that is easy for a mobile app to consume. It generalizes the backend patterns used in this project: API Routing on one endpoint, plugin-owned routes, schema actions, opaque session tokens, predictable envelopes, and mobile-friendly list/upload contracts.

Use this as the backend companion to the React Native integration guide. Replace domain names such as `Products`, `Businesses`, or `Customers` with the domain names from your project, but keep the same API shape unless there is a clear reason to diverge.

---

## Project Shape

A mobile-oriented Total.js backend should keep the HTTP surface small and the domain behavior organized by feature:

```text
backend-api/
  index.js                    # boot Total.js and set the port
  controllers/                # top-level HTTP/API gateway routes
  definitions/                # auth, database, globals, boot hooks, services
  modules/                    # reusable service modules and integrations
  plugins/
    products/
      index.js                # route registration and plugin metadata
      schemas/*.js            # NEWSCHEMA actions
      *.test.api              # endpoint fixtures when useful
  schemas/                    # shared/global schemas
  database.sql                # database snapshot or reference schema
  databases/migrations/       # migration files when the project uses them
```

Keep `index.js` minimal:

```javascript
require('total5');
Total.run({ port: 5000 });
```

Put gateway concerns in controllers, cross-cutting boot logic in `definitions/`, reusable helpers in `modules/`, and feature routes/actions in `plugins/`.

---

## API Routing Contract

Mobile clients should call one Total.js API endpoint with an envelope:

```http
POST /
Content-Type: application/json
x-token: <session_token>

{
  "schema": "products_smart_list?page=1&limit=20",
  "data": { "optional": "payload" }
}
```

Some projects mount this at `/api/`; this project-style mount uses `/`. Make the path configurable in the mobile app.

Schema strings carry the operation address:

```text
account_login
account_orders_read/{id}
products_smart_list?categoryid=cat123&limit=20
businesses_update/{id}
```

Use these conventions consistently:

| Pattern | Purpose |
|---------|---------|
| `domain_list` or `domain` | list/query records |
| `domain_read/{id}` | read one record |
| `domain_insert` or `domain_create` | create a record |
| `domain_update/{id}` | update a record |
| `domain_remove/{id}` | remove/archive a record |
| `domain_action/{id}` | explicit business operation |
| `account_*` | current customer/session operations |
| `admin_*` | dashboard/admin-only operations |

Append filters to the schema string, not to the HTTP URL. In actions, read them from `$.query`.

---

## Routes And Plugins

Each feature plugin should own its route registration in `plugins/<feature>/index.js`:

```javascript
exports.icon = 'ti ti-box';
exports.name = '@(Products)';
exports.position = 5;
exports.visible = user => user.sa || user.permissions.includes('products');
exports.permissions = [{ id: 'products', name: '@(Products)' }];

exports.install = function() {
	ROUTE('API     /       -products_smart_list --> Products/smart_query');
	ROUTE('API     /       -products_read/{id} --> Products/read');
	ROUTE('+API    /       +products_insert_mobile --> Products/insert_mobile');
	ROUTE('+API    /       +products_update_mobile/{id} --> Products/update_mobile');

	ROUTE('+API    /admin/ -admin_products --> Products/query');
	ROUTE('+API    /admin/ +admin_products_update/{id} --> Products/update');
};
```

Route strings are useful documentation, but do not make the mobile client infer public/protected behavior from the `+`, `-`, or `#` markers. Treat auth as a backend contract and publish a client allowlist of anonymous schemas such as login, registration, catalog discovery, and OTP endpoints.

Use separate schema names for mobile only when the mobile contract really differs. Good reasons include returning `{ token, user }` on login, accepting phone-number country normalization, or simplifying upload/media fields. Avoid mobile-specific duplicates for identical behavior.

---

## Actions And Validation

Put business behavior in `NEWSCHEMA()` files under the feature's `schemas/` directory:

```javascript
NEWSCHEMA('Products', function(schema) {

	schema.action('smart_query', {
		name: 'List products',
		query: 'search:String,categoryid:String,country:String,city:String,limit:Number,page:Number,sort:String',
		action: async function($) {
			var db = DB();
			var builder = db.list('view_product_listing');

			builder.autoquery($.query, MAIN.products_list, 'dtcreated_desc', 100);
			$.query.search && builder.search($.language === 'fr' ? 'search' : 'search_en', $.query.search);
			builder.where('isremoved=FALSE');
			builder.where('status', 'approved');
			builder.where('ispublished', true);

			builder.callback($);
		}
	});

	schema.action('read', {
		name: 'Read product',
		params: '*id:UID',
		action: async function($) {
			var response = await DB().read('view_product_listing').id($.params.id).promise($);
			response ? $.success(response) : $.invalid(404);
		}
	});
});
```

Use Total.js action context consistently:

| Context | Use |
|---------|-----|
| `$.params` | path segments from `schema/{id}` |
| `$.query` | filters from `schema?key=value` |
| `model` or `$.model` | validated request body from `data` |
| `$.user` | authenticated user/session |
| `$.headers` | token, device, locale, and integration headers |
| `$.invalid(messageOrCode)` | validation/auth/not-found failures |
| `$.success(value)` | successful value envelope |
| `$.callback(value)` | pass through querybuilder/action results |

Declare `input`, `query`, and `params` schemas on actions where possible. That keeps mobile validation failures predictable and prevents screens from having to guess whether an empty response means invalid input, missing auth, or no data.

---

## Authentication And Sessions

Mobile sessions should use opaque server-issued tokens. The client stores the token, sends it on requests, and never decodes it.

Accept the common headers:

```javascript
function readheadertoken($) {
	var headers = $.headers || EMPTYOBJECT;
	var token = headers['x-token'] || headers.token || '';
	var auth = headers.authorization || '';

	if (!token && auth && auth.substring(0, 7).toLowerCase() === 'bearer ')
		token = auth.substring(7).trim();

	return token || '';
}
```

In `AUTH()`, route by context:

- `/admin/` resolves admin sessions.
- `/delivery/` resolves delivery sessions.
- default API routes resolve customer sessions.
- special system routes, file services, or flow/MCP routes should have explicit token checks.

For mobile login/register, prefer returning both token and user snapshot:

```javascript
$.success({ token: result.token, user: result.user });
```

The user snapshot should include fields needed for app bootstrap: identity, language, currency, photo, settings, unread notification count, cart count, and role or business membership hints. Keep the token itself out of logs and out of non-secure persisted app state.

---

## Authorization And Ownership

Authenticate at the route level, then authorize inside an action or composable action chain. This keeps ownership rules close to the data they protect:

```javascript
ROUTE('+API / #products_publish/{id} --> Products/business_auth Products/publish (response)');
```

Typical ownership checks should:

- read `$.user` and `$.params.id`
- verify business/team membership or admin privilege
- call `$.invalid(401)` or `$.invalid(403)` before the mutating action
- pass through with `$.success()` or `$.callback()` only when access is valid

Do not rely only on client-side navigation to protect seller or admin operations. Mobile apps can hide buttons, but the backend must enforce ownership.

---

## Mobile-Friendly Responses

Keep transport shapes boring. The mobile client should have one normalizer, not per-screen response parsing.

Recommended shapes:

```javascript
$.success(item);              // read/create/update
$.success(id);                // create when only ID is needed
$.callback(items);            // simple list
$.callback({ items, count }); // paginated list
$.invalid('@(Message)');      // validation error
$.invalid(404);               // not found
```

For list endpoints, choose one of these and keep it stable:

- array for small or fixed lists
- `{ items, count, page, limit }` for paginated/searchable lists

For media fields, normalize server-side where possible. Return absolute URLs or a clearly documented relative media path. If legacy fields exist, mobile-friendly aliases are acceptable during transition:

```javascript
{
	id: model.id,
	logo: model.logo,
	logoUrl: model.logo,
	cover: model.cover,
	coverUrl: model.cover
}
```

For public marketplace/discovery screens, unauthenticated routes may still accept an optional token to personalize results. When doing that, treat auth as optional and fall back to public behavior if token resolution fails.

---

## Data Access

Use `querybuilderpg`/`DB()` for normal CRUD and list endpoints:

```javascript
DB().find('view_business')
	.autoquery($.query, 'id:String,name:String,status:String,dtcreated:Date', 'dtcreated_desc', 50)
	.where('isremoved=FALSE')
	.callback($);
```

Use views for mobile listing screens when rows combine multiple tables. This keeps expensive joins and field naming in the backend instead of duplicating them across clients.

Prefer allowlisted fields in `.fields()` or `.autoquery()` definitions. Avoid returning internal columns such as password hashes, session internals, removal flags, private notes, or integration tokens.

For raw SQL, keep parameters separate from SQL text. Build dynamic filters with helper functions or querybuilder methods instead of string concatenating user input.

---

## Uploads And Files

Mobile apps usually need a multipart upload endpoint separate from the API Routing endpoint:

```javascript
exports.install = function() {
	ROUTE('POST /upload/', upload, ['upload'], 1024 * 5);
	ROUTE('FILE /download/*.*', files);
};

async function upload($) {
	var output = [];

	for (var file of $.files) {
		var response = await file.fs('files', UID());
		response.url = '/download/{0}.{1}'.format(response.id.sign(CONF.salt), response.ext);
		response.url = FUNC.media_url($, response.url);
		output.push(response);
	}

	$.json(output);
}
```

Keep upload limits explicit. If a separate file storage service is used, expose its upload URL to mobile through public environment config and protect it with a deliberately scoped token/header, not a general backend secret.

Signed download IDs are preferable to raw storage IDs. The download handler should reject unsigned or malformed paths.

---

## CORS, Devices, And Environments

Native iOS/Android runtimes cannot normally call `localhost` on the developer machine. Document a reachable LAN, tunnel, staging, or production URL for mobile builds.

Configure CORS centrally in the API controller:

```javascript
CORS('/*', [
	'https://app.example.com',
	'http://localhost:3000',
	'http://localhost:8081',
	'http://127.0.0.1:8081'
], true);
```

Do not commit real credentials in `config`, `.env`, EAS settings, or app public variables. Client-side environment variables are public by design.

---

## Backend Checklist For A Mobile Feature

Before handing a mobile-backed feature to the client team, verify:

- The route is registered in the owning plugin `index.js`.
- The schema name follows existing naming conventions.
- Public/protected behavior is documented and the mobile anonymous allowlist is updated if needed.
- The action declares `input`, `query`, and `params` validation where practical.
- The response shape is stable and handled by the shared mobile normalizer.
- Ownership checks are enforced on the backend for seller/admin mutations.
- List endpoints support predictable pagination, filtering, and sorting.
- Media fields are absolute URLs or documented relative paths.
- Upload endpoints, size limits, and auth headers are documented.
- A `*.test.api` fixture or manual request example exists for important flows.

Run at least:

```bash
cd backend-api
node --check path/to/edited-file.js
```

If you changed the mobile app integration as well:

```bash
cd mobileapp
npm run type-check
npm run lint
```
