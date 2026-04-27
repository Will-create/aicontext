# React Native Integration Guide

Complete implementation for a React Native app (Expo) connecting to a Total.js API Routing backend.
Stack: **Expo + TypeScript + axios + expo-secure-store + Zustand**.

Adapt `totaljsbackend.com` and the example resource `posts` to your project.

---

## Setup

```bash
npx create-expo-app my-app --template expo-template-blank-typescript
cd my-app
npx expo install expo-secure-store
npm install axios zustand
```

---

## Project structure

```
src/
  api/
    client.ts          ← axios instance + interceptors + apiRequest()
    tokenStorage.ts    ← SecureStore wrapper
    authEvents.ts      ← event bus for 401 redirect
  services/
    authService.ts
    postsService.ts
  store/
    authStore.ts       ← Zustand auth state
  hooks/
    usePosts.ts
    useBackendSocket.ts
  utils/
    errorHandler.ts
  screens/
    LoginScreen.tsx
    RegisterScreen.tsx
    PostsScreen.tsx
  navigation/
    AppNavigator.tsx
```

---

## Token storage — `src/api/tokenStorage.ts`

```typescript
import * as SecureStore from 'expo-secure-store';

const KEY = 'totaljs_session_token';

export const tokenStorage = {
  get: (): Promise<string | null> => SecureStore.getItemAsync(KEY),
  set: (token: string): Promise<void> => SecureStore.setItemAsync(KEY, token),
  clear: (): Promise<void> => SecureStore.deleteItemAsync(KEY),
};
```

> Always use `expo-secure-store` for session tokens. It stores them in Keychain (iOS) and Keystore-backed storage (Android). Never store tokens in `AsyncStorage`.

---

## Auth event bus — `src/api/authEvents.ts`

React Navigation does not have `window.location`. Use an event emitter so the HTTP client can trigger navigation from outside the component tree.

```typescript
import { EventEmitter } from 'events';
export const authEvents = new EventEmitter();
```

---

## API Client — `src/api/client.ts`

```typescript
import axios, { AxiosError, AxiosResponse } from 'axios';
import { tokenStorage } from './tokenStorage';
import { authEvents } from './authEvents';

const BASE_URL = 'https://totaljsbackend.com';

export const apiClient = axios.create({
  baseURL: BASE_URL,
  headers: { 'Content-Type': 'application/json' },
});

// Inject token before every request (async — SecureStore is async)
apiClient.interceptors.request.use(async (config: any) => {
  const token = await tokenStorage.get();
  if (token && config.headers) config.headers['x-token'] = token;
  return config;
});

// Handle 401 globally
apiClient.interceptors.response.use(
  (res: AxiosResponse) => res,
  async (err: AxiosError) => {
    if (err.response?.status === 401) {
      await tokenStorage.clear();
      authEvents.emit('unauthorized'); // navigation layer listens to this
    }
    return Promise.reject(err);
  }
);

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
  if (Array.isArray(error)) {
    const first = (error as any[])[0];
    return first?.error || first?.value || 'An error occurred';
  }
  const data = (error as any)?.response?.data;
  if (data) {
    if (Array.isArray(data)) return data[0]?.error || 'An error occurred';
    if (data.error) return data.error;
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
import { tokenStorage } from '../api/tokenStorage';

export const authService = {
  login: async (email: string, password: string) => {
    const raw = await apiRequest('account_login', { email, password });
    const item = Array.isArray(raw) ? raw[0] : raw;
    if (!item.success) throw new Error(item.error || 'Login failed');
    const token = item.token ?? item.value;
    if (token) await tokenStorage.set(token);
    return item;
  },

  register: async (name: string, email: string, password: string) => {
    const res = await apiRequest('account_create', { name, email, password });
    if (res?.success && res?.value) await tokenStorage.set(res.value);
    return res;
  },

  getProfile: () => apiRequest('account'),

  logout: async () => {
    await apiRequest('account_logout').catch(() => {});
    await tokenStorage.clear();
  },

  changePassword: (current: string, next: string) =>
    apiRequest('account_password', { current_password: current, new_password: next }),

  requestPasswordReset: (email: string) => apiRequest('account_reset', { email }),
  verifyAccount: (token: string) => apiRequest('account_verify', { token }),

  // Mobile OAuth — direct ID token exchange
  loginWithGoogle: async (idToken: string) => {
    const res = await apiRequest('account_login_google', { token: idToken });
    const token = res?.value?.token ?? res?.token;
    if (token) await tokenStorage.set(token);
    return res;
  },

  loginWithGithub: async (accessToken: string) => {
    const res = await apiRequest('account_login_github', { token: accessToken });
    const token = res?.value?.token ?? res?.token;
    if (token) await tokenStorage.set(token);
    return res;
  },

  generate2FA: () => apiRequest('account_2fa_generate'),
  enable2FA: (token: string) => apiRequest('account_2fa_enable', { token }),
  disable2FA: () => apiRequest('account_2fa_disable'),
  verify2FA: (token: string) => apiRequest('account_2fa_verify', { token }),
};
```

