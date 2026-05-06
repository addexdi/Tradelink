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
4. [Technical Stack](#4-technical-stack)
5. [System Architecture](#5-system-architecture)
6. [Database Schema](#6-database-schema)
7. [Security Design](#7-security-design)
8. [Roadmap](#8-roadmap)
9. [Risk Mitigation](#9-risk-mitigation)

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

## 4. Technical Stack

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

## 5. System Architecture

### 5.1 Deployment Architecture

All services are containerized with Docker and orchestrated via Kubernetes (or AWS ECS) for horizontal scalability.

```
Internet → CDN → Load Balancer → API Servers (auto-scaled)
                                       ↓
                              Message Queue (e.g., Redis / SQS)
                                       ↓
                         Background Workers (notifications, KYB)
```

### 5.2 API Design Principles

- **RESTful** conventions (resource-based URLs, HTTP verbs, standard status codes).
- **JSON** request and response bodies.
- Versioned endpoints (e.g., `/api/v1/`).
- Idempotent design for order and payment endpoints.

See [API Overview](API.md) for the full endpoint reference.

### 5.3 Scalability Considerations

- Stateless API servers allow horizontal scaling behind a load balancer.
- Read replicas for PostgreSQL reduce load on the primary node.
- Redis caching layer for frequently accessed data (product listings, search results).
- Asynchronous processing (message queues) for non-critical background jobs (emails, KYB checks).

---

## 6. Database Schema

See [Database Schema](DATABASE_SCHEMA.md) for the full relational and document store schemas.

### Summary of Core Entities

| Table / Collection | Primary Purpose |
|---|---|
| `users` | Authentication and profile data |
| `organizations` | Business entity and KYB verification |
| `products` | Supplier product listings |
| `orders` | Order lifecycle and status tracking |
| `order_items` | Line items within an order |
| `messages` | Real-time chat messages (MongoDB) |
| `ratings` | Buyer reviews for suppliers |

---

## 7. Security Design

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

## 8. Roadmap

See [Roadmap](ROADMAP.md) for the full phased delivery plan.

| Phase | Focus |
|---|---|
| Phase 1 | MVP — Core marketplace and messaging |
| Phase 2 | Trust & Finance — KYB verification and payment gateways |
| Phase 3 | Logistics — 3PL provider integrations |
| Phase 4 | Optimization — AI recommendations and multilingual support |

---

## 9. Risk Mitigation

| Risk | Likelihood | Impact | Mitigation Strategy |
|---|---|---|---|
| Fraudulent suppliers | High | High | KYB verification tiers; trade assurance escrow |
| Platform downtime | Medium | High | 99.9% uptime SLA; redundant multi-AZ deployments |
| Data breach | Low | Critical | Encryption at rest and in transit; regular penetration testing |
| Payment failures | Medium | High | Multiple payment gateways (Paystack, Flutterwave); retry logic |
| Regulatory non-compliance | Medium | High | Jurisdiction-specific KYB flows; legal review per market |
| DDoS attacks | Medium | Medium | CDN-level rate limiting; AWS Shield or GCP Cloud Armor |
| Scalability bottlenecks | Low | Medium | Auto-scaling groups; database read replicas; caching |
