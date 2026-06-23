# API Routing Contract

Total.js API Routing is an action-based API style. The mobile app calls one endpoint and sends the operation name in `schema`.

## Request Shape

```http
POST /
Content-Type: application/json
x-token: <session_token>

{
  "schema": "businesses_products/biz123?page=1&limit=20",
  "data": { "optional": "payload" }
}
```

Some Total.js projects mount the API at `/api/`; this app-style backend mounts it at `/`. The mobile app should configure the API path instead of hard-coding it.

## Schema String Anatomy

```text
<operation>
<operation>/<id>
<operation>/<id>/<childid>
<operation>?key=value&key2=value2
<operation>/<id>?key=value
```

Examples:

```text
account_login_mobile
account_orders_read/ord123
businesses_products/biz123?search=rice&limit=20
products_update_mobile/prod123
business_wallet_transfer/src123/tgt456
```

## Operation Naming

Prefer readable, action-oriented names:

| Schema | Meaning |
|--------|---------|
| `account` | current authenticated user profile |
| `account_login_mobile` | mobile login |
| `account_logout` | logout current session |
| `products_smart_list` | public product discovery list |
| `products_read/{id}` | public product detail |
| `products_insert_mobile` | seller mobile product create |
| `businesses_listing` | public business discovery |
| `businesses_myproducts/{id}` | seller products for owned business |
| `admin_products` | admin product list |

The mobile app should be able to read a schema name and understand intent without knowing database table names.

## Query Parameters

Filters belong in the schema string:

```json
{
  "schema": "products_smart_list?categoryid=cat1&country=ML&limit=20"
}
```

In the action, read filters from `$.query`:

```javascript
schema.action('smart_query', {
	query: 'categoryid:String,country:String,limit:Number,page:Number,sort:String',
	action: async function($) {
		$.query.categoryid && builder.where('categoryid', $.query.categoryid);
	}
});
```

Avoid putting API filters on the HTTP URL. The HTTP URL should stay the gateway path.

## Path Parameters

Path parameters belong in the schema after `/`:

```javascript
ROUTE('API / -businesses_products/{id} --> Businesses/products');
```

The action declares and reads them:

```javascript
schema.action('products', {
	params: '*id:UID',
	action: function($) {
		DB().list('view_product_listing')
			.where('businessid', $.params.id)
			.callback($);
	}
});
```

## Public API Contract

For each schema exposed to mobile, document:

- schema name
- public or protected
- required `data`
- supported query parameters
- path parameters
- response shape
- common error cases

The route file alone is not enough documentation for mobile. Route metadata such as `+`, `-`, and `#` can be useful internally, but mobile clients should use an explicit anonymous allowlist and real response behavior as the source of truth.

## Versioning And Compatibility

Do not rename schema strings casually. For mobile apps, schema names are as public as REST URLs.

When changing a contract:

- keep the old schema until old app versions can age out
- add a new schema such as `_mobile`, `_v2`, or a clearer action name when shape changes
- keep response aliases during transition, for example `logo` and `logoUrl`
- avoid breaking list item fields used by home, search, map, and seller screens

## Error Behavior

Use the Total.js context helpers:

```javascript
$.invalid('@(Unsupported country)');
$.invalid(401);
$.invalid(404);
```

The mobile client should be able to normalize:

- HTTP errors
- `{ success: false, value/error/code }`
- Total.js validation errors
- plain string messages
- one-item array envelopes
