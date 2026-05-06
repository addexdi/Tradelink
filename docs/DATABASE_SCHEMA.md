# Database Schema

**Project:** TradeLink Global  
**Version:** 1.0  

---

## Overview

TradeLink Global uses two database technologies:

- **PostgreSQL** — Primary relational store for all transactional data (users, products, orders, organizations).
- **MongoDB** — Document store for unstructured or high-volume data (messages, audit/event logs).

---

## PostgreSQL Schema

### `users`

Stores authentication credentials and profile data for all platform users.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique user identifier |
| `name` | VARCHAR(255) | NOT NULL | Full display name |
| `phone` | VARCHAR(20) | UNIQUE | Phone number (E.164 format) |
| `email` | VARCHAR(255) | UNIQUE | Email address |
| `password_hash` | TEXT | NOT NULL | bcrypt-hashed password |
| `role` | ENUM | NOT NULL | `buyer`, `supplier`, `admin` |
| `verification_level` | ENUM | NOT NULL DEFAULT `unverified` | `unverified`, `verified`, `premium` |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | Soft-delete / account suspension flag |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Account creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Last update timestamp |

> Either `phone` or `email` must be provided (enforced at the application layer).

---

### `organizations`

Stores business entity information and KYB (Know Your Business) verification status for suppliers.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique organization identifier |
| `user_id` | UUID | FOREIGN KEY → `users.id` | Owner / primary contact |
| `name` | VARCHAR(255) | NOT NULL | Registered business name |
| `registration_no` | VARCHAR(100) | UNIQUE | Company registration number |
| `tax_id` | VARCHAR(100) | | Tax identification number |
| `country` | VARCHAR(100) | NOT NULL | Country of registration |
| `address` | TEXT | | Registered business address |
| `verification_status` | ENUM | NOT NULL DEFAULT `pending` | `pending`, `under_review`, `verified`, `rejected` |
| `verified_at` | TIMESTAMP | | Timestamp of verification approval |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Record creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Last update timestamp |

---

### `products`

Stores product listings created by verified suppliers.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique product identifier |
| `supplier_id` | UUID | FOREIGN KEY → `users.id` | Supplier who owns the listing |
| `name` | VARCHAR(255) | NOT NULL | Product name |
| `description` | TEXT | | Detailed product description |
| `price` | NUMERIC(14, 2) | NOT NULL | Unit price |
| `currency` | CHAR(3) | NOT NULL DEFAULT `USD` | ISO 4217 currency code |
| `moq` | INTEGER | NOT NULL DEFAULT 1 | Minimum Order Quantity |
| `category` | VARCHAR(100) | NOT NULL | Product category |
| `country_of_origin` | VARCHAR(100) | | Country where the product is manufactured |
| `is_active` | BOOLEAN | NOT NULL DEFAULT TRUE | Listing active / deactivated flag |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Listing creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Last update timestamp |

**Indexes:**
- `(category)` — for category browse queries
- `(supplier_id)` — for supplier product listings
- Full-text index on `(name, description)` — for keyword search

---

### `orders`

Tracks the lifecycle of each trade transaction from RFQ to completion.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique order identifier |
| `buyer_id` | UUID | FOREIGN KEY → `users.id` | Buyer placing the order |
| `supplier_id` | UUID | FOREIGN KEY → `users.id` | Supplier fulfilling the order |
| `status` | ENUM | NOT NULL DEFAULT `pending` | See order statuses below |
| `total_amount` | NUMERIC(14, 2) | | Agreed total order value |
| `currency` | CHAR(3) | NOT NULL DEFAULT `USD` | ISO 4217 currency code |
| `notes` | TEXT | | Buyer notes or special instructions |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Order creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Last status update timestamp |

**Order Status Values:**

| Status | Description |
|---|---|
| `pending` | RFQ submitted, awaiting supplier response |
| `quoted` | Supplier has provided a quotation |
| `confirmed` | Buyer has accepted the quotation |
| `paid` | Payment successfully processed |
| `shipped` | Order has been dispatched by the supplier |
| `completed` | Buyer has confirmed receipt |
| `disputed` | Order is under dispute review |
| `cancelled` | Order has been cancelled |

---

### `order_items`

Line items associated with a single order (supports multi-product orders).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique line item identifier |
| `order_id` | UUID | FOREIGN KEY → `orders.id` | Parent order |
| `product_id` | UUID | FOREIGN KEY → `products.id` | Product being ordered |
| `quantity` | INTEGER | NOT NULL | Number of units |
| `unit_price` | NUMERIC(14, 2) | NOT NULL | Agreed unit price at time of order |
| `subtotal` | NUMERIC(14, 2) | GENERATED | `quantity × unit_price` |

---

### `ratings`

Buyer reviews and ratings submitted after order completion.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique rating identifier |
| `order_id` | UUID | FOREIGN KEY → `orders.id`, UNIQUE | One review per completed order |
| `reviewer_id` | UUID | FOREIGN KEY → `users.id` | User submitting the review |
| `reviewed_id` | UUID | FOREIGN KEY → `users.id` | User being reviewed |
| `score` | SMALLINT | NOT NULL CHECK (1–5) | Star rating (1 = lowest, 5 = highest) |
| `comment` | TEXT | | Written review |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Review submission timestamp |

---

## MongoDB Collections

### `messages`

Stores real-time chat messages exchanged between buyers and suppliers.

```json
{
  "_id": "ObjectId",
  "conversation_id": "string (buyer_id:supplier_id ordered)",
  "sender_id": "UUID (ref: users.id)",
  "receiver_id": "UUID (ref: users.id)",
  "content": "string",
  "attachments": [
    {
      "filename": "string",
      "url": "string",
      "mime_type": "string"
    }
  ],
  "read_at": "ISODate | null",
  "created_at": "ISODate"
}
```

**Indexes:**
- `{ conversation_id: 1, created_at: -1 }` — for message history queries
- `{ receiver_id: 1, read_at: 1 }` — for unread message counts

---

### `audit_logs`

Immutable event log for security and compliance auditing.

```json
{
  "_id": "ObjectId",
  "event_type": "string (e.g., LOGIN, ORDER_STATUS_CHANGE, KYB_APPROVED)",
  "actor_id": "UUID | null",
  "target_id": "UUID | null",
  "metadata": {},
  "ip_address": "string",
  "user_agent": "string",
  "created_at": "ISODate"
}
```

**Indexes:**
- `{ actor_id: 1, created_at: -1 }` — for user activity queries
- `{ event_type: 1, created_at: -1 }` — for event-type filtering

---

## Entity Relationship Diagram (Summary)

```
users ──< orders (as buyer)
users ──< orders (as supplier)
users ──< products (as supplier)
users ──< organizations
users ──< ratings (as reviewer)
users ──< ratings (as reviewed)
orders ──< order_items
products ──< order_items
orders ──< ratings (one-to-one)
```
