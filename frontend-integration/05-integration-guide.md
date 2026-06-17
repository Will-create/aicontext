# Integration Guide — Client Architecture

This document describes the architecture that makes integrating a Total.js backend clean and maintainable. These patterns apply to every platform (React, React Native, Flutter, or any other).

---

## The fundamental insight

Because Total.js API Routing exposes a single endpoint with a uniform `{ schema, data }` body and a uniform `{ success, value, error }` response, the entire HTTP layer of your client collapses to one function:

```
apiRequest(schema, data?) → Promise<response>
```

Every service in your app calls this one function. You configure auth once. You parse responses once. You handle errors once.

---

## Four-layer architecture

```
┌─────────────────────────────────────────┐
│         UI / Screens / Components        │  render, user input, call hooks
├─────────────────────────────────────────┤
│       Custom Hooks / Navigation          │  state, side effects, auth gates
├─────────────────────────────────────────┤
│       Domain API / Service Layer         │  schema strings, typed signatures
├─────────────────────────────────────────┤
│              API Client                  │  one function, interceptors, errors
└─────────────────────────────────────────┘
```

Each layer has one job. Nothing leaks across layer boundaries.

---

## Layer 1 — API Client

**One file. One function. One axios (or http) instance.**

Responsibilities:
- Configure base URL
- Attach token headers to protected requests via an interceptor
- Avoid attaching stale tokens to anonymous schemas
- Handle HTTP `401` globally for protected schemas
- Normalize response envelopes, one-item arrays, root tokens, and errors
- Wrap errors into a consistent thrown shape

```typescript
// apiClient.ts
import axios from 'axios';

const client = axios.create({
  baseURL: 'https://totaljsbackend.com',
  headers: { 'Content-Type': 'application/json' },
});

// Inject token before every request
client.interceptors.request.use((config: any) => {
  const token = getStoredToken(); // platform-specific (see below)
  if (token) config.headers['x-token'] = token;
  return config;
});

// Handle session expiry globally
client.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      clearStoredToken();
      redirectToLogin(); // platform-specific
    }
    return Promise.reject(err);
  }
);

// THE single API function
export async function apiRequest(schema: string, data?: unknown): Promise<any> {
  const payload: Record<string, unknown> = { schema };
  if (data !== undefined) payload.data = data;
  const response = await client.post('/api/', payload);
  return response.data;
}
```

`getStoredToken`, `clearStoredToken`, and `redirectToLogin` are the only platform-specific parts of this layer.

Production mobile clients should make the endpoint path configurable. Generic examples often use `/api/`; Total.js projects that declare `ROUTE('API / ...')` use `/`.

---

## Layer 2 — Service Layer

**One file per resource domain. Pure functions. No state.**

Responsibilities:
- Wrap schema strings so the rest of the codebase never hard-codes them
- Provide typed function signatures
- Keep response normalization to a minimum (mostly pass-through)

```typescript
// postsService.ts
import { apiRequest } from './apiClient';

export type Post = {
  id: string;
  title: string;
  body: string;
  status: 'draft' | 'published';
  dtcreated: string;
};

export type CreatePostInput = Pick<Post, 'title' | 'body' | 'status'>;

export const postsService = {
  list: (params?: { page?: number; limit?: number; status?: string }) => {
    let schema = 'posts_list';
    if (params) {
      const qs = new URLSearchParams(
        Object.entries(params)
          .filter(([, v]) => v !== undefined)
          .map(([k, v]) => [k, String(v)])
      ).toString();
      if (qs) schema += `?${qs}`;
    }
    return apiRequest(schema);
  },

  read: (id: string) => apiRequest(`posts_read/${id}`),
  create: (data: CreatePostInput) => apiRequest('posts_create', data),
  update: (id: string, data: Partial<Post>) => apiRequest(`posts_update/${id}`, data),
  remove: (id: string) => apiRequest(`posts_remove/${id}`),
  publish: (id: string) => apiRequest(`posts_publish/${id}`),

  search: (query: string, options?: { limit?: number; mode?: string }) => {
    let schema = 'posts_search';
    if (options) {
      const qs = new URLSearchParams(
        Object.entries(options)
          .filter(([, v]) => v !== undefined)
          .map(([k, v]) => [k, String(v)])
      ).toString();
      if (qs) schema += `?${qs}`;
    }
    return apiRequest(schema, { query });
  },
};
```

The service layer is thin. It only knows schema names and data shapes. It does not touch state, UI, or side effects.

---

## Layer 3 — Custom Hooks / State

**One hook per resource (or feature). Owns state and side effects.**

Responsibilities:
- Own the state: list of records, single record, loading flag, error message
- Expose stable action functions to the UI
- Call service functions and handle results
- Trigger side effects on success or failure (notifications, navigation, state updates)

```typescript
// usePosts.ts
import { useState, useEffect, useCallback } from 'react';
import { postsService, Post, CreatePostInput } from '../services/postsService';
import { extractErrorMessage } from '../utils/errorHandler';

export function usePosts() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const load = useCallback(async (params?: { page?: number; status?: string }) => {
    setLoading(true);
    setError(null);
    try {
      const res = await postsService.list(params);
      setPosts(res.value ?? []);
    } catch (err) {
      setError(extractErrorMessage(err));
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => { load(); }, [load]);

  const create = async (data: CreatePostInput) => {
    const res = await postsService.create(data);
    setPosts(prev => [res.value, ...prev]);
    return res.value as Post;
  };

  const update = async (id: string, data: Partial<Post>) => {
    const res = await postsService.update(id, data);
    setPosts(prev => prev.map(p => p.id === id ? { ...p, ...res.value } : p));
  };

  const remove = async (id: string) => {
    await postsService.remove(id);
    setPosts(prev => prev.filter(p => p.id !== id));
  };

  return { posts, loading, error, create, update, remove, refresh: load };
}
```

