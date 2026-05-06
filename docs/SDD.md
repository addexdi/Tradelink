# Software Design Document (SDD)

**Project:** TradeLink Global  
**Subtitle:** Digital Trade Infrastructure for African Businesses  
**Version:** 1.0  
**Date:** 2026-05-06  

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Functional Requirements](#3-functional-requirements)
4. [Non-Functional Requirements](#4-non-functional-requirements)
5. [Technical Stack](#5-technical-stack)
6. [System Architecture](#6-system-architecture)
7. [Key Data Flows](#7-key-data-flows)
8. [Error Handling Strategy](#8-error-handling-strategy)
9. [Database Schema](#9-database-schema)
10. [Security Design](#10-security-design)
11. [Roadmap](#11-roadmap)
12. [Risk Mitigation](#12-risk-mitigation)
13. [Glossary](#13-glossary)
14. [Revision History](#14-revision-history)

---

## 1. Introduction

### 1.1 Purpose

This document defines the system architecture, technical specifications, and implementation roadmap for **TradeLink Global**. It serves as a guide for developers, stakeholders, and project managers throughout the development lifecycle.

### 1.2 Scope

TradeLink Global is a **B2B marketplace platform** that combines:

- Secure buyer–supplier communication
- Searchable product discovery with verified suppliers
- End-to-end order management and logistics tracking

The platform is specifically tailored for **African cross-border and local trade**, addressing challenges around supplier verification, trust, and trade finance.

### 1.3 Intended Audience

| Audience | Usage |
|---|---|
| Software Engineers | Implementation reference and API contract |
| Product Managers | Feature scope and phased delivery |
| Security Team | Security controls and compliance review |
| Business Stakeholders | System capabilities and roadmap alignment |

---

## 2. System Overview

TradeLink Global employs a **client-server architecture** with three primary tiers:

1. **Presentation Tier** — React web application and React Native / Flutter mobile application.
2. **Application Tier** — RESTful backend API handling business logic, authentication, and integrations.
3. **Data Tier** — PostgreSQL for relational data and MongoDB for unstructured data (chat, logs).

### 2.1 User Roles

| Role | Description |
|---|---|
| **Buyer** | Businesses seeking to procure products from verified suppliers |
| **Supplier** | Verified businesses listing products, managing inventory, and fulfilling orders |
| **Admin** | System operators managing KYB verification, dispute resolution, and platform health |

### 2.2 High-Level Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Clients                              │
│          React Web App       React Native / Flutter         │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTPS / WebSocket
┌────────────────────▼────────────────────────────────────────┐
│                    API Gateway / Load Balancer               │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                  Backend API (Node.js / Django)              │
│  ┌──────────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐  │
│  │  Auth Service│ │Marketplace│ │ Messaging │ │  Orders  │  │
│  └──────────────┘ └───────────┘ └──────────┘ └──────────┘  │
└────────┬───────────────────────────────────────┬────────────┘
         │                                       │
┌────────▼──────────┐                 ┌──────────▼───────────┐
│   PostgreSQL DB   │                 │      MongoDB          │
│  (Primary Store)  │                 │  (Chat / Audit Logs)  │
└───────────────────┘                 └──────────────────────┘
```

---

## 3. Functional Requirements

### 3.1 Authentication and Authorization

- Users can register and log in via **email or phone number**.
- Sessions are managed with **JSON Web Tokens (JWT)** — stateless, short-lived access tokens with refresh token rotation.
- Role-based access control (RBAC) enforces permissions for Buyer, Supplier, and Admin roles.
- Password storage uses a strong one-way hashing algorithm (e.g., bcrypt with a high work factor).

### 3.2 Marketplace

- Suppliers can create, update, and deactivate product listings.
- Buyers can **browse and search** products by keyword, category, price range, and MOQ.
- Each product listing displays:
  - Name, description, images
  - Unit price and MOQ
  - Supplier name and verification badge
  - Category and origin country
- Paginated search results with sorting options (price, relevance, date).

### 3.3 Messaging

- Real-time **negotiation chat** between buyers and suppliers using WebSockets.
- Messages are persisted for audit and dispute resolution.
- Notifications (push / email) for new messages.
- File attachments (e.g., product samples, trade documents) are supported.

### 3.4 Order Management

The order lifecycle follows these statuses:

```
RFQ Submitted → Quotation Received → Order Confirmed
    → Payment Processed → Shipped → Delivered → Completed
                                              ↘ Disputed
```

| Status | Description |
|---|---|
| Pending | RFQ submitted, awaiting supplier response |
| Quoted | Supplier has provided a quotation |
| Confirmed | Buyer has accepted the quotation |
| Paid | Payment has been processed |
| Shipped | Order has been dispatched |
| Completed | Buyer has confirmed receipt |
| Disputed | Order under dispute review |

### 3.5 Trust System

- **KYB (Know Your Business)** verification collects and validates business registration documents, tax IDs, and director information.
- Verified suppliers display a **trust badge** on their profile and listings.
- A **rating and review system** allows buyers to rate suppliers after order completion.
- Verification tiers:
  - `Unverified` — Self-registered, basic access
  - `Verified` — KYB documents submitted and reviewed
  - `Premium` — Enhanced due diligence completed

---

## 4. Non-Functional Requirements

### 4.1 Performance

| Metric | Target |
|---|---|
| API response time (p95) | < 200 ms for read endpoints |
| API response time (p99) | < 500 ms for all endpoints |
| Product search latency | < 300 ms at p95 |
| WebSocket message delivery | < 100 ms end-to-end on same continent |
| Concurrent users supported | ≥ 10,000 without degradation |
| Throughput | ≥ 1,000 API requests/second |

### 4.2 Availability and Reliability

| Metric | Target |
|---|---|
| Platform uptime SLA | 99.9% (≤ 8.7 hours downtime/year) |
| RTO (Recovery Time Objective) | < 1 hour for any single-component failure |
| RPO (Recovery Point Objective) | < 5 minutes for PostgreSQL data |
| Deployment downtime | Zero-downtime blue/green deployments |

### 4.3 Scalability

- The system must scale horizontally to 5× baseline load within 10 minutes via auto-scaling.
- Database design must support 10 million product listings and 1 million active users without architectural change.

### 4.4 Security

- Zero critical or high CVEs in production at any time.
- All user data encrypted at rest and in transit.
- Full audit trail for all financial and verification events.

### 4.5 Maintainability

- Code test coverage ≥ 80% for all business logic.
- All public API endpoints documented in [API.md](API.md).
- All schema changes delivered via versioned migrations.
- Mean time to deploy a hotfix to production: < 30 minutes.

### 4.6 Usability and Accessibility

- Web application meets **WCAG 2.1 Level AA** accessibility standards.
- Marketplace search returns relevant results with ≥ 80% user satisfaction in usability testing.
- Mobile app supports Android 11+ and iOS 15+.

---

## 5. Technical Stack

| Layer | Technology | Justification |
|---|---|---|
| Web Frontend | React | Wide ecosystem, component reusability |
| Mobile | React Native / Flutter | Cross-platform mobile from a single codebase |
| Backend | Node.js (Express) or Django | High-throughput API; Django offers built-in admin and ORM |
| Primary Database | PostgreSQL | ACID compliance for transactional trade data |
| Chat & Logs | MongoDB | Schema-flexible storage for messages and event logs |
| Real-time | WebSockets (Socket.IO or Django Channels) | Low-latency messaging |
| Hosting | AWS or Google Cloud | Scalable, globally distributed infrastructure |
| Containerization | Docker | Consistent deployment environments |
| CI/CD | GitHub Actions / Cloud Pipelines | Automated build, test, and deploy |
| CDN | AWS CloudFront or GCP Cloud CDN | Asset delivery and DDoS mitigation |
| Object Storage | AWS S3 or GCP Cloud Storage | Product images and document uploads |

---

## 6. System Architecture

### 6.1 Deployment Architecture

All services are containerized with Docker and orchestrated via Kubernetes (or AWS ECS) for horizontal scalability.

```
Internet → CDN → Load Balancer → API Servers (auto-scaled)
                                       ↓
                              Message Queue (e.g., Redis / SQS)
                                       ↓
                         Background Workers (notifications, KYB)
```

### 6.2 API Design Principles

- **RESTful** conventions (resource-based URLs, HTTP verbs, standard status codes).
- **JSON** request and response bodies.
- Versioned endpoints (e.g., `/api/v1/`).
- Idempotent design for order and payment endpoints.

See [API Overview](API.md) for the full endpoint reference.

### 6.3 Scalability Considerations

- Stateless API servers allow horizontal scaling behind a load balancer.
- Read replicas for PostgreSQL reduce load on the primary node.
- Redis caching layer for frequently accessed data (product listings, search results).
- Asynchronous processing (message queues) for non-critical background jobs (emails, KYB checks).

---

## 7. Key Data Flows

### 7.1 Buyer Registration and Login

```
Client → POST /auth/register → Validate input → Hash password (bcrypt)
       → Insert user row → Issue JWT pair → Return tokens to client
```

### 7.2 Product Search

```
Client → GET /products?q=shea&category=Agriculture
       → API validates query params
       → Query PostgreSQL full-text index (with Redis cache check first)
       → Paginate and enrich results with supplier verification badge
       → Return JSON response
```

### 7.3 RFQ-to-Order Flow

```
Buyer → POST /orders (RFQ with product list)
      → API creates order (status: pending) + order_items rows
      → Notify supplier (WebSocket push + email)
      → Supplier responds → PATCH /orders/:id/status (status: quoted, total_amount)
      → Buyer reviews → PATCH /orders/:id/status (status: confirmed)
      → Buyer pays → Payment gateway webhook → PATCH status: paid
      → Supplier ships → PATCH status: shipped
      → Buyer confirms → PATCH status: completed
      → System prompts buyer for rating
```

### 7.4 KYB Verification Flow

```
Supplier → POST /organizations (business details + documents)
         → Documents uploaded to S3 (pre-signed URL)
         → Admin notified of new KYB submission
         → Admin reviews in dashboard → PATCH /organizations/:id/verify
         → Supplier notified of decision (email + in-app)
         → verification_level updated on users row
```

### 7.5 Real-Time Messaging

```
Client A → wss://api.tradelinkglobal.com/ws (JWT auth)
         → Sends message event
         → Server persists to MongoDB
         → Server pushes message:new event to Client B's WebSocket
         → Client B marks as read → Sends message:read event
         → Server updates read_at in MongoDB
```

---

## 8. Error Handling Strategy

### 8.1 Standard Error Envelope

All error responses follow a consistent JSON structure:

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
| `details` | Array of field-level validation errors (optional) |
| `request_id` | Correlation ID for log tracing |

### 8.2 Error Code Reference

| HTTP Status | Code | Meaning |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Request body or query params failed schema validation |
| 401 | `UNAUTHORIZED` | Missing or expired authentication token |
| 403 | `FORBIDDEN` | Authenticated user lacks permission for this action |
| 404 | `NOT_FOUND` | Requested resource does not exist |
| 409 | `CONFLICT` | Duplicate resource or invalid state transition |
| 422 | `UNPROCESSABLE` | Valid request but business rule violation |
| 429 | `RATE_LIMITED` | Too many requests; includes `Retry-After` header |
| 500 | `INTERNAL_ERROR` | Unexpected server error; use `request_id` for support |

### 8.3 Error Handling Principles

- All errors are logged with the `request_id`, user ID (if authenticated), endpoint, and timestamp.
- Internal error details (stack traces, database errors) are **never** exposed to API consumers.
- Unhandled promise rejections and uncaught exceptions trigger an alert and graceful shutdown.
- Idempotency keys prevent duplicate order/payment creation on client retry.

---

## 9. Database Schema

See [Database Schema](DATABASE_SCHEMA.md) for the full relational and document store schemas.

### Summary of Core Entities

| Table / Collection | Primary Purpose |
|---|---|
| `users` | Authentication and profile data |
| `organizations` | Business entity and KYB verification |
| `products` | Supplier product listings |
| `product_images` | Product listing images |
| `orders` | Order lifecycle and status tracking |
| `order_items` | Line items within an order |
| `order_status_history` | Immutable audit trail of status changes |
| `payments` | Payment transaction records |
| `disputes` | Buyer-initiated dispute cases |
| `notifications` | In-app notification feed |
| `ratings` | Buyer reviews for suppliers |
| `messages` | Real-time chat messages (MongoDB) |
| `audit_logs` | Security and compliance event log (MongoDB) |

---

## 10. Security Design

See [Security Design](SECURITY.md) for the full security specification.

### Summary

| Control | Implementation |
|---|---|
| Authentication | JWT (access + refresh token rotation) |
| Transport Security | TLS 1.2+ / HTTPS enforced |
| Data at Rest | AES-256 encryption for sensitive fields |
| Input Validation | Server-side schema validation; parameterized queries |
| Rate Limiting | Per-IP and per-user rate limits on all endpoints |
| Audit Logging | Immutable logs for authentication events and order changes |

---

## 11. Roadmap

See [Roadmap](ROADMAP.md) for the full phased delivery plan.

| Phase | Focus |
|---|---|
| Phase 1 | MVP — Core marketplace and messaging |
| Phase 2 | Trust & Finance — KYB verification and payment gateways |
| Phase 3 | Logistics — 3PL provider integrations |
| Phase 4 | Optimization — AI recommendations and multilingual support |

---

## 12. Risk Mitigation

| Risk | Likelihood | Impact | Mitigation Strategy |
|---|---|---|---|
| Fraudulent suppliers | High | High | KYB verification tiers; trade assurance escrow |
| Platform downtime | Medium | High | 99.9% uptime SLA; redundant multi-AZ deployments |
| Data breach | Low | Critical | Encryption at rest and in transit; regular penetration testing |
| Payment failures | Medium | High | Multiple payment gateways (Paystack, Flutterwave); retry logic |
| Regulatory non-compliance | Medium | High | Jurisdiction-specific KYB flows; legal review per market |
| DDoS attacks | Medium | Medium | CDN-level rate limiting; AWS Shield or GCP Cloud Armor |
| Scalability bottlenecks | Low | Medium | Auto-scaling groups; database read replicas; caching |

---

## 13. Glossary

| Term | Definition |
|---|---|
| **B2B** | Business-to-Business — trade between companies, not individual consumers |
| **MOQ** | Minimum Order Quantity — the smallest number of units a supplier will sell in one order |
| **RFQ** | Request for Quotation — a formal buyer request for a price and terms from a supplier |
| **KYB** | Know Your Business — verification process to confirm a business's legal identity and legitimacy |
| **3PL** | Third-Party Logistics — an outsourced provider of shipping, warehousing, and fulfilment services |
| **JWT** | JSON Web Token — a compact, URL-safe means of representing claims between two parties |
| **RBAC** | Role-Based Access Control — restricting system access based on the user's role |
| **TLS** | Transport Layer Security — cryptographic protocol for securing data in transit |
| **AES-256** | Advanced Encryption Standard with a 256-bit key — symmetric encryption standard |
| **TOTP** | Time-based One-Time Password — MFA method using a rotating 6-digit code |
| **GMV** | Gross Merchandise Value — total value of goods transacted on the platform |
| **SLA** | Service Level Agreement — commitment to a minimum level of service uptime or performance |
| **RTO** | Recovery Time Objective — maximum acceptable time to restore service after an outage |
| **RPO** | Recovery Point Objective — maximum acceptable data loss measured in time |
| **CDN** | Content Delivery Network — geographically distributed network for fast asset delivery |
| **PCI-DSS** | Payment Card Industry Data Security Standard — security standard for handling card data |
| **NDPR** | Nigeria Data Protection Regulation — Nigerian personal data privacy law |
| **GDPR** | General Data Protection Regulation — EU personal data privacy law |
| **SIEM** | Security Information and Event Management — system for real-time security monitoring |
| **WAL** | Write-Ahead Log — PostgreSQL mechanism for durability and point-in-time recovery |

---

## 14. Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-05-06 | TradeLink Engineering | Initial document created |
