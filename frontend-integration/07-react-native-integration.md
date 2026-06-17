# React Native Integration Guide

Complete implementation notes for an Expo React Native app connected to a Total.js API Routing backend.

Reference stack: **Expo + TypeScript + axios + expo-secure-store + Zustand persist + MMKV fallback**.

This guide captures the patterns used by the SahelBusiness mobile app. Replace the schema names and hostnames with the ones from your project, but keep the same client architecture.

---

## Setup

```bash
npx create-expo-app my-app --template expo-template-blank-typescript
cd my-app
npx expo install expo-secure-store
npm install axios zustand react-native-mmkv
```

Recommended structure:

```text
src/
  api/
    client.ts          # axios instance, apiRequest(), response/error normalization
    auth.ts            # auth schemas
    session.ts         # token save + startup hydration
    upload.ts          # multipart upload helper
    index.ts           # domain APIs
  store/
    useAppStore.ts     # auth, mode, active business, persisted preferences
  hooks/
    useAuthGate.ts     # queue protected actions behind login
  navigation/
    RootNavigator.tsx  # bootstrap + buyer/seller/auth shells
```

---

## Environment Config

Expo exposes public client variables with the `EXPO_PUBLIC_` prefix. Keep secrets out of the app bundle; upload tokens are only acceptable when they are intentionally scoped to the file service.

| Variable | Purpose |
|----------|---------|
| `EXPO_PUBLIC_APP_ENV` | `dev` or `production`; selects suffixed values. |
| `EXPO_PUBLIC_API_BASE_URL` | Default API host. |
| `EXPO_PUBLIC_API_BASE_URL_DEV` | Dev API host override. |
| `EXPO_PUBLIC_API_BASE_URL_PRODUCTION` | Production API host override. |
| `EXPO_PUBLIC_API_PATH` | API path, usually `/` for `ROUTE('API / ...')` projects or `/api/` for conventional deployments. |
| `EXPO_PUBLIC_UPLOAD_URL` | File service upload base URL. |
| `EXPO_PUBLIC_UPLOAD_TOKEN` | Optional file service token. |
| `EXPO_PUBLIC_UPLOAD_AUTH_HEADER` | Optional header name for the upload token, for example `Authorization`. |
| `EXPO_PUBLIC_ALLOW_LOCALHOST_NATIVE` | Set to `true` only when a native runtime can really reach `localhost`. |

Native iOS/Android runtimes usually cannot reach the host machine through `localhost` or `127.0.0.1`. Detect loopback hosts and replace them with a reachable default or LAN/proxy URL unless `EXPO_PUBLIC_ALLOW_LOCALHOST_NATIVE=true`.

```typescript
function hasLoopbackHost(url: string): boolean {
  return /^https?:\/\/(localhost|127\.0\.0\.1|\[::1\])(?::\d+)?(?:\/|$)/i.test(url.trim());
}

function buildApiBaseUrl(baseUrl: string, path: string): string {
  const base = baseUrl.trim().replace(/\/+$/, '');
  const apiPath = path.trim();
  if (!apiPath || apiPath === '/') return `${base}/`;
  return `${base}/${apiPath.replace(/^\/+/, '').replace(/\/+$/, '')}/`;
}
```

---

## API Client

The client has one public function:

```typescript
apiRequest<T>(schema: string, data?: unknown, opts?: { method?: 'GET' | 'POST'; query?: object }): Promise<T>
```

Default to `POST /` or `POST /api/` with `{ schema, data }`. Some projects also expose `GET /?schema=...`; support it only as a convenience, not as the primary integration contract.

Build schema query strings programmatically and omit empty values:

```typescript
function buildSchemaWithQuery(schema: string, query?: object): string {
  if (!query) return schema;
  const parts: string[] = [];

  for (const [key, raw] of Object.entries(query)) {
    if (raw == null || raw === '') continue;
    const values = Array.isArray(raw) ? raw : [raw];
    for (const value of values) {
      if (value == null || value === '') continue;
      parts.push(`${encodeURIComponent(key)}=${encodeURIComponent(String(value))}`);
    }
  }

  return parts.length ? `${schema}?${parts.join('&')}` : schema;
}
```

