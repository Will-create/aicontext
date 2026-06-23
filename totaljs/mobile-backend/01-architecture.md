# Architecture And Project Shape

A Total.js backend for a mobile app should be organized around feature plugins and a single API Routing contract. The mobile app should not need to discover many URLs, infer server internals, or duplicate backend business rules.

## Reference Structure

Use this project shape:

```text
backend-api/
  index.js                    # Total.js boot file
  controllers/                # gateway HTTP routes and error handlers
  definitions/                # auth, DB, boot hooks, globals, services
  modules/                    # reusable integrations and service modules
  plugins/
    customers/
      index.js                # route registration and plugin metadata
      schemas/*.js            # NEWSCHEMA actions
      *.test.api              # API request fixtures
    products/
      index.js
      schemas/*.js
  schemas/                    # shared schemas loaded outside plugins
  public/                     # admin/static assets when needed
  database.sql                # snapshot/reference schema
  databases/migrations/       # migration files when used
  config                      # environment-specific config, not secrets for commits
```

The boot file should stay tiny:

```javascript
require('total5');
Total.run({ port: 5000 });
```

## Responsibility Boundaries

Keep each backend concern in its own layer:

| Layer | Responsibility |
|-------|----------------|
| `controllers/` | API gateway route, CORS, HTTP-only endpoints, file storage routes, global error responses |
| `definitions/auth.js` | `AUTH()` middleware, token parsing, user/session resolution |
| `definitions/db.js` | database driver setup, migrations, SQL utilities |
| `definitions/func.js` | shared domain helpers, token/session helpers, normalization helpers |
| `modules/` | reusable services such as mail, monitoring, docs, recommendations, CDN, backup |
| `plugins/<feature>/index.js` | plugin metadata, permissions, route registration |
| `plugins/<feature>/schemas/*.js` | business actions, validation, DB reads/writes |

Do not put feature business rules in the mobile app. If a seller can publish a product only when they own the business, that rule belongs in a backend action or route action chain.

## Mobile-First Backend Principles

- One gateway endpoint for normal API calls.
- Feature routes registered near the feature plugin.
- Schema actions are the public backend interface.
- Auth tokens are opaque and server-issued.
- Public schemas are intentionally documented.
- Response shapes are predictable enough for one mobile normalizer.
- Expensive joins and field aliases happen in backend views or actions.
- Mobile-specific endpoints exist only when the contract truly differs from admin/web.

## Naming And Style

The backend codebase uses CommonJS and Total.js globals:

```javascript
const mailer = require('../modules/mailer');

exports.install = function() {
	ROUTE('API / -api_ping --> api_ping', api_ping);
};

function api_ping($) {
	$.success(true);
}
```

Match the local style:

- `require()` and `module.exports`/`exports.install`
- tabs and semicolons in backend JavaScript
- lowercase filenames, kebab-case for multiword modules
- feature domains in PascalCase schema names such as `Products` or `Customers/Login`
- route schemas in snake-style names such as `products_smart_list`

## What To Generalize From This App

The concepts that transfer well to other Total.js mobile backends are:

- root or `/api/` API Routing with `{ schema, data }`
- plugin-owned route registration
- `NEWSCHEMA()` action files split by domain
- `AUTH()` resolving different identities by path/context
- encrypted session tokens accepted through compatibility headers
- explicit public schema allowlists for mobile clients
- list endpoints based on views and `autoquery()`
- composable ownership checks in route action chains
- separate upload/file service routes
- `.test.api` fixtures for important flows

The exact domain names, database tables, and route list should be project-specific.