---

## Auth store — `src/store/authStore.ts`

```typescript
import { create } from 'zustand';
import { authService } from '../services/authService';
import { tokenStorage } from '../api/tokenStorage';

type User = { id: string; name: string; email: string; role?: string; sa?: boolean };

type AuthState = {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  initialize: () => Promise<void>;
  login: (email: string, password: string) => Promise<void>;
  register: (name: string, email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  setUser: (user: User) => void;
};

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  isAuthenticated: false,
  isLoading: true,

  initialize: async () => {
    const token = await tokenStorage.get();
    if (!token) { set({ isLoading: false }); return; }
    try {
      const res = await authService.getProfile();
      if (res.success) set({ user: res.value, isAuthenticated: true });
    } catch {
      // 401 already handled by interceptor (token cleared)
    } finally {
      set({ isLoading: false });
    }
  },

  login: async (email, password) => {
    const item = await authService.login(email, password);
    set({ user: item.value ?? { email } as User, isAuthenticated: true });
  },

  register: async (name, email, password) => {
    await authService.register(name, email, password);
    const res = await authService.getProfile();
    if (res.success) set({ user: res.value, isAuthenticated: true });
  },

  logout: async () => {
    await authService.logout();
    set({ user: null, isAuthenticated: false });
  },

  setUser: (user) => set({ user }),
}));
```

---

## Root navigator — `src/navigation/AppNavigator.tsx`

```typescript
import { useEffect } from 'react';
import { NavigationContainer, createNavigationContainerRef } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { useAuthStore } from '../store/authStore';
import { authEvents } from '../api/authEvents';
import { LoginScreen } from '../screens/LoginScreen';
import { RegisterScreen } from '../screens/RegisterScreen';
import { DashboardScreen } from '../screens/DashboardScreen';

export const navigationRef = createNavigationContainerRef();
const Stack = createNativeStackNavigator();

export function AppNavigator() {
  const { isAuthenticated, isLoading, initialize } = useAuthStore();

  useEffect(() => { initialize(); }, []);

  // Listen for global 401 events
  useEffect(() => {
    const handler = () =>
      navigationRef.current?.reset({ index: 0, routes: [{ name: 'Login' }] });
    authEvents.on('unauthorized', handler);
    return () => { authEvents.off('unauthorized', handler); };
  }, []);

  if (isLoading) return null; // or <SplashScreen />

  return (
    <NavigationContainer ref={navigationRef}>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <Stack.Screen name="Dashboard" component={DashboardScreen} />
        ) : (
          <>
            <Stack.Screen name="Login" component={LoginScreen} />
            <Stack.Screen name="Register" component={RegisterScreen} />
          </>
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

---

## Example service — `src/services/postsService.ts`

```typescript
import { apiRequest } from '../api/client';

function buildSchema(base: string, params?: Record<string, any>): string {
  if (!params) return base;
  const qs = new URLSearchParams(
    Object.entries(params)
      .filter(([, v]) => v !== undefined)
      .map(([k, v]) => [k, String(v)])
  ).toString();
  return qs ? `${base}?${qs}` : base;
}

export const postsService = {
  list: (params?: { page?: number; limit?: number; status?: string }) =>
    apiRequest(buildSchema('posts_list', params)),

  read: (id: string) => apiRequest(`posts_read/${id}`),
  create: (data: { title: string; body: string; status: string }) => apiRequest('posts_create', data),
  update: (id: string, data: object) => apiRequest(`posts_update/${id}`, data),
  remove: (id: string) => apiRequest(`posts_remove/${id}`),
};
```

---

## Login screen — `src/screens/LoginScreen.tsx`

```typescript
import { useState } from 'react';
import {
  View, Text, TextInput, TouchableOpacity,
  ActivityIndicator, StyleSheet, KeyboardAvoidingView, Platform,
} from 'react-native';
import { useAuthStore } from '../store/authStore';