Attach auth headers from fast in-memory store first, then `SecureStore` as fallback:

```typescript
apiClient.interceptors.request.use(async (config) => {
  const storeToken = useAppStore.getState().token;
  const secureToken = storeToken ? null : await SecureStore.getItemAsync(TOKEN_KEY);
  const token = storeToken ?? secureToken;
  const schema = extractSchemaFromRequest(config);

  if (token && !isAnonymousApiSchema(schema)) {
    config.headers.token = token;
    config.headers['x-token'] = token;
    config.headers.Authorization = `Bearer ${token}`;
  }

  return config;
});
```

`x-token` is the portable Total.js baseline. Sending `token` and `Authorization: Bearer` as compatibility headers is useful when a project has multiple middlewares or file services.

On `401`, clear the token only for protected schemas. Public schemas such as login, registration, discovery lists, and OTP should not be treated as session expiry.

---

## Anonymous Schema Allowlist

Total.js route definitions show the available API schemas, but do not treat the `+` or `-` prefix in the route string as a portable auth contract:

```javascript
ROUTE('API / +account_login --> Customers/Login/exec');
ROUTE('+API / -account_logout --> Customers/logout');
ROUTE('+API / +account_cart_add/{id} --> Customers/Cart/add');
```

Confirm public/protected behavior from backend middleware and real responses, then mirror the public schemas in the mobile client:

```typescript
const ANONYMOUS_API_SCHEMAS = new Set([
  'account_create',
  'account_create_mobile',
  'account_login',
  'account_login_mobile',
  'account_login_google',
  'account_login_facebook',
  'account_login_github',
  'account_google',
  'account_facebook',
  'account_oauth',
  'account_oauth_mobile',
  'account_password',
  'account_reset',
  'account_password_reset',
  'account_verify',
  'products_smart_list',
  'categories',
  'countries_list',
  'cities_list',
  'quarters_list',
  'zones_list',
  'businesses_listing',
  'businesses_read',
  'businesses_products',
  'service_catalog',
  'business_availability',
  'explorer_nearby',
  'explorer_bounds',
  'explorer_map',
  'mobile_home',
  'announcements',
  'announcements_read',
  'otp_sms',
  'otp_sms_verify',
  'otp_sms_verify_mobile',
  'otp_email',
  'otp_email_verify',
]);
```

Compare only the base schema before `/` or `?`:

```typescript
function getBaseSchema(schema: string): string {
  return schema.split('?')[0].split('/')[0];
}
```

This prevents stale tokens from being attached to login/register/public discovery calls and prevents public `401` responses from logging users out.

---

## Response Normalization

Total.js responses may be plain payloads, `{ success, value, error }` envelopes, or one-item array envelopes. Normalize in the API client so screens never parse transport shapes.

Rules:

- If the response is `[ { success/value/error/token } ]`, collapse it to the first item.
- If `success === false` or `error` is present, throw a normalized API error.
- If root `token` exists and `value` is an object, merge the token into the returned object.
- If `value` exists, return `value`.
- If `success === true` with no `value`, return `undefined`.
- For list endpoints, accept both arrays and `{ items: [] }`.

```typescript
export function extractItems<T>(payload: unknown): T[] {
  if (Array.isArray(payload)) return payload as T[];
  if (payload && typeof payload === 'object' && Array.isArray((payload as any).items))
    return (payload as any).items as T[];
  return [];
}
```

Keep one error normalizer for HTTP errors, Total.js array errors, envelope errors, and plain strings. Map network and timeout errors to localized strings before they reach the UI.

---

## Auth And Session Hydration

Store the session token in `expo-secure-store`. Zustand/MMKV can persist non-secret app state such as user snapshot, mode, active business, language, cart count, and notification count, but the token should be restored from SecureStore into memory during bootstrap.

Startup flow:

```text
App starts
  -> wait for Zustand persisted state hydration
  -> restore preferred language
  -> load token from SecureStore
  -> if no token: clear auth state and show buyer/public shell
  -> if token: set token in store and call account
  -> hydrate user, cart count, notification count
  -> load account_businesses
  -> if seller mode has no business membership, switch to buyer mode
  -> choose BuyerShell or SellerShell
```

Login/register flow:

