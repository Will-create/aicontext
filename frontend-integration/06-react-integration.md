# React Integration Guide

Complete, copy-pasteable React implementation for a Total.js API Routing backend.
Stack: **Vite + TypeScript + React 18 + React Router v6 + axios**.

Adapt `totaljsbackend.com` and the example resource `posts` to your project.

---

## Setup

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install axios react-router-dom
```

### Environment variables

```bash
# .env
VITE_API_BASE_URL=https://totaljsbackend.com
VITE_FILE_UPLOAD_BASE_URL=https://fs.totaljsbackend.com
VITE_FILE_UPLOAD_TOKEN=your_upload_service_token
VITE_FILE_UPLOAD_BUCKET=documents
```

---

## Project structure

```
src/
  api/
    client.ts          ← axios instance + interceptors + apiRequest()
  services/
    authService.ts
    postsService.ts    ← example resource
  context/
    AuthContext.tsx
  hooks/
    usePosts.ts
  utils/
    errorHandler.ts
  components/
    ProtectedRoute.tsx
  pages/
    LoginPage.tsx
    RegisterPage.tsx
    PostsPage.tsx
```

---

## API Client — `src/api/client.ts`

```typescript
import axios, { AxiosError, AxiosResponse } from 'axios';

const BASE_URL = import.meta.env.VITE_API_BASE_URL ?? 'https://totaljsbackend.com';

export const apiClient = axios.create({
  baseURL: BASE_URL,
  headers: { 'Content-Type': 'application/json' },
});

// Inject session token before every request
apiClient.interceptors.request.use((config: any) => {
  const token = localStorage.getItem('session_token');
  if (token && config.headers) config.headers['x-token'] = token;
  return config;
});

// Global 401 handler — session expired
apiClient.interceptors.response.use(
  (res: AxiosResponse) => res,
  (err: AxiosError) => {
    if (err.response?.status === 401) {
      localStorage.removeItem('session_token');
      window.location.href = '/login';
    }
    return Promise.reject(err);
  }
);

/**
 * The single function for all Total.js API calls.
 * schema: "resource_action", "resource_action/{id}", "resource_action?param=value"
 */
export async function apiRequest(schema: string, data?: unknown): Promise<any> {
  const payload: Record<string, unknown> = { schema };
  if (data !== undefined) payload.data = data;
  const response = await apiClient.post('/api/', payload);
  return response.data;
}
```

---

## Error handler — `src/utils/errorHandler.ts`

```typescript
export function extractErrorMessage(error: unknown): string {
  // Total.js array response: [{ "error": "..." }]
  if (Array.isArray(error)) {
    const first = (error as any[])[0];
    return first?.error || first?.value || 'An error occurred';
  }

  // Axios error with a Total.js body
  const responseData = (error as any)?.response?.data;
  if (responseData) {
    if (Array.isArray(responseData)) return responseData[0]?.error || 'An error occurred';
    if (responseData.error) return responseData.error;
  }

  if ((error as any)?.message) return (error as any).message;
  if (typeof error === 'string') return error;

  return 'An unexpected error occurred';
}
```

---

## Auth service — `src/services/authService.ts`

```typescript
import { apiRequest } from '../api/client';

export const authService = {
  login: async (email: string, password: string) => {
    const raw = await apiRequest('account_login', { email, password });
    // Normalize array vs object response
    const item = Array.isArray(raw) ? raw[0] : raw;
    if (!item.success) throw new Error(item.error || 'Login failed');
    const token = item.token ?? item.value;
    if (token) localStorage.setItem('session_token', token);
    return item;
  },

  register: async (name: string, email: string, password: string) => {
    const res = await apiRequest('account_create', { name, email, password });
    if (res?.success && res?.value) localStorage.setItem('session_token', res.value);
    return res;
  },

  getProfile: () => apiRequest('account'),

  logout: async () => {
    await apiRequest('account_logout').catch(() => {});
    localStorage.removeItem('session_token');
  },

  updateProfile: (data: unknown) => apiRequest('account_update', data),

  changePassword: (currentPassword: string, newPassword: string) =>
    apiRequest('account_password', { current_password: currentPassword, new_password: newPassword }),

  requestPasswordReset: (email: string) => apiRequest('account_reset', { email }),
  verifyAccount: (token: string) => apiRequest('account_verify', { token }),

  // OAuth
  getGoogleOAuthUrl: (page: string) => apiRequest(`account_google?page=${page}`),
  getGithubOAuthUrl: (page: string) => apiRequest(`account_github?page=${page}`),
  exchangeOAuthSession: (sessionid: string) => apiRequest('account_oauth', { sessionid }),
  loginWithGoogle: (token: string) => apiRequest('account_login_google', { token }),
  loginWithGithub: (token: string) => apiRequest('account_login_github', { token }),

  // 2FA
  generate2FA: () => apiRequest('account_2fa_generate'),
  enable2FA: (token: string) => apiRequest('account_2fa_enable', { token }),
  disable2FA: () => apiRequest('account_2fa_disable'),
  verify2FA: (token: string) => apiRequest('account_2fa_verify', { token }),
};
```

---

## Auth Context — `src/context/AuthContext.tsx`

```typescript
import { createContext, useContext, useState, useEffect, ReactNode } from 'react';
import { useNavigate } from 'react-router-dom';
import { authService } from '../services/authService';
import { extractErrorMessage } from '../utils/errorHandler';

