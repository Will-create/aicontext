# Schemas, Actions, And Validation

`NEWSCHEMA()` files are the core of feature behavior. Route strings expose schemas; schema actions implement them.

## Basic Action Shape

```javascript
NEWSCHEMA('Products', function(schema) {

	schema.action('read', {
		name: 'Read product',
		params: '*id:UID',
		action: async function($) {
			var response = await DB().read('view_product_listing').id($.params.id).promise($);

			if (!response) {
				$.invalid(404);
				return;
			}

			$.success(response);
		}
	});
});
```

Actions should be small enough to understand but complete enough that the mobile app does not need to fill in missing business rules.

## Context Fields

| Field | Meaning |
|-------|---------|
| `$` | Total.js action context |
| `model` | validated input from `data` |
| `$.params` | schema path parameters |
| `$.query` | schema query parameters |
| `$.user` | resolved user/session |
| `$.headers` | request headers |
| `$.language` | request/user language when available |
| `$.ip`, `$.ua` | request IP and user agent |
| `$.files` | uploaded files on upload routes |

Prefer context data over manually reading raw request bodies.

## Validation Declarations

Declare expected input:

```javascript
schema.action('insert_mobile', {
	input: '*name:String,*businessid:UID,description:String,pictures:[String],saleprice:Number',
	action: async function($, model) {
		model.id = UID();
		model.userid = $.user.id;
		await DB().insert('tbl_product', model).promise($);
		$.success(model.id);
	}
});
```

Declare query and path params:

```javascript
schema.action('smart_query', {
	query: 'search:String,categoryid:String,limit:Number,page:Number,sort:String',
	action: async function($) {
		// $.query is prepared according to the declaration
	}
});

schema.action('update', {
	params: '*id:UID',
	input: 'name:String,description:String',
	action: async function($, model) {
		await DB().modify('tbl_product', model).id($.params.id).promise($);
		$.success();
	}
});
```

Use required `*` fields only when the action cannot safely continue without them.

## Common Action Patterns

List:

```javascript
schema.action('list', {
	query: 'search:String,page:Number,limit:Number',
	action: function($) {
		DB().list('view_business')
			.autoquery($.query, 'id:String,name:String,status:String,dtcreated:Date', 'dtcreated_desc', 50)
			.where('isremoved=FALSE')
			.callback($);
	}
});
```

Read:

```javascript
schema.action('read', {
	params: '*id:UID',
	action: async function($) {
		var item = await DB().read('view_business').id($.params.id).promise($);
		item ? $.success(item) : $.invalid(404);
	}
});
```

Precondition check:

```javascript
schema.action('check', {
	action: function($, model) {
		DB().check('view_business')
			.where('name', model.name)
			.error('@(Business with this name already exists)', true)
			.callback($.done());
	}
});
```

Ownership middleware:

```javascript
schema.action('business_auth', {
	params: '*id:UID',
	action: async function($) {
		var allowed = await FUNC.requireBusinessAccess($, $.params.id);
		allowed ? $.success() : $.invalid(403);
	}
});
```

## Action Composition

Routes can compose actions:

```javascript
ROUTE('API / +account_create --> Customers/check Customers/insert (response)');
```

Use composition when steps are separately meaningful:

- validate uniqueness
- authorize ownership
- calculate derived fields
- perform mutation
- return final response

Avoid long chains where each action depends on hidden side effects from the previous action. If the flow is tightly coupled, a single action may be clearer.

## Helper Placement

Put helpers in the narrowest useful scope:

- local function inside schema file for feature-only formatting
- `FUNC.*` for shared domain utilities
- `modules/*` for external integrations or reusable services
- `definitions/*` for boot-time globals or cross-cutting runtime setup

Do not put mobile UI assumptions in backend helpers. Return domain data; let the app decide presentation.

## Errors And Localization

Use Total.js localization markup in messages that may reach users:

```javascript
$.invalid('@(Unsupported country)');
$.invalid('@(Invalid phone number for selected country)');
```

Use numeric status errors for generic transport states:

```javascript
$.invalid(401);
$.invalid(403);
$.invalid(404);
```

The mobile app can map these to localized UI copy, but backend messages are still useful for direct fixtures and admin tools.
