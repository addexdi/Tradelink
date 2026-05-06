# Security Design

**Project:** TradeLink Global  
**Version:** 1.0  

---

## Overview

Security is a foundational concern for TradeLink Global given the financial and business-sensitive nature of cross-border trade transactions. This document describes the security controls applied across authentication, data protection, transport, input handling, and operational practices.

---

## 1. Authentication and Session Management

### 1.1 JSON Web Tokens (JWT)

TradeLink Global uses a **stateless JWT-based authentication** scheme:

| Token | Lifetime | Storage |
|---|---|---|
| Access Token | 15 minutes | Memory (not localStorage) |
| Refresh Token | 30 days (sliding) | HttpOnly, Secure cookie |

- Access tokens contain: `user_id`, `role`, `iat`, `exp`.
- Refresh token rotation is enforced — every use of a refresh token issues a new refresh token and invalidates the old one.
- A token revocation list (Redis-backed) is maintained to support immediate logout and compromised token invalidation.

### 1.2 Password Security

- Passwords are hashed using **bcrypt** with a minimum work factor of 12.
- Passwords are never stored, logged, or transmitted in plaintext.
- Minimum password policy: 8 characters, including upper-case, lower-case, digit, and special character.

### 1.3 Multi-Factor Authentication (MFA)

- MFA via **TOTP** (Time-based One-Time Password) is available for all users.
- MFA is **required** for Admin accounts.
- SMS OTP is supported as an alternative channel for markets with low smartphone penetration.

---

## 2. Transport Security

- All HTTP traffic is redirected to **HTTPS** (TLS 1.2 minimum, TLS 1.3 preferred).
- **HTTP Strict Transport Security (HSTS)** is enforced with a minimum `max-age` of one year.
- WebSocket connections require `wss://` (TLS-encrypted).
- TLS certificates are provisioned and auto-renewed via Let's Encrypt or AWS ACM.

---

## 3. Data Encryption

### 3.1 Data in Transit

- All API communications are encrypted with TLS.
- Internal service-to-service communication within the cluster is encrypted (mTLS via service mesh).

### 3.2 Data at Rest

- PostgreSQL volumes are encrypted at rest using **AES-256** (cloud provider managed keys).
- MongoDB volumes are encrypted at rest using **AES-256**.
- Object storage (S3 / GCS) uses server-side encryption (SSE-S3 or SSE-KMS).
- Particularly sensitive fields (e.g., `tax_id`, `registration_no`) are encrypted at the application layer before storage using a dedicated secrets manager (AWS KMS / GCP Cloud KMS).

---

## 4. Input Validation and Injection Prevention

- All incoming request bodies are validated against a strict **JSON schema** before reaching business logic.
- The API rejects requests with unexpected fields (strict mode validation).
- Database queries use **parameterized statements / ORM queries** exclusively — raw string interpolation is prohibited.
- File uploads are validated for MIME type, file size (max 10 MB), and content (magic-byte verification). Files are stored in isolated object storage, not the application server.
- HTML content is never rendered from user-supplied data without sanitization (prevents XSS).

---

## 5. Rate Limiting and Abuse Prevention

| Endpoint Group | Limit |
|---|---|
| Authentication (`/auth/*`) | 10 requests / minute per IP |
| Search and browse | 60 requests / minute per user |
| Messaging | 100 messages / minute per conversation |
| Order submission | 20 requests / minute per user |
| All other endpoints | 120 requests / minute per user |

- Rate limiting is enforced at the **API Gateway / Load Balancer** level using a sliding window algorithm.
- Repeated authentication failures trigger a **progressive lockout** (exponential back-off) and optional CAPTCHA challenge.
- **AWS Shield** or **GCP Cloud Armor** provides DDoS mitigation at the CDN/network edge.

---

## 6. Role-Based Access Control (RBAC)

Every API endpoint enforces role-based access:

| Resource | Buyer | Supplier | Admin |
|---|---|---|---|
| Browse products | ✅ | ✅ | ✅ |
| Create/edit products | ❌ | ✅ (own) | ✅ |
| Submit RFQ | ✅ | ❌ | ✅ |
| Update order status | ✅ (limited) | ✅ (limited) | ✅ |
| Approve KYB | ❌ | ❌ | ✅ |
| View all users | ❌ | ❌ | ✅ |
| Access audit logs | ❌ | ❌ | ✅ |

- Horizontal authorization (object-level checks) ensures users can only modify their own resources.
- Admin actions are logged and require MFA session confirmation for destructive operations.

---

## 7. Audit Logging

- All security-relevant events are written to an **immutable audit log** (MongoDB `audit_logs` collection with write-once enforcement).
- Logged events include:

  | Event Type | Trigger |
  |---|---|
  | `LOGIN` | Successful authentication |
  | `LOGIN_FAILURE` | Failed authentication attempt |
  | `PASSWORD_RESET` | Password change initiated or completed |
  | `TOKEN_REVOKED` | Logout or forced token revocation |
  | `KYB_SUBMITTED` | Organization KYB documents submitted |
  | `KYB_APPROVED` / `KYB_REJECTED` | Admin KYB decision |
  | `ORDER_STATUS_CHANGE` | Any order status transition |
  | `ADMIN_ACTION` | Any admin-initiated operation |

- Logs include: timestamp, event type, actor ID, target ID, IP address, and user agent.
- Logs are exported to a SIEM (Security Information and Event Management) tool for alerting.

---

## 8. Dependency and Supply Chain Security

- All third-party dependencies are tracked and scanned with automated tools (e.g., GitHub Dependabot, `npm audit`, `pip-audit`).
- Critical security patches are applied within **48 hours** of disclosure.
- Docker base images are pinned to specific digest hashes and scanned with Trivy or Snyk.

---

## 9. Penetration Testing and Vulnerability Management

- An initial **penetration test** is conducted before Phase 1 launch by an independent security firm.
- Subsequent penetration tests are performed **annually** and after major feature releases.
- A **responsible disclosure policy** and bug bounty program are established at launch.
- Critical vulnerabilities (CVSS ≥ 9.0) require remediation within 24 hours; high (7.0–8.9) within 7 days.

---

## 10. Compliance Considerations

- User data handling follows applicable data protection regulations (e.g., Nigeria NDPR, Kenya DPA, GDPR where EU data subjects are involved).
- Payment processing complies with **PCI-DSS** requirements — card data is never stored on TradeLink servers; payment tokenization is handled by the payment gateway (Paystack / Flutterwave).
- Data retention policies define how long user data, order data, and logs are retained before secure deletion.
