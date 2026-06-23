# Data Access, Lists, And Responses

Mobile apps are sensitive to response shape drift. Design backend reads and writes so the client can normalize once and screens can stay simple.

## Querybuilder And Views

Use `DB()` and views for normal list/read endpoints:

```javascript
DB().find('view_business')
	.autoquery($.query, 'id:String,name:String,status:String,logo:String,cover:String,dtcreated:Date', 'dtcreated_desc', 50)
	.where('isremoved=FALSE')
	.callback($);
```

Views are useful for mobile because list screens often need denormalized fields:

- product plus business name
- product plus category name
- business plus location
- order plus totals and status
- wallet transaction plus source/target labels

Keep joins and field aliases in the backend. Do not make mobile stitch multiple calls together for every list row unless the data is truly independent.

## Field Allowlists

Always choose fields intentionally:

```javascript
builder.fields('id,name,logo,cover,status,countryname,cityname');
```

or:

```javascript
builder.autoquery($.query, 'id:String,name:String,status:String,dtcreated:Date', 'dtcreated_desc', 50);
```

Do not expose:

- password hashes
- reset/confirmation tokens
- private admin notes
- raw session rows
- internal removal flags unless the screen needs them
- third-party access tokens
- database-only audit fields irrelevant to mobile

## Public Listing Rules

Public lists should filter out data not meant for buyers:

```javascript
builder.where('isremoved=FALSE');
builder.where('status', 'approved');
builder.where('ispublished', true);
builder.where('isarchived', false);
builder.where('isdisabled', false);
```

Admin lists can expose broader states, but keep those behind `/admin/` and admin auth.

## List Shapes

Pick one stable shape per endpoint:

Simple fixed list:

```javascript
$.callback(items);
```

Paginated/searchable list:

```javascript
$.callback({
	items: items,
	count: count,
	page: page,
	limit: limit
});
```

Querybuilder `.list()` often returns a list envelope. The mobile client can accept arrays and `{ items: [] }`, but backend teams should still document which one each schema returns.

## Response Helpers

Use predictable helpers:

```javascript
$.success(item);              // read/create/update value
$.success(id);                // created ID
$.success();                  // command succeeded
$.callback(items);            // pass list/query result
$.invalid('@(Message)');      // validation failure
$.invalid(404);               // not found
```

Avoid ad hoc shapes like `{ ok: true }`, `{ data: ... }`, and `{ result: ... }` unless the client contract explicitly requires them. If a normal HTTP route must return `{ success, data }`, keep that outside the API Routing contract.

## Mobile-Friendly Aliases

During transitions, returning aliases can be better than breaking old app builds:

```javascript
{
	id: model.id,
	name: model.name,
	logo: model.logo,
	logoUrl: model.logo,
	cover: model.cover,
	coverUrl: model.cover,
	bannerUrl: model.cover,
	isOwner: true,
	isowner: true
}
```

Use aliases intentionally and document them. Do not let every action invent different casing for the same concept.

## Money, Currency, And Locale

For marketplaces, mobile often needs display-ready currency fields:

- source currency
- display currency
- converted display price
- rounding mode
- locale/language-dependent names

Resolve user or query display currency in the backend when the calculation requires server exchange rates. Return both raw and display values if sellers/admins need exact source values.

## Raw SQL

Raw SQL is acceptable for complex discovery or geospatial queries, but keep it parameterized:

```javascript
var rows = await DB().query(`
	SELECT p.id, p.name, p.saleprice
	FROM tbl_product p
	WHERE p.isremoved = FALSE
	AND p.categoryid = $1
	LIMIT $2
`, [categoryid, limit]).promise($);
```

If building dynamic filters, keep values in a parameter array and only append trusted SQL fragments. Never concatenate user input into SQL text.

## Error Consistency

Common backend failure cases should map cleanly:

| Situation | Backend |
|-----------|---------|
| missing token | `$.invalid(401)` or auth invalid |
| authenticated but not owner | `$.invalid(403)` |
| row not found | `$.invalid(404)` |
| validation issue | `$.invalid('@(Message)')` |
| unsupported country/city | `$.invalid('@(Unsupported country)')` |
| empty list | `[]` or `{ items: [], count: 0 }` |

Empty list is not an error. Missing required object is an error.
