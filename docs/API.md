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
