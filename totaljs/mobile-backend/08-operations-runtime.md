# Operations, Jobs, Integrations, And Runtime Hooks

Total.js backends often use definitions and runtime hooks for work that is not a direct request/response action. Keep these concerns explicit so mobile-facing behavior stays reliable.

## Boot-Time Definitions

Use `definitions/` for cross-cutting startup setup:

- database driver initialization
- migrations or idempotent schema setup
- Redis/cache setup
- auth middleware
- localization and money helpers
- realtime/socket setup
- recommendation/search indexing

Keep feature CRUD out of definitions unless it truly is global.

## `ON('ready')`

Use `ON('ready')` for work that needs the app booted:

```javascript
ON('ready', async function() {
	// warm caches, register flow components, run safe startup sync
});
```

Good uses:

- load code lists
- warm cache stores
- register Flow/Redis components
- initialize recommendation/search indexes
- run idempotent migrations

Avoid long blocking tasks that delay API availability. If a task can be slow, run it asynchronously and log failures.

## `ON('service')`

Use `ON('service')` for periodic maintenance:

```javascript
ON('service', async function(counter) {
	if (counter % 30 === 0)
		await MAIN.SESSION.flush();

	if (counter % 360 === 0)
		await DB().remove('tbl_admin_session').query('dtcreated<=NOW() + interval \'-1 month\'').promise();
});
```

Good uses:

- flush expired session caches
- update online flags
- run periodic indexing
- send scheduled notifications
- clean old temporary records
- refresh exchange rates or code lists

Keep mobile request actions fast. Do not make a user wait for maintenance work that belongs in `ON('service')`.

## Shared Globals

This backend uses globals such as `MAIN`, `FUNC`, `CONF`, `REPO`, and `NOW`. Use them consistently:

| Global | Typical Use |
|--------|-------------|
| `CONF` | environment config |
| `FUNC` | shared helper functions |
| `MAIN` | app-level caches/stores/constants |
| `REPO` | in-memory repository/session state |
| `NOW` | Total.js current date helper |

Avoid hiding feature-specific state in globals when a module or DB table would be clearer.

## Caches And Redis

Cache user/session/code-list data when it reduces repeated DB work:

- session cache by session ID
- user cache by session ID
- code lists such as countries/cities/categories
- discovery or recommendation results with a short TTL

Rules:

- cache invalidation must happen on update/logout/remove
- stale cache must not grant permissions
- protected actions should still fail closed if cache data is missing or invalid
- keep TTLs short for volatile data

## Integrations

External integrations should sit behind modules or schema-specific helper functions:

- mailer
- OneSignal/push notifications
- Telegram/WhatsApp
- OAuth providers
- recommendation service
- maps/geolocation
- payment services

Do not call third-party APIs directly from many unrelated actions. Centralize request shape, error handling, logging, and retries.

## Logs And Diagnostics

Diagnostic routes should be protected with explicit tokens or admin auth:

```javascript
ROUTE('GET /logs/', logs);

function logs($) {
	var token = $.query ? $.query.token : null;
	if (!token || token !== CONF.recommendation_token) {
		respond($, 401, 'Unauthorized request');
		return;
	}
}
```

Never expose logs publicly. Scrub tokens, passwords, OAuth values, and personal data from logs when possible.

## Fixtures And Manual Tests

This codebase uses `*.test.api` files for request examples. Keep them close to the plugin they test:

```text
plugins/products/products.test.api
plugins/businesses/businesses.test.api
plugins/transactions/transactions.test.api
```

Fixtures should cover:

- login/register
- public discovery
- protected reads
- seller mutations
- admin mutations
- error cases such as missing auth or invalid IDs

Do not commit live production tokens in fixtures. Use placeholders or short-lived local tokens.

## Runtime Verification

For backend changes, run:

```bash
cd backend-api
node --check path/to/edited-file.js
```

When route behavior changes, manually exercise the schema through an API fixture or request client. For mobile contract changes, run the mobile type-check/lint after updating the client.
