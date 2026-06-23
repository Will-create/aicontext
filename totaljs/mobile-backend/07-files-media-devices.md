# Files, Uploads, Media, And Devices

File handling is usually the one part of a mobile backend that should not use the API Routing envelope. Use normal multipart routes for upload and signed file routes for download.

## Upload Route

```javascript
exports.install = function() {
	if (!is_file_storage_server())
		return;

	ROUTE('POST /upload/', upload, ['upload'], 1024 * 5);
	ROUTE('POST /upload/{id}/', upload, ['upload'], 1024 * 5);
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

Keep upload limits explicit. Mobile apps should know max file size before opening an image picker or camera flow.

## Download Route

Signed download IDs protect raw storage IDs:

```javascript
function files($) {
	var req = $.req || $;
	var split = $.split || req.split || [];

	var index = split[1].lastIndexOf('.');
	if (index !== -1) {
		var hash = split[1].substring(0, index);
		var id = hash.substring(0, hash.indexOf('-', 10));
		if (hash === id.sign(CONF.salt)) {
			$.filefs('files', id);
			return;
		}
	}

	$.throw404();
}
```

Do not return raw filesystem paths to mobile.

## Media URL Normalization

Mobile clients prefer absolute URLs. If storage returns relative paths, normalize them before returning business/product/user payloads:

```javascript
response.url = FUNC.media_url($, response.url);
```

For domain objects, normalize common media fields server-side:

```javascript
FUNC.normalize_media_fields($, response);
```

Good field names:

- `photo`
- `logo`
- `cover`
- `pictures`
- `video`
- compatibility aliases such as `logoUrl` and `coverUrl` when needed

## Upload Auth

If uploads are served by a separate file storage service, mobile config may include:

```text
EXPO_PUBLIC_UPLOAD_URL
EXPO_PUBLIC_UPLOAD_TOKEN
EXPO_PUBLIC_UPLOAD_AUTH_HEADER
```

Only use a public upload token when it is intentionally scoped to upload behavior and safe to distribute in a client bundle. Never expose general API secrets or admin tokens through mobile environment variables.

## Native Device URLs

Native iOS and Android apps usually cannot reach the developer machine through `localhost` or `127.0.0.1`.

Document one of these for development:

- LAN IP, such as `http://192.168.1.20:5000/`
- tunnel URL
- staging API host
- emulator-specific host when applicable

The backend should not assume every local client is a browser. The frontend guide should explain loopback fallback behavior.

## CORS

Configure CORS centrally:

```javascript
CORS('/*', [
	'https://app.example.com',
	'http://localhost:3000',
	'http://localhost:8081',
	'http://127.0.0.1:8081'
], true);
```

For native apps, CORS is less important than for web, but Expo web, development tools, dashboards, and file services still need correct origins.

## Media Response Practices

When returning uploaded media:

- return an array for multi-file upload
- include `url`, `id`, `ext`, `type`, and size when available
- return absolute `url` if the mobile app should display it immediately
- keep the response stable across upload route variants

When saving media into domain records:

- store canonical URLs or storage IDs consistently
- accept arrays for multi-image fields such as product pictures
- validate whether the current user can attach media to the target business/product
- avoid deleting old media immediately unless the action is clearly destructive
