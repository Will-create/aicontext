# API Reference — Patterns & Conventions

This document explains how Total.js API Routing schemas are structured and provides a complete annotated example domain. Use it as a template when learning a new Total.js backend — ask the backend developer for the actual schema names used in their project.

---

## Base URL and endpoint

```
POST https://totaljsbackend.com/api/
```

Replace `totaljsbackend.com` with the actual backend hostname for your project.

If the backend declares routes with `ROUTE('API / ...')`, the API endpoint is the root path:

```text
POST https://api.example.com/
```

Keep the API path configurable in clients.

---

## Auth legend

- `🌐` — Public. No `x-token` header required.
- `🔒` — Protected. Requires a valid `x-token` header.
- `🔒⚡` — Protected + elevated role (admin, etc.). Server decides.

---

## Standard CRUD schema patterns

Every resource follows this pattern. Replace `{resource}` with the actual name (`posts`, `users`, `orders`, `messages`, etc.):

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `{resource}_list` | 🔒 | — | Return all records |
| `{resource}_list?page=&limit=` | 🔒 | — | Paginated list |
| `{resource}_read/{id}` | 🔒 | — | Return one record |
| `{resource}_create` | 🔒 | Resource fields | Create a record |
| `{resource}_insert` | 🔒 | Resource fields | Alias for create |
| `{resource}_update/{id}` | 🔒 | Fields to update | Update a record |
| `{resource}_remove/{id}` | 🔒 | — | Delete a record |

---

## Standard account schemas

These are present in virtually every Total.js backend:

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `account_create` | 🌐 | `name`, `email`, `password` | Register |
| `account_create_mobile` | 🌐 | `phone`, `country`, `password`, profile fields | Mobile registration |
| `account_login` | 🌐 | `email`, `password` | Login — returns session token |
| `account_login_mobile` | 🌐 | `phone` or `email`, `country`, `password` | Mobile login |
| `account_logout` | 🔒 | — | Invalidate session |
| `account` | 🔒 | — | Get current user profile |
| `account_update` | 🔒 | Profile fields | Update profile |
| `account_password` | 🔒 | `current_password`, `new_password` | Change password |
| `account_reset` | 🌐 | `email` | Request password reset |
| `account_password_reset` | 🌐 | `token`, `password`, `confirm` | Complete password reset |
| `account_verify` | 🌐 | `token` | Verify email address |
| `account_list` | 🔒⚡ | — | List all users (admin) |

### Login response
```json
[{ "success": true, "token": "9f4a2c...", "value": { "id": "...", "name": "...", "email": "..." } }]
```

### Register response
```json
{ "success": true, "value": "<session_token>" }
```

### Profile response
```json
{
  "success": true,
  "value": {
    "id": "usr_abc",
    "name": "Jane Doe",
    "email": "user@example.com",
    "role": "admin",
    "sa": false
  }
}
```

---

## OAuth schemas (when implemented)

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `account_google?page={page}` | 🌐 | — | Get Google OAuth redirect URL |
| `account_github?page={page}` | 🌐 | — | Get GitHub OAuth redirect URL |
| `account_oauth` | 🌐 | `sessionid` | Exchange OAuth session for token |
| `account_login_google` | 🌐 | `token` | Login with Google ID token (mobile) |
| `account_login_facebook` | 🌐 | `token` | Login with Facebook token (mobile) |
| `account_login_github` | 🌐 | `token` | Login with GitHub token (mobile) |

---

## 2FA schemas (when implemented)

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `account_2fa_generate` | 🔒 | — | Generate TOTP secret + QR URI |
| `account_2fa_enable` | 🔒 | `token` (6-digit TOTP) | Activate 2FA |
| `account_2fa_disable` | 🔒 | — | Deactivate 2FA |
| `account_2fa_verify` | 🔒 | `token` (6-digit TOTP) | Verify TOTP during login |

---

## SahelBusiness mobile schema map

These are the high-value schemas used by the Expo mobile app. Use this as a practical map when integrating React Native with the Total.js backend.

### Public discovery

