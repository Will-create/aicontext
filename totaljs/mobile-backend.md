# Total.js Mobile App Backend Guide

This guide set explains how to build a Total.js backend for a mobile app using the architecture, style, and operational practices seen in this app's backend.

The main idea is simple: expose a small HTTP surface, route every mobile operation through Total.js API Routing, keep domain behavior in plugin schemas, enforce auth and ownership server-side, and return shapes that a mobile client can normalize once.

## Guide Map

Read these in order when designing a new backend:

1. [Architecture And Project Shape](mobile-backend/01-architecture.md)
2. [API Routing Contract](mobile-backend/02-api-routing.md)
3. [Plugins And Route Registration](mobile-backend/03-plugins-routes.md)
4. [Schemas, Actions, And Validation](mobile-backend/04-schemas-actions.md)
5. [Authentication, Sessions, And Identity](mobile-backend/05-auth-sessions.md)
6. [Data Access, Lists, And Responses](mobile-backend/06-data-responses.md)
7. [Files, Uploads, Media, And Devices](mobile-backend/07-files-media-devices.md)
8. [Operations, Jobs, Integrations, And Runtime Hooks](mobile-backend/08-operations-runtime.md)
9. [Mobile Feature Checklist](mobile-backend/09-feature-checklist.md)

## Core Contract

Mobile clients should need one API client:

```http
POST /
Content-Type: application/json
x-token: <session_token>

{
  "schema": "products_smart_list?page=1&limit=20",
  "data": { "optional": "payload" }
}
```

The backend should make these guarantees:

- schema names are stable and action-oriented
- public/protected behavior is explicit and documented
- mobile login/register can return `{ token, user }`
- list endpoints return arrays or `{ items, count, page, limit }`
- errors flow through `$.invalid()` and can be normalized by one client parser
- ownership checks live in backend actions, not in mobile screens
- upload/media URLs are documented and stable

## Companion Guides

- Frontend: [React Native Integration Guide](../frontend-integration/07-react-native-integration.md)
- Total.js basics: [Actions](actions.md), [Plugins](plugin.md), [Databases](databases.md), [Globals](globals.md)