```text
account_login_mobile or account_login
  -> normalize { token, user? } or plain token string
  -> save token to SecureStore
  -> set token/user in store
  -> call account and account_businesses in background
```

Logout flow:

```text
account_logout best-effort
  -> delete SecureStore token
  -> clear auth state
  -> preserve non-sensitive preferences such as language/country/city
```

The mobile app can be guest-first: public marketplace screens load without auth, auth opens as a modal, and seller-only navigation mounts only when authenticated.

---

## Auth Gate

Use an auth gate for actions that are available from public screens but require login to complete, such as checkout, wishlist, follow, seller dashboard, or booking.

```typescript
let pendingAction: (() => void) | null = null;

export function useAuthGate() {
  const isAuthenticated = useIsAuthenticated();
  const navigation = useNavigation();

  return {
    requireAuth(action: () => void) {
      if (isAuthenticated) action();
      else {
        pendingAction = action;
        navigation.navigate('Auth', { screen: 'Welcome' });
      }
    },
  };
}

export function consumePendingAuthAction() {
  const action = pendingAction;
  pendingAction = null;
  action?.();
}
```

Call `consumePendingAuthAction()` after successful login/register.

---

## Domain API Layer

Do not call schema strings from screens. Keep typed domain APIs thin:

```typescript
export const productsApi = {
  smartList: (params?: ProductsListParams) =>
    apiRequest<unknown>('products_smart_list', undefined, { query: params }).then(extractItems<Product>),
  read: (id: string) =>
    apiRequest<Product>(`products_read/${encodeURIComponent(id.trim())}`),
  insertMobile: (data: Partial<Product>) =>
    apiRequest<Product>('products_insert_mobile', data),
};
```

Useful mobile domains:

- `authApi`: login, mobile login, register, profile, password reset, OAuth.
- `mobileHomeApi`: cached aggregate home data with fallback to parallel domain calls.
- `productsApi`: marketplace and seller product actions.
- `businessesApi`: public discovery and seller business management.
- `serviceBusinessApi`: public catalog/availability and owner service catalog.
- `cartApi`, `ordersApi`, `walletApi`, `sellerWalletApi`: authenticated commerce flows.

---

## Uploads

File uploads do not use the Total.js API envelope. Upload multipart form data to the file service, then store the returned URL/metadata through a normal schema if the domain requires it.

Build the bucket from active business id, user id, or `anonymous`:

```typescript
function buildUploadUrl(userId: string): string {
  const id = encodeURIComponent(userId || 'anonymous');
  const hasPlaceholder = UPLOAD_URL.includes('{id}') || UPLOAD_URL.includes('{0}');
  const raw = UPLOAD_URL.replace('{id}', id).replace('{0}', id);
  const url = new URL(raw);

  if (!hasPlaceholder)
    url.pathname = `${url.pathname.replace(/\/+$/, '')}/${id}`;

  url.pathname = `${url.pathname.replace(/\/+$/, '')}/`;

  if (UPLOAD_TOKEN && !UPLOAD_AUTH_HEADER && !url.searchParams.has('token'))
    url.searchParams.set('token', UPLOAD_TOKEN);

  return url.toString();
}
```

Auth priority:

1. If `UPLOAD_TOKEN` and `UPLOAD_AUTH_HEADER` are configured, send the upload token in that header.
2. If `UPLOAD_TOKEN` is configured without a header, add it as `?token=...`.
3. Otherwise send the user session token as `token`, `x-token`, and `Authorization`.

Normalize relative upload responses into absolute URLs using the upload service origin.

---

## Production Checklist

- `apiRequest()` is the only low-level API entrypoint.
- API base URL and upload URL are environment-driven.
- Native localhost is blocked or replaced unless explicitly allowed.
- Session token lives in SecureStore, not AsyncStorage.
- Anonymous schemas are explicitly allowlisted and do not receive stale tokens.
- `401` clears auth only for protected schemas.
- Response normalization handles envelopes, arrays, root tokens, and raw lists.
- Domain APIs own schema strings; screens call typed functions/hooks.
- Auth bootstrap waits for persisted store hydration before navigation decisions.
- Uploads use the file service and normalize returned URLs.
- Dev network logs mask token/password/secret fields.