| Schema | Auth | Description |
|--------|------|-------------|
| `mobile_home` | 🌐 | Aggregated mobile home payload when enabled. |
| `products_smart_list` | 🌐 | Public product discovery list. |
| `products_read/{id}` | 🌐 | Product details. |
| `products_view/{id}` | 🌐 | Track product view. |
| `categories` | 🌐 | Category list. |
| `businesses_listing` | 🌐 | Public business discovery list. |
| `businesses_read/{id}` | 🌐 | Public business profile. |
| `businesses_products/{id}` | 🌐 | Products for one business. |
| `explorer_nearby` | 🌐 | Nearby map/list discovery. |
| `explorer_bounds` | 🌐 | Discovery within map bounds. |
| `explorer_map` | 🌐 | Map discovery payload. |
| `announcements` | 🌐 | Public announcements list. |
| `announcements_read/{id}` | 🌐 | Announcement detail. |
| `jobs`, `jobs_read/{id}`, `jobs_featured`, `jobs_nearby`, `jobs_categories`, `jobs_suggestions` | 🌐 | Jobs marketplace flows. |
| `service_catalog/{businessid}` | 🌐 | Public service/gas/hotel catalog. |
| `business_availability/{businessid}` | 🌐 | Public availability for service businesses. |

### Auth and profile

| Schema | Auth | Description |
|--------|------|-------------|
| `account_login`, `account_login_mobile` | 🌐 | Password login. |
| `account_create`, `account_create_mobile` | 🌐 | Register customer account. |
| `account` | 🔒 | Current user profile. |
| `account_logout` | 🔒 | End session. |
| `account_update` | 🔒 | Update profile. |
| `account_settings_save` | 🔒 | Persist notification/profile settings. |
| `account_search` | 🔒 | Search accounts for team invites or contacts. |
| `otp_sms`, `otp_sms_verify`, `otp_sms_verify_mobile`, `otp_email`, `otp_email_verify` | 🌐 | OTP verification flows. |

### Buyer commerce

| Schema | Auth | Description |
|--------|------|-------------|
| `account_cart`, `account_cart_add/{id}`, `account_cart_replace/{id}`, `account_cart_remove/{id}`, `account_cart_clear`, `account_cart_preview` | 🔒 | Cart read and mutation flows. |
| `account_orders`, `account_orders_read/{id}` | 🔒 | Buyer order history and details. |
| `orders_create`, `orders_pay/{id}`, `orders_cancel/{id}` | 🔒 | Checkout/payment/cancel flows. |
| `account_address`, `account_address_insert`, `account_address_read/{id}`, `account_address_update/{id}`, `account_address_remove/{id}` | 🔒 | Buyer addresses. |
| `account_notifications`, `account_notifications_read/{id}` | 🔒 | Notification center. |
| `account_wallet`, `account_wallet_deposits`, `account_wallet_withdrawals`, `account_wallet_history`, `account_wallet_deposit` | 🔒 | Buyer wallet. |

### Seller and business owner

| Schema | Auth | Description |
|--------|------|-------------|
| `account_businesses`, `account_businesses_read/{id}` | 🔒 | Seller memberships and active business hydration. |
| `businesses_register`, `businesses_update/{id}`, `businesses_myproducts/{businessid}`, `businesses_orders/{businessid}` | 🔒 | Seller business management. |
| `businesses_documents_insert`, `businesses_documents_read/{id}` | 🔒 | Business documents. |
| `businesses_users_list/{businessid}`, `businesses_users_invite`, `businesses_users_revoke/{businessid}/{userid}`, `businesses_users_role_update/{businessid}/{userid}/{roleid}` | 🔒 | Business team management. |
| `products_insert_mobile`, `products_update_mobile/{id}`, `products_remove/{id}`, `products_publish/{id}`, `products_unpublish/{id}`, `products_archive/{id}`, `products_unarchive/{id}` | 🔒 | Seller product lifecycle. |
| `products_variants/{productid}`, `products_variants_insert/{productid}`, `products_variants_update/{productid}/{id}`, `products_variants_remove/{productid}/{id}` | 🔒 | Product variants. |
| `business_wallet/{businessid}`, `business_wallet_history/{businessid}`, `business_wallet_deposits/{businessid}`, `business_wallet_withdrawals/{businessid}`, `business_wallet_withdraw/{businessid}` | 🔒 | Seller wallet. |

### Service business

| Schema | Auth | Description |
|--------|------|-------------|
| `owner_service_catalog/{businessid}` | 🔒 | Owner catalog list. |
| `owner_service_catalog_insert/{businessid}` | 🔒 | Add fuel/gas/service/appointment item. |
| `owner_service_catalog_update/{businessid}/{itemid}` | 🔒 | Update catalog item. |
| `owner_service_catalog_remove/{businessid}/{itemid}` | 🔒 | Remove catalog item. |
| `owner_availability_update/{businessid}` | 🔒 | Upsert service availability. |
| `booking_create`, `account_bookings`, `booking_cancel/{bookingid}` | 🔒 | Customer booking flows. |
| `owner_bookings/{businessid}`, `owner_booking_status/{businessid}/{bookingid}` | 🔒 | Owner booking management. |