export function LoginScreen() {
  const { login } = useAuthStore();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    setLoading(true);
    setError('');
    try {
      await login(email.trim(), password);
      // AuthStore.isAuthenticated changes → AppNavigator re-renders to Dashboard
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      <Text style={styles.title}>Sign in</Text>
      <TextInput
        style={styles.input}
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
        autoCorrect={false}
      />
      <TextInput
        style={styles.input}
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
      />
      {error ? <Text style={styles.error}>{error}</Text> : null}
      <TouchableOpacity style={styles.button} onPress={handleLogin} disabled={loading}>
        {loading
          ? <ActivityIndicator color="#fff" />
          : <Text style={styles.buttonText}>Sign in</Text>
        }
      </TouchableOpacity>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', padding: 24 },
  title: { fontSize: 24, fontWeight: '700', marginBottom: 24 },
  input: {
    borderWidth: 1, borderColor: '#d1d5db', borderRadius: 8,
    padding: 12, marginBottom: 12, fontSize: 16,
  },
  button: {
    backgroundColor: '#2563eb', borderRadius: 8,
    padding: 14, alignItems: 'center', marginTop: 8,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  error: { color: '#dc2626', marginBottom: 8 },
});
```

---

## File upload

```typescript
import * as DocumentPicker from 'expo-document-picker';
import { apiRequest } from '../api/client';

async function uploadAndRegisterDocument() {
  const result = await DocumentPicker.getDocumentAsync({ type: '*/*' });
  if (result.canceled) return;

  const file = result.assets[0];
  const uploadToken = process.env.EXPO_PUBLIC_UPLOAD_TOKEN ?? '';
  const bucket = process.env.EXPO_PUBLIC_UPLOAD_BUCKET ?? 'documents';

  // Step 1: Upload to file service
  const form = new FormData();
  form.append('file', {
    uri: file.uri,
    name: file.name,
    type: file.mimeType ?? 'application/octet-stream',
  } as any);

  const uploadRes = await fetch(
    `https://fs.totaljsbackend.com/upload/${bucket}/?token=${uploadToken}&hostname=1`,
    { method: 'POST', body: form }
  );
  const uploadData = await uploadRes.json();

  // Step 2: Register in the app
  const res = await apiRequest('documents_create', {
    id:   uploadData.id,
    url:  uploadData.url,
    name: uploadData.name ?? file.name,
    size: uploadData.size ?? file.size,
    type: uploadData.type ?? file.mimeType,
  });

  return res.value;
}
```

---

## WebSocket hook — `src/hooks/useBackendSocket.ts`

```typescript
import { useEffect, useRef } from 'react';
import { AppState, AppStateStatus } from 'react-native';
import { tokenStorage } from '../api/tokenStorage';

export function useBackendSocket(
  path: string,
  params: Record<string, string>,
  onMessage: (msg: any) => void,
  enabled = true
) {
  const wsRef = useRef<WebSocket | null>(null);
  const retryRef = useRef<ReturnType<typeof setTimeout> | null>(null);
  const retriesRef = useRef(0);
  const mountedRef = useRef(true);

  useEffect(() => {
    if (!enabled) return;
    mountedRef.current = true;

    async function connect() {
      if (!mountedRef.current) return;
      const token = await tokenStorage.get() ?? '';
      const qs = new URLSearchParams({ ...params, token }).toString();
      const url = `wss://totaljsbackend.com${path}?${qs}`;

      const ws = new WebSocket(url);
      wsRef.current = ws;

      ws.onmessage = (e) => {
        if (!mountedRef.current) return;
        try { onMessage(JSON.parse(e.data)); } catch {}
      };

      ws.onclose = (e) => {
        if (!mountedRef.current || e.code === 1000) return;
        const delay = Math.min(1000 * 2 ** retriesRef.current, 30000);
        retriesRef.current++;
        retryRef.current = setTimeout(connect, delay);
      };

      ws.onerror = () => ws.close();
    }

    connect();

    // Pause socket when app goes to background to save battery
    const handleAppState = (state: AppStateStatus) => {
      if (state === 'background' || state === 'inactive') {
        if (retryRef.current) clearTimeout(retryRef.current);
        wsRef.current?.close(1000);
      } else if (state === 'active') {
        retriesRef.current = 0;
        connect();
      }
    };

    const sub = AppState.addEventListener('change', handleAppState);

    return () => {
      mountedRef.current = false;
      sub.remove();
      if (retryRef.current) clearTimeout(retryRef.current);
      wsRef.current?.close(1000);
    };
  }, [path, enabled]);
}
```

Usage:

```typescript
function NotificationsScreen() {
  const [events, setEvents] = useState<any[]>([]);

  useBackendSocket('/ws/', {}, (msg) => {
    if (msg.type === 'notification') {
      setEvents(prev => [msg.data, ...prev]);
    }
  });

  return (
    <FlatList
      data={events}
      keyExtractor={(_, i) => String(i)}
      renderItem={({ item }) => <Text>{item.title}</Text>}
    />
  );
}
```

---

## Key differences from React web

| Concern | React (web) | React Native |
|---------|-------------|--------------|
| Token storage | `localStorage` | `expo-secure-store` |
| 401 redirect | `window.location.href` | `authEvents` + `navigationRef.reset()` |
| Navigation | React Router `<Navigate>` | `AppNavigator` re-renders on auth state change |
| File upload body | `FormData` with `File` | `FormData` with `{ uri, name, type }` object |
| WebSocket lifecycle | Component unmount | Component unmount + `AppState` background handling |
| Environment vars | `import.meta.env.VITE_*` | `process.env.EXPO_PUBLIC_*` |