type User = {
  id: string;
  name: string;
  email: string;
  role?: string;
  sa?: boolean;
};

type AuthContextType = {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (name: string, email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  updateUser: (data: Partial<User>) => void;
};

const AuthContext = createContext<AuthContextType>({} as AuthContextType);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const navigate = useNavigate();

  // Startup: validate stored token and hydrate user
  useEffect(() => {
    const token = localStorage.getItem('session_token');
    if (!token) { setIsLoading(false); return; }

    authService.getProfile()
      .then((res) => { if (res.success) setUser(res.value); })
      .catch(() => localStorage.removeItem('session_token'))
      .finally(() => setIsLoading(false));
  }, []);

  const login = async (email: string, password: string) => {
    try {
      const item = await authService.login(email, password);
      setUser(item.value ?? { email } as User);
      navigate('/dashboard');
    } catch (err) {
      throw new Error(extractErrorMessage(err));
    }
  };

  const register = async (name: string, email: string, password: string) => {
    try {
      await authService.register(name, email, password);
      const profile = await authService.getProfile();
      if (profile.success) setUser(profile.value);
      navigate('/dashboard');
    } catch (err) {
      throw new Error(extractErrorMessage(err));
    }
  };

  const logout = async () => {
    await authService.logout();
    setUser(null);
    navigate('/login');
  };

  const updateUser = (data: Partial<User>) => {
    setUser(prev => prev ? { ...prev, ...data } : prev);
  };

  return (
    <AuthContext.Provider value={{ user, isAuthenticated: !!user, isLoading, login, register, logout, updateUser }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => useContext(AuthContext);
```

### App root wiring — `src/main.tsx`

```typescript
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <BrowserRouter>
    <AuthProvider>
      <App />
    </AuthProvider>
  </BrowserRouter>
);
```

### Protected route — `src/components/ProtectedRoute.tsx`

```typescript
import { Navigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function ProtectedRoute({ children }: { children: JSX.Element }) {
  const { isAuthenticated, isLoading } = useAuth();
  if (isLoading) return <div>Loading...</div>;
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}
```

---

## Example service — `src/services/postsService.ts`

```typescript
import { apiRequest } from '../api/client';

export type Post = {
  id: string;
  title: string;
  body: string;
  status: 'draft' | 'published';
  dtcreated: string;
};

function buildSchema(base: string, params?: Record<string, any>): string {
  if (!params) return base;
  const qs = new URLSearchParams(
    Object.entries(params)
      .filter(([, v]) => v !== undefined && v !== null)
      .map(([k, v]) => [k, String(v)])
  ).toString();
  return qs ? `${base}?${qs}` : base;
}

export const postsService = {
  list: (params?: { page?: number; limit?: number; status?: string }) =>
    apiRequest(buildSchema('posts_list', params)),

  read: (id: string) => apiRequest(`posts_read/${id}`),

  create: (data: Pick<Post, 'title' | 'body' | 'status'>) =>
    apiRequest('posts_create', data),

  update: (id: string, data: Partial<Post>) =>
    apiRequest(`posts_update/${id}`, data),

  remove: (id: string) => apiRequest(`posts_remove/${id}`),

  publish: (id: string) => apiRequest(`posts_publish/${id}`),

  search: (query: string, options?: { limit?: number; mode?: string }) =>
    apiRequest(buildSchema('posts_search', options), { query }),
};
```

---

## Custom hook — `src/hooks/usePosts.ts`

```typescript
import { useState, useEffect, useCallback } from 'react';
import { postsService, Post } from '../services/postsService';
import { extractErrorMessage } from '../utils/errorHandler';

export function usePosts(initialParams?: { status?: string }) {
  const [posts, setPosts] = useState<Post[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const load = useCallback(async (params?: { page?: number; status?: string }) => {
    setLoading(true);
    setError(null);
    try {
      const res = await postsService.list({ ...initialParams, ...params });
      setPosts(res.value ?? []);
    } catch (err) {
      setError(extractErrorMessage(err));
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => { load(); }, [load]);

  const create = async (data: Pick<Post, 'title' | 'body' | 'status'>) => {
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

## Login page — `src/pages/LoginPage.tsx`

```typescript
import { useState, FormEvent } from 'react';
import { Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

export function LoginPage() {
  const { login } = useAuth();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    setLoading(true);
    setError('');
    try {
      await login(email, password);
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        placeholder="Email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        required
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        required
      />
      {error && <p style={{ color: 'red' }}>{error}</p>}
      <button type="submit" disabled={loading}>
        {loading ? 'Signing in…' : 'Sign in'}
      </button>
      <Link to="/register">Create account</Link>
      <Link to="/reset-password">Forgot password?</Link>
    </form>
  );
}
```

---

## File upload

```typescript
async function uploadFile(file: File): Promise<{ id: string; url: string; name: string; size: number; type: string }> {
  const baseUrl = import.meta.env.VITE_FILE_UPLOAD_BASE_URL ?? 'https://fs.totaljsbackend.com';
  const token = import.meta.env.VITE_FILE_UPLOAD_TOKEN;
  const bucket = import.meta.env.VITE_FILE_UPLOAD_BUCKET ?? 'documents';

  const form = new FormData();
  form.append('file', file);

  const res = await fetch(`${baseUrl}/upload/${bucket}/?token=${token}&hostname=1`, {
    method: 'POST',
    body: form,
  });

  if (!res.ok) throw new Error(`Upload failed: ${res.status}`);
  return res.json();
}

// Usage — upload then register
async function handleFileSelect(file: File) {
  const uploadData = await uploadFile(file);

  // Register in the app via the main API
  const res = await apiRequest('documents_create', {
    id:   uploadData.id,
    url:  uploadData.url,
    name: uploadData.name,
    size: uploadData.size,
    type: uploadData.type,
  });

  return res.value;
}
```

---

## WebSocket hook

```typescript
import { useEffect, useRef } from 'react';

export function useBackendSocket(
  path: string,               // e.g. "/ws/" or "/ws/channel_123"
  params: Record<string, string>,
  onMessage: (msg: any) => void,
  enabled = true
) {
  const wsRef = useRef<WebSocket | null>(null);
  const retryRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const retriesRef = useRef(0);

  useEffect(() => {
    if (!enabled) return;

    const token = localStorage.getItem('session_token') ?? '';
    const qs = new URLSearchParams({ ...params, token }).toString();

    function connect() {
      const url = `wss://totaljsbackend.com${path}?${qs}`;
      const ws = new WebSocket(url);
      wsRef.current = ws;

      ws.onmessage = (e) => {
        try { onMessage(JSON.parse(e.data)); } catch {}
      };

      ws.onclose = (e) => {
        if (e.code === 1000) return; // intentional close
        const delay = Math.min(1000 * 2 ** retriesRef.current, 30000);
        retriesRef.current++;
        retryRef.current = setTimeout(connect, delay);
      };

      ws.onerror = () => ws.close();
    }

    connect();

    return () => {
      if (retryRef.current) clearTimeout(retryRef.current);
      wsRef.current?.close(1000);
    };
  }, [path, enabled]);
}
```

Usage:

```typescript
function NotificationsPanel() {
  const [events, setEvents] = useState<any[]>([]);

  useBackendSocket(
    '/ws/',
    {},
    (msg) => {
      if (msg.type === 'notification') {
        setEvents(prev => [msg.data, ...prev]);
      }
    }
  );

  return <div>{events.map((e, i) => <p key={i}>{e.title}</p>)}</div>;
}
```
