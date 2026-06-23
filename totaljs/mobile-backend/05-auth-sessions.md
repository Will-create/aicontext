# Authentication, Sessions, And Identity

Mobile apps should treat backend tokens as opaque session handles. The backend creates, validates, expires, and revokes sessions.

## Token Headers

Accept a small compatibility set of headers:

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

Recommended client behavior:

- send `x-token`
- optionally also send `token` and `Authorization: Bearer` for compatibility
- never decode or inspect the token client-side
- store the token in secure device storage

## Session Creation

On successful login/register:

1. create or reuse a session row
2. encrypt/sign the session ID into a token
3. load a safe user snapshot
4. return token and user to mobile

Preferred mobile response:

```javascript
$.success({ token: result.token, user: result.user });
```

The user snapshot should include bootstrap fields:

- id, name, firstname, lastname
- email and/or phone
- photo/cover
- language and currency
- session ID only if needed by the backend contract
- settings needed at startup
- notification count
- cart count
- seller/business membership hints if cheap to load

Never include password hashes, session secrets, reset tokens, admin-only notes, or integration tokens.

## Auth Middleware

Use `AUTH()` to resolve identity before protected routes run:

```javascript
AUTH(async function($) {
	var routeurl = $.url || ($.uri ? $.uri.pathname : '') || '';
	var cookie = readheadertoken($);

	if (!cookie || cookie.length < 20) {
		$.invalid();
		return;
	}

	if (routeurl.substring(0, 9) === '/delivery') {
		await FUNC.auth_delivery($, cookie);
		return;
	}

	if (routeurl.substring(0, 6) === '/admin')
		await FUNC.auth_admin($, cookie);
	else
		await FUNC.auth_customer($, cookie);
});
```

This path-based identity split lets the same backend support customer mobile, admin dashboard, delivery app, and service routes without confusing user types.

## Public And Optional Auth

Some discovery routes should be public but can personalize when a token is present. In that case:

- do not require auth at the route level
- attempt optional token resolution inside the action or helper
- fall back to anonymous behavior on missing/invalid token
- never return `401` just because optional personalization failed

This pattern is useful for wishlist flags, followed businesses, location personalization, and home feeds.

## Anonymous Schema Allowlist

Mobile clients should keep a base-schema allowlist for public calls:

```text
account_create
account_create_mobile
account_login
account_login_mobile
account_password
products_smart_list
businesses_listing
businesses_read
businesses_products
explorer_nearby
mobile_home
```

The backend team should publish this list. The mobile app should compare only the base schema before `/` or `?`.

This prevents stale tokens from being attached to public login/register calls and prevents public `401` responses from incorrectly clearing the session.

## Logout And Expiry

Logout should be best-effort:

- remove the server session when possible
- clear in-memory/cache entries
- return success even if the client will delete the local token anyway

Service hooks can periodically clean expired sessions and online flags. Keep this logic in `ON('service')` handlers or a dedicated module, not in mobile endpoints.

## Role And Ownership Checks

Authentication answers "who is this?" Authorization answers "can this user do this?"

Use action-level helpers for ownership:

```javascript
var access = await FUNC.requireBusinessAccess($, $.params.id);
if (!access)
	return;
```

Typical checks:

- `$.user.sa` can perform admin operations
- customer owns the session
- seller belongs to the business or has a role
- delivery user owns the delivery task
- public user can only read approved/published data

Backend authorization is mandatory even when mobile navigation hides screens.
