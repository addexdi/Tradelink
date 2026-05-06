# API Overview

**Project:** TradeLink Global  
**Version:** v1  
**Base URL:** `https://api.tradelinkglobal.com/api/v1`  

---

## Conventions

- All requests and responses use **JSON** (`Content-Type: application/json`).
- Authentication is via **Bearer token** in the `Authorization` header:  
  `Authorization: Bearer <access_token>`
- Dates are **ISO 8601** strings (e.g., `2026-05-06T18:00:00Z`).
- Paginated endpoints accept `page` (default: `1`) and `limit` (default: `20`, max: `100`) query parameters and return:

```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 150
  }
}
```

- Standard HTTP status codes are used:

| Code | Meaning |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request (validation error) |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Unprocessable Entity |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

---

## Error Handling

All error responses use a consistent JSON envelope:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request body is invalid.",
    "details": [
      { "field": "email", "issue": "Must be a valid email address." }
    ],
    "request_id": "req_abc123"
  }
}
```

| Field | Description |
|---|---|
| `code` | Machine-readable error code (uppercase snake case) |
| `message` | Human-readable summary |
| `details` | Field-level validation issues (present on 400/422 only) |
| `request_id` | Correlation ID — include this when contacting support |

Include `request_id` in the `X-Request-ID` request header to propagate your own correlation ID through the system.

---

## Authentication

### `POST /auth/register`

Register a new user account.

**Request Body:**

```json
{
  "name": "Amara Diallo",
  "email": "amara@example.com",
  "phone": "+2348012345678",
  "password": "SecurePass123!",
  "role": "buyer"
}
```

**Response `201`:**

```json
{
  "user": {
    "id": "uuid",
    "name": "Amara Diallo",
    "email": "amara@example.com",
    "role": "buyer",
    "verification_level": "unverified"
  },
  "tokens": {
    "access_token": "eyJ...",
    "refresh_token": "eyJ..."
  }
}
```

---

### `POST /auth/login`

Authenticate an existing user.

**Request Body:**

```json
{
  "email": "amara@example.com",
  "password": "SecurePass123!"
}
```

**Response `200`:** Same structure as register response.

---

### `POST /auth/refresh`

Exchange a refresh token for a new access token.

**Request Body:**

```json
{ "refresh_token": "eyJ..." }
```

**Response `200`:**

```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ..."
}
```

---

### `POST /auth/logout`

Revoke the current refresh token.

**Auth required.** Returns `204 No Content`.

---

## Products

### `GET /products`

Browse and search product listings.

**Query Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `q` | string | Keyword search |
| `category` | string | Filter by category |
| `min_price` | number | Minimum unit price |
| `max_price` | number | Maximum unit price |
| `country_of_origin` | string | Filter by origin country |
| `page` | integer | Page number |
| `limit` | integer | Results per page |

**Response `200`:**

```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Shea Butter (Refined)",
      "description": "High-quality refined shea butter...",
      "price": 4.50,
      "currency": "USD",
      "moq": 500,
      "category": "Agriculture",
      "country_of_origin": "Ghana",
      "supplier": {
        "id": "uuid",
        "name": "Accra Naturals Ltd.",
        "verification_level": "verified"
      }
    }
  ],
  "meta": { "page": 1, "limit": 20, "total": 83 }
}
```

---

### `POST /products`

Create a new product listing. **Supplier role required.**

**Request Body:**

```json
{
  "name": "Shea Butter (Refined)",
  "description": "High-quality refined shea butter...",
  "price": 4.50,
  "currency": "USD",
  "moq": 500,
  "category": "Agriculture",
  "country_of_origin": "Ghana"
}
```

**Response `201`:** Returns the created product object.

---

### `GET /products/:id`

Retrieve a single product listing.

**Response `200`:** Full product object including supplier details and rating summary.

---

### `PATCH /products/:id`

Update a product listing. **Supplier (owner) role required.**

---

### `DELETE /products/:id`

Deactivate (soft-delete) a product listing. **Supplier (owner) role required.** Returns `204`.

---

## Orders

### `POST /orders`

Submit a Request for Quotation (RFQ). **Buyer role required.**

**Request Body:**

```json
{
  "supplier_id": "uuid",
  "items": [
    { "product_id": "uuid", "quantity": 500 }
  ],
  "notes": "Please include a sample with the first shipment."
}
```

**Response `201`:** Returns the created order object with status `pending`.

---

### `GET /orders`

List orders for the authenticated user (buyer sees their purchases; supplier sees their sales).

**Query Parameters:** `status`, `page`, `limit`

---

### `GET /orders/:id`

Retrieve full order details including line items and status history.

---

### `PATCH /orders/:id/status`

Update an order's status. Allowed transitions depend on role:

| Role | Allowed Transitions |
|---|---|
| Supplier | `pending` → `quoted`, `confirmed` → `shipped` |
| Buyer | `quoted` → `confirmed`, `shipped` → `completed` or `disputed` |
| Admin | Any transition |

**Request Body:**

```json
{ "status": "quoted", "total_amount": 2250.00 }
```

---

## Messages

### `GET /conversations`

List all conversations for the authenticated user.

---

### `GET /conversations/:conversation_id/messages`

Retrieve paginated message history for a conversation.

---

### `POST /conversations/:conversation_id/messages`

Send a message. Supports JSON body for text and multipart for file attachments.

**Request Body:**

```json
{
  "content": "Can you do 1000 units at $4.20 per kg?"
}
```

**Response `201`:** Returns the created message object.

---

## Organizations (KYB)

### `POST /organizations`

Submit an organization profile for KYB verification. **Supplier role required.**

**Request Body:**

```json
{
  "name": "Accra Naturals Ltd.",
  "registration_no": "GH-2021-00123",
  "tax_id": "GH-TAX-9876",
  "country": "Ghana",
  "address": "45 Cantonments Road, Accra, Ghana"
}
```

---

### `GET /organizations/:id`

Retrieve organization details and verification status.

---

### `PATCH /organizations/:id/verify`

Approve or reject a KYB application. **Admin role required.**

**Request Body:**

```json
{ "verification_status": "verified" }
```

---

## Ratings

### `POST /orders/:id/rating`

Submit a rating for a completed order. **Buyer role required; order must be `completed`.**

**Request Body:**

```json
{
  "score": 5,
  "comment": "Excellent quality and fast shipping."
}
```

**Response `201`:** Returns the created rating object.

---

## WebSocket Events

Real-time messaging is handled over a WebSocket connection authenticated with a valid JWT.

**Connection:** `wss://api.tradelinkglobal.com/ws?token=<access_token>`