---

## Layer 4 — UI

**No schema strings. No API calls. No token logic.**

```typescript
function PostsPage() {
  const { posts, loading, error, remove } = usePosts();

  if (loading) return <Spinner />;
  if (error) return <ErrorBanner message={error} />;

  return (
    <ul>
      {posts.map(p => (
        <li key={p.id}>
          {p.title}
          <button onClick={() => remove(p.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
}
```

---

## Auth state — global, initialized on startup

Auth state lives in a global context or store that wraps the entire app. It runs one check on startup: read the stored token → call `account` → set user or clear token.

```
AppRoot
  └── AuthProvider / AuthStore (global)
        └── Router / Navigator
              ├── Protected area (requires isAuthenticated)
              └── Public area (login, register, reset)
```

The global auth object exposes:
- `user` — the current user object (null if not authenticated)
- `isAuthenticated` — boolean
- `isLoading` — true while the startup token check is running
- `login(email, password)` — calls service, stores token, sets user
- `register(name, email, password)` — same
- `logout()` — calls service, clears token, clears user

The `isLoading` state is critical — it prevents the login screen from flashing before the token check completes.

For marketplace apps, prefer a guest-first shell when the product allows public browsing:

```text
AppRoot
  -> wait for persisted state hydration
  -> restore language/location preferences
  -> load secure token
  -> no token: show public buyer shell
  -> token: hydrate account + memberships
  -> seller mode only mounts when authenticated
```

Keep the session token in secure storage. Persist only non-secret state such as user snapshot, active business id, app mode, language, country/city, cart count, and notification count.

---

## Handling the array response format

Login (and sometimes other schemas) returns an array. Normalize at the service or hook level, not the UI level:

```typescript
// In authService.login
const raw = await apiRequest('account_login', { email, password });
const item = Array.isArray(raw) ? raw[0] : raw;
if (!item.success) throw new Error(item.error || 'Login failed');
// item.token is the session token
```

Better: normalize all transport formats inside the API client. Screens and hooks should receive domain values, not `{ success, value }` wrappers.

---

## Error handling strategy

| Layer | Responsibility |
|-------|---------------|
| API Client interceptor | Protected HTTP `401` → clear token + redirect or show guest shell |
| Service layer | Re-throw without catching (do not swallow errors) |
| Hook `catch` block | Call `extractErrorMessage`, set `error` state |
| UI component | Read `error` from hook, display a user-facing message |

Never let a raw axios error or unformatted object reach the UI.

---

## Building schema strings with query params

Always build schema query strings programmatically — never via string concatenation:

```typescript
function buildSchema(base: string, params?: Record<string, string | number | boolean | undefined>): string {
  if (!params) return base;
  const entries = Object.entries(params).filter(([, v]) => v !== undefined && v !== null);
  if (!entries.length) return base;
  const qs = new URLSearchParams(entries.map(([k, v]) => [k, String(v)])).toString();
  return `${base}?${qs}`;
}

// Usage
const schema = buildSchema('posts_list', { page: 2, limit: 20, status: 'published' });
// → "posts_list?page=2&limit=20&status=published"
```

---

## Token storage — by platform

| Platform | Storage | Notes |
|----------|---------|-------|
| React (web) | `localStorage` | Standard for SPAs. Use `sessionStorage` for session-scoped auth. |
| React Native | `expo-secure-store` | Keychain (iOS) / Keystore (Android). Never use plain `AsyncStorage` for tokens. |
| Flutter | `flutter_secure_storage` | Keychain (iOS) / Keystore (Android). |

React Native apps can use MMKV/Zustand persist for app state, but not as the source of truth for the session token.

---

## File upload — two-step pattern

```
1. POST multipart to https://fs.totaljsbackend.com/upload/{bucket}/?token=...
   → Response: { id, url, name, size, type }

2. POST /api/ { schema: "documents_create", data: { id, url, name, size, type } }
   → Registers the file in the application
```

Step 1 uses the upload service token (static, from your env config).  
Step 2 uses the session token (from the logged-in user).

Mobile upload helpers should support `{id}` or `{0}` placeholders in the upload URL, append the bucket id when no placeholder is present, and normalize relative returned URLs against the upload service origin.

---

## React Native production checklist

- `apiRequest()` is the only low-level API entrypoint.
- API host, API path, upload URL, upload token, and upload auth header come from Expo env variables.
- Native `localhost` URLs are rejected or replaced unless explicitly allowed.
- Token is stored in `expo-secure-store`.
- Zustand/MMKV persists only non-secret app state.
- Anonymous schemas are allowlisted from the backend public schema contract and verified route behavior.
- The client does not attach stale tokens to login/register/public discovery/OTP calls.
- Protected `401` clears auth state; public `401` does not.
- Response normalization handles `{ success, value }`, array envelopes, raw arrays, root `token`, and `{ items: [] }`.
- Dev network logs mask fields matching token/password/secret/authorization/cookie.
- Seller navigation is gated by auth and business memberships.

---

## WebSocket pattern

One connection per real-time channel. Reconnect with exponential backoff on unexpected close. Pause when the app goes to background (mobile). Clean up on unmount/dispose.

```
connect()
  → on message → dispatch to handler
  → on close (not intentional) → schedule reconnect (delay = min(1000 * 2^n, 30000ms))
  → on error → close, trigger reconnect
  → on intentional disconnect → close(1000), cancel backoff
```

Full implementations are in the platform integration guides.