### Location/reference data

| Schema | Auth | Description |
|--------|------|-------------|
| `countries_list`, `cities_list`, `quarters_list`, `zones_list` | 🌐 | Location selectors and filters. |

Use this map to decide the mobile anonymous allowlist, but keep the actual allowlist intentionally narrow: include public schemas that must not receive stale tokens or whose `401` responses should not clear the local session.

---

## Example domain: a blog API

This is a fully worked example showing what a real Total.js backend schema set looks like across three resources: `posts`, `comments`, and `categories`.

### Posts

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `posts_list` | 🌐 | — | List published posts |
| `posts_list?page=&limit=&status=` | 🔒 | — | Filtered/paginated list |
| `posts_read/{id}` | 🌐 | — | Read one post |
| `posts_create` | 🔒 | `title`, `body`, `status`, `tags` | Create post |
| `posts_update/{id}` | 🔒 | Any post fields | Update post |
| `posts_remove/{id}` | 🔒 | — | Delete post |
| `posts_publish/{id}` | 🔒 | — | Publish draft |
| `posts_search?limit=` | 🌐 | `query` | Full-text search |

**Post object:**
```json
{
  "id": "post_abc",
  "title": "Getting started",
  "body": "...",
  "status": "published",
  "tags": ["intro", "guide"],
  "authorId": "usr_xyz",
  "dtcreated": "2025-01-15T09:00:00.000Z",
  "dtupdated": "2025-01-16T12:00:00.000Z"
}
```

### Comments

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `comments_list?postid=` | 🌐 | — | List comments on a post |
| `comments_create` | 🔒 | `postid`, `body` | Add comment |
| `comments_update/{id}` | 🔒 | `body` | Edit own comment |
| `comments_remove/{id}` | 🔒 | — | Delete comment |

### Categories

| Schema | Auth | Data | Description |
|--------|------|------|-------------|
| `categories_list` | 🌐 | — | List all categories |
| `categories_create` | 🔒⚡ | `name`, `slug` | Create category (admin) |
| `categories_update/{id}` | 🔒⚡ | `name`, `slug` | Update category (admin) |
| `categories_remove/{id}` | 🔒⚡ | — | Delete category (admin) |

---

## Query parameters in schema strings — patterns

### Pagination

```json
{ "schema": "posts_list?page=2&limit=20" }
```

### Filtering

```json
{ "schema": "posts_list?status=published&authorid=usr_xyz" }
```

### Sorting

```json
{ "schema": "posts_list?sort=dtcreated&order=desc" }
```

### Search with options

```json
{
  "schema": "posts_search?limit=10&mode=fulltext",
  "data": { "query": "total.js tutorial" }
}
```

---

## Response shapes — summary

**Single record:**
```json
{ "success": true, "value": { "id": "...", ... } }
```

**List:**
```json
{ "success": true, "value": [ { "id": "..." }, ... ] }
```

**Paginated list:**
```json
{
  "success": true,
  "value": [ ... ],
  "meta": { "total": 152, "page": 2, "limit": 20 }
}
```

**Create / update:**
```json
{ "success": true, "value": { "id": "...", ... } }
```

**Delete:**
```json
{ "success": true, "value": true }
```

**Error:**
```json
{ "success": false, "error": "Not found", "code": "not_found" }
```

---

## File upload service

```
POST https://fs.totaljsbackend.com/upload/{bucket}/
    ?token=<upload_token>&hostname=1
Content-Type: multipart/form-data
```

Upload response:
```json
{
  "id": "file_abc",
  "url": "https://fs.totaljsbackend.com/file/...",
  "name": "report.pdf",
  "size": 204800,
  "type": "application/pdf"
}
```

After upload, register the file via the relevant `*_create` schema with the returned metadata.

---

## WebSocket

```
wss://totaljsbackend.com/ws/
    ?token=<session_token>
```

Or with API key for service-to-service / webhook style:

```
wss://totaljsbackend.com/ws/{channel_id}
    ?apikey=<api_key>&token=<session_token>
```

Incoming messages always carry a `type` discriminator. Common types:

```json
{ "type": "message", "data": { ... } }
{ "type": "notification", "data": { "title": "...", "body": "..." } }
{ "type": "status", "state": "connected" }
{ "type": "error", "message": "..." }
```