| Event | Direction | Payload |
|---|---|---|
| `message:new` | Server → Client | New message object |
| `message:read` | Client → Server | `{ message_id }` |
| `order:status_changed` | Server → Client | Order object with new status |
| `notification:new` | Server → Client | Notification object |

---

## User Profile

### `GET /users/me`

Retrieve the authenticated user's profile.

**Auth required.**

**Response `200`:**

```json
{
  "id": "uuid",
  "name": "Amara Diallo",
  "email": "amara@example.com",
  "phone": "+2348012345678",
  "role": "buyer",
  "verification_level": "unverified",
  "created_at": "2026-01-15T09:00:00Z"
}
```

---

### `PATCH /users/me`

Update the authenticated user's profile.

**Auth required.**

**Request Body** (all fields optional):

```json
{
  "name": "Amara Diallo-Bah",
  "phone": "+2348099887766"
}
```

**Response `200`:** Updated user profile object.

---

### `POST /auth/change-password`

Change the authenticated user's password.

**Auth required.**

**Request Body:**

```json
{
  "current_password": "OldPass123!",
  "new_password": "NewPass456!"
}
```

**Response `200`:**

```json
{ "message": "Password updated successfully." }
```

---

## Categories

### `GET /categories`

Return all available product categories.

**Response `200`:**

```json
{
  "data": [
    { "slug": "agriculture", "label": "Agriculture & Agro-commodities" },
    { "slug": "textiles", "label": "Textiles & Apparel" },
    { "slug": "minerals", "label": "Minerals & Mining" },
    { "slug": "manufacturing", "label": "Manufacturing & Industrial" },
    { "slug": "fmcg", "label": "Fast-Moving Consumer Goods" },
    { "slug": "technology", "label": "Technology & Electronics" },
    { "slug": "health", "label": "Health & Pharmaceuticals" },
    { "slug": "construction", "label": "Construction & Building Materials" },
    { "slug": "energy", "label": "Energy & Utilities" },
    { "slug": "services", "label": "Professional Services" }
  ]
}
```

---

## File Uploads

### `POST /uploads/presign`

Request a pre-signed S3 URL to upload a file directly from the client browser.

**Auth required.**

**Request Body:**

```json
{
  "filename": "product-sample.jpg",
  "content_type": "image/jpeg",
  "purpose": "product_image"
}
```

**`purpose` values:** `product_image`, `kyb_document`, `message_attachment`

**Response `200`:**

```json
{
  "upload_url": "https://s3.amazonaws.com/tradelink-uploads/...",
  "file_key": "uploads/product_images/uuid/product-sample.jpg",
  "expires_in": 300
}
```

The client uploads the file directly to `upload_url` with a `PUT` request, then passes `file_key` to the relevant API endpoint (e.g., `POST /products/:id/images`).

