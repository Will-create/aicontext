# Total.js Backend — Frontend Integration Guide

A complete, platform-agnostic reference for integrating any frontend or mobile client with a **Total.js API Routing** backend.

This guide stack is designed to be reused across projects. Replace `totaljsbackend.com` with your actual backend URL and substitute example resource names (`items`, `posts`, `users`) with your own domain schemas.

---

## Documents

| File | What it covers |
|------|----------------|
| [01-totaljs-routing.md](./01-totaljs-routing.md) | The Total.js API Routing philosophy — what it is and how it differs from REST |
| [02-request-response.md](./02-request-response.md) | The exact HTTP contract — request shape, response envelope, errors, file uploads, WebSocket |
| [03-authentication.md](./03-authentication.md) | Auth flows: password login, registration, OAuth, 2FA, token lifecycle |
| [04-api-reference.md](./04-api-reference.md) | Schema naming conventions, standard CRUD patterns, and a full annotated example domain |
| [05-integration-guide.md](./05-integration-guide.md) | Client architecture — the four-layer pattern, error strategy, token storage |
| [06-react-integration.md](./06-react-integration.md) | React (Vite + TypeScript) — complete, copy-pasteable implementation |
| [07-react-native-integration.md](./07-react-native-integration.md) | React Native (Expo) — complete implementation with mobile-specific concerns |
| [08-flutter-integration.md](./08-flutter-integration.md) | Flutter (Dart) — complete implementation |

---

## The three facts you need to know first

```
1. Every API call is:   POST https://totaljsbackend.com/api/
2. Every request body:  { "schema": "resource_action[/id][?params]", "data": { ... } }
3. Every auth:          x-token: <session_token>  (header, injected globally)
```

That is the entire surface area of a Total.js API Routing backend from the client's perspective.

---

## Reading order for new developers

1. **01** — understand the routing model before writing a single line of code
2. **02** — internalize the exact wire format
3. **03** — implement auth before anything else
4. **05** — understand the recommended client architecture
5. **Your platform guide** (06, 07, or 08) — copy the boilerplate, adapt to your domain
6. **04** — reference when you need to know how a schema should be named or structured
