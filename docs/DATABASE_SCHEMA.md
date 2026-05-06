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

## PostgreSQL Schema — Additional Tables

### `payments`

Records payment transactions associated with orders.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique payment identifier |
| `order_id` | UUID | FOREIGN KEY → `orders.id` | Associated order |
| `gateway` | VARCHAR(50) | NOT NULL | Payment gateway used (`paystack`, `flutterwave`) |
| `gateway_reference` | VARCHAR(255) | UNIQUE | External transaction reference from gateway |
| `amount` | NUMERIC(14, 2) | NOT NULL | Amount charged |
| `currency` | CHAR(3) | NOT NULL | ISO 4217 currency code |
| `status` | ENUM | NOT NULL DEFAULT `initiated` | `initiated`, `pending`, `successful`, `failed`, `refunded` |
| `paid_at` | TIMESTAMP | | Timestamp of successful payment |
| `metadata` | JSONB | | Gateway-specific response data |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Record creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Last update timestamp |

**Indexes:**
- `(order_id)` — for order payment lookup
- `(gateway_reference)` — for webhook reconciliation
- `(status, created_at DESC)` — for payment reporting

---

### `disputes`

Tracks buyer-initiated dispute cases for orders.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique dispute identifier |
| `order_id` | UUID | FOREIGN KEY → `orders.id`, UNIQUE | One active dispute per order |
| `raised_by_id` | UUID | FOREIGN KEY → `users.id` | User who raised the dispute |
| `reason` | TEXT | NOT NULL | Buyer's description of the issue |
| `status` | ENUM | NOT NULL DEFAULT `open` | `open`, `under_review`, `resolved`, `closed` |
| `resolution` | TEXT | | Admin's resolution notes |
| `resolved_by_id` | UUID | FOREIGN KEY → `users.id` | Admin who resolved the dispute |
| `resolved_at` | TIMESTAMP | | Timestamp of resolution |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Dispute creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Last update timestamp |

---

### `notifications`

Stores in-app and push notifications for users.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique notification identifier |
| `user_id` | UUID | FOREIGN KEY → `users.id` | Recipient user |
| `type` | VARCHAR(100) | NOT NULL | Notification type (e.g., `order_status_changed`, `message_received`) |
| `title` | VARCHAR(255) | NOT NULL | Short notification title |
| `body` | TEXT | NOT NULL | Notification body |
| `data` | JSONB | | Machine-readable payload (e.g., `{ "order_id": "..." }`) |
| `read_at` | TIMESTAMP | | Timestamp when user read the notification |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Notification creation timestamp |

**Indexes:**
- `(user_id, read_at, created_at DESC)` — for unread notification feed

---

### `product_images`

Stores image metadata for product listings (multiple images per product).

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique image identifier |
| `product_id` | UUID | FOREIGN KEY → `products.id` ON DELETE CASCADE | Parent product |
| `url` | TEXT | NOT NULL | Public CDN URL of the image |
| `s3_key` | TEXT | NOT NULL | Object key in S3 for deletion management |
| `is_primary` | BOOLEAN | NOT NULL DEFAULT FALSE | Whether this is the primary listing image |
| `sort_order` | SMALLINT | NOT NULL DEFAULT 0 | Display order |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Upload timestamp |

**Indexes:**
- `(product_id, sort_order)` — for ordered image retrieval

---

### `order_status_history`

Immutable audit trail of all order status transitions.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | PRIMARY KEY | Unique record identifier |
| `order_id` | UUID | FOREIGN KEY → `orders.id` | Associated order |
| `from_status` | VARCHAR(50) | | Previous status (`null` for initial creation) |
| `to_status` | VARCHAR(50) | NOT NULL | New status |
| `changed_by_id` | UUID | FOREIGN KEY → `users.id` | User who triggered the change |
| `note` | TEXT | | Optional note explaining the transition |
| `created_at` | TIMESTAMP | NOT NULL DEFAULT NOW() | Transition timestamp |

**Indexes:**
- `(order_id, created_at)` — for order history queries

---

## SQL DDL — Core Tables