---

### `POST /products/:id/images`

Attach an uploaded image to a product listing. **Supplier (owner) role required.**

**Request Body:**

```json
{
  "s3_key": "uploads/product_images/uuid/product-sample.jpg",
  "is_primary": true
}
```

**Response `201`:**

```json
{
  "id": "uuid",
  "url": "https://cdn.tradelinkglobal.com/product_images/uuid/product-sample.jpg",
  "is_primary": true,
  "sort_order": 0
}
```

---

## Payments

### `POST /orders/:id/payment`

Initiate a payment for a confirmed order. **Buyer role required; order must be `confirmed`.**

**Request Body:**

```json
{
  "gateway": "paystack",
  "callback_url": "https://app.tradelinkglobal.com/orders/uuid/payment/callback"
}
```

**Response `200`:**

```json
{
  "payment_id": "uuid",
  "gateway": "paystack",
  "authorization_url": "https://checkout.paystack.com/...",
  "status": "initiated"
}
```

Redirect the buyer to `authorization_url` to complete payment. The gateway will call the platform's webhook on completion.

---

### `GET /orders/:id/payment`

Get the payment status for an order.

**Response `200`:**

```json
{
  "id": "uuid",
  "order_id": "uuid",
  "gateway": "paystack",
  "gateway_reference": "TRX_ABC123",
  "amount": 2250.00,
  "currency": "USD",
  "status": "successful",
  "paid_at": "2026-05-06T12:30:00Z"
}
```

---

## Disputes

### `POST /orders/:id/dispute`

Raise a dispute for a shipped or delivered order. **Buyer role required.**

**Request Body:**

```json
{
  "reason": "Received wrong product — ordered shea butter but received palm oil."
}
```

**Response `201`:** Returns the created dispute object with status `open`.

---

### `GET /disputes/:id`

Retrieve dispute details.

---

### `PATCH /disputes/:id`

Update dispute status or add resolution. **Admin role required.**

**Request Body:**

```json
{
  "status": "resolved",
  "resolution": "Supplier agreed to a full refund. Refund initiated via Paystack."
}
```

---

## Notifications

### `GET /notifications`

List notifications for the authenticated user.

**Query Parameters:** `read` (`true`/`false`), `page`, `limit`

**Response `200`:**

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "order_status_changed",
      "title": "Your order has been shipped",
      "body": "Supplier has marked order #TL-10042 as shipped.",
      "data": { "order_id": "uuid" },
      "read_at": null,
      "created_at": "2026-05-06T10:00:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 20, "total": 5 }
}
```

---

### `PATCH /notifications/:id/read`

Mark a notification as read. Returns `204 No Content`.

---

### `PATCH /notifications/read-all`

Mark all unread notifications as read. Returns `204 No Content`.

---

## Admin Endpoints

All admin endpoints require **Admin role** and an active MFA-confirmed session.

### `GET /admin/users`

List all registered users with filtering.

**Query Parameters:** `role`, `verification_level`, `is_active`, `q` (name/email search), `page`, `limit`

---

### `PATCH /admin/users/:id`

Update a user's account (e.g., suspend, change role).

**Request Body:**

```json
{
  "is_active": false,
  "verification_level": "unverified"
}
```

---

### `GET /admin/kyb-queue`

List pending KYB applications awaiting review.

**Query Parameters:** `status` (`pending`, `under_review`), `page`, `limit`

---

### `GET /admin/disputes`

List all disputes.

**Query Parameters:** `status`, `page`, `limit`

---

### `GET /admin/audit-logs`

Query the audit log.

**Query Parameters:** `actor_id`, `event_type`, `from`, `to`, `page`, `limit`

**Response `200`:**

```json
{
  "data": [
    {
      "id": "ObjectId",
      "event_type": "LOGIN",
      "actor_id": "uuid",
      "ip_address": "197.210.0.1",
      "created_at": "2026-05-06T08:00:00Z"
    }
  ],
  "meta": { "page": 1, "limit": 20, "total": 4821 }
}
```

---

## Health Check

### `GET /health`

Returns the operational status of the API and its dependencies. Used by load balancer health checks.

**No auth required.**

**Response `200` (healthy):**

```json
{
  "status": "ok",
  "timestamp": "2026-05-06T18:00:00Z",
  "services": {
    "postgres": "ok",
    "mongo": "ok",
    "redis": "ok"
  },
  "version": "1.0.0",
  "sha": "a3f9c1d"
}
```

**Response `503` (degraded):**

```json
{
  "status": "degraded",
  "services": {
    "postgres": "ok",
    "mongo": "error",
    "redis": "ok"
  }
}
```