The following DDL illustrates the PostgreSQL schema for the primary tables. These are generated by migration files — do not execute manually.

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TYPE user_role AS ENUM ('buyer', 'supplier', 'admin');
CREATE TYPE verification_level AS ENUM ('unverified', 'verified', 'premium');
CREATE TYPE order_status AS ENUM (
  'pending', 'quoted', 'confirmed', 'paid',
  'shipped', 'completed', 'disputed', 'cancelled'
);
CREATE TYPE payment_status AS ENUM (
  'initiated', 'pending', 'successful', 'failed', 'refunded'
);
CREATE TYPE dispute_status AS ENUM (
  'open', 'under_review', 'resolved', 'closed'
);
CREATE TYPE kyb_status AS ENUM (
  'pending', 'under_review', 'verified', 'rejected'
);

CREATE TABLE users (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name             VARCHAR(255)        NOT NULL,
  phone            VARCHAR(20)         UNIQUE,
  email            VARCHAR(255)        UNIQUE,
  password_hash    TEXT                NOT NULL,
  role             user_role           NOT NULL,
  verification_level verification_level NOT NULL DEFAULT 'unverified',
  is_active        BOOLEAN             NOT NULL DEFAULT TRUE,
  created_at       TIMESTAMPTZ         NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ         NOT NULL DEFAULT NOW(),
  CONSTRAINT users_contact_check CHECK (phone IS NOT NULL OR email IS NOT NULL)
);

CREATE TABLE organizations (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             UUID            NOT NULL REFERENCES users(id),
  name                VARCHAR(255)    NOT NULL,
  registration_no     VARCHAR(100)    UNIQUE,
  tax_id              VARCHAR(100),
  country             VARCHAR(100)    NOT NULL,
  address             TEXT,
  verification_status kyb_status      NOT NULL DEFAULT 'pending',
  verified_at         TIMESTAMPTZ,
  created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  supplier_id       UUID            NOT NULL REFERENCES users(id),
  name              VARCHAR(255)    NOT NULL,
  description       TEXT,
  price             NUMERIC(14,2)   NOT NULL,
  currency          CHAR(3)         NOT NULL DEFAULT 'USD',
  moq               INTEGER         NOT NULL DEFAULT 1,
  category          VARCHAR(100)    NOT NULL,
  country_of_origin VARCHAR(100),
  is_active         BOOLEAN         NOT NULL DEFAULT TRUE,
  created_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category    ON products(category);
CREATE INDEX idx_products_supplier    ON products(supplier_id);
CREATE INDEX idx_products_search      ON products USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

CREATE TABLE orders (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  buyer_id      UUID            NOT NULL REFERENCES users(id),
  supplier_id   UUID            NOT NULL REFERENCES users(id),
  status        order_status    NOT NULL DEFAULT 'pending',
  total_amount  NUMERIC(14,2),
  currency      CHAR(3)         NOT NULL DEFAULT 'USD',
  notes         TEXT,
  created_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
```

---

## Migration Strategy

- All schema changes are managed through **numbered migration files** in `migrations/`.
- Migrations are applied sequentially and are idempotent.
- **Backward-compatible** changes (adding nullable columns, new indexes) may be applied without downtime.
- **Breaking changes** (column renames, type changes, dropping columns) require a multi-step migration with a compatibility window.
- A migration dry-run is always executed against the staging database before production.

## Backup Strategy

| Database | Method | Frequency | Retention | Restore Target |
|---|---|---|---|---|
| PostgreSQL (RDS) | Automated snapshots + WAL archiving | Daily snapshots; continuous WAL | 30 days | Point-in-time to within 5 minutes |
| MongoDB | `mongodump` to S3 | Daily | 30 days | Last successful dump |
| Redis | Redis persistence (AOF) | Continuous | N/A | Last flush on restart |

Backup restoration is tested quarterly as part of the disaster recovery drill.

---

## Entity Relationship Diagram (Summary)

```
users ──< orders (as buyer)
users ──< orders (as supplier)
users ──< products (as supplier)
users ──< organizations
users ──< ratings (as reviewer)
users ──< ratings (as reviewed)
users ──< notifications
orders ──< order_items
orders ──< order_status_history
orders ──< payments
orders ──  disputes (one-to-one)
orders ──< ratings (one-to-one)
products ──< order_items
products ──< product_images
```
