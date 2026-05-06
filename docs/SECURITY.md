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

---

## 11. Security Headers

Every HTTP response from the API includes the following headers:

| Header | Value | Purpose |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Force HTTPS for one year |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME-type sniffing |
| `X-Frame-Options` | `DENY` | Block clickjacking via iframes |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer information leakage |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Disable unused browser APIs |
| `Content-Security-Policy` | (see below) | Prevent XSS and data injection |
| `X-Request-ID` | `<uuid>` | Request correlation for log tracing |

### Content Security Policy

```
default-src 'self';
script-src 'self' https://js.paystack.co https://checkout.flutterwave.com;
style-src 'self' 'unsafe-inline';
img-src 'self' https://cdn.tradelinkglobal.com data:;
connect-src 'self' wss://api.tradelinkglobal.com;
frame-ancestors 'none';
base-uri 'self';
form-action 'self';
```

---

## 12. CORS Policy

Cross-Origin Resource Sharing (CORS) is enforced at the API layer:

- **Allowed origins** — Configured via the `CORS_ORIGINS` environment variable (comma-separated list). Only known frontend origins are permitted.
- **Allowed methods** — `GET, POST, PATCH, DELETE, OPTIONS`
- **Allowed headers** — `Authorization, Content-Type, X-Request-ID`
- **Credentials** — `true` (required for `HttpOnly` refresh-token cookies)
- **Preflight cache** — `Access-Control-Max-Age: 600` (10 minutes)
- Wildcard `*` origins are **never** permitted in production.

---

## 13. Cookie Security

The refresh token is stored in a server-set cookie with the following attributes:

| Attribute | Value | Reason |
|---|---|---|
| `HttpOnly` | true | Prevents JavaScript access (blocks XSS token theft) |
| `Secure` | true | Cookie only sent over HTTPS |
| `SameSite` | `Strict` | Blocks CSRF attacks |
| `Path` | `/api/v1/auth` | Limits scope to authentication endpoints |
| `Max-Age` | 2592000 (30 days) | Matches refresh token lifetime |
| `Domain` | `.tradelinkglobal.com` | Shared across app subdomains |

---

## 14. Secrets Management

All application secrets (JWT keys, database credentials, payment gateway keys) are:

- Stored in **AWS Secrets Manager** or **GCP Secret Manager**.
- Referenced in container definitions by ARN — never embedded in code or Docker images.
- Rotated on a scheduled cadence (see [DEPLOYMENT.md](DEPLOYMENT.md) — Secrets Management).
- Accessed at runtime via the cloud SDK, not environment files, in production.

The application fails to start if any required secret is missing, preventing partial configuration deployments.

---

## 15. Incident Response

### Severity Levels

| Level | Definition | Response Time |
|---|---|---|
| **P0 — Critical** | Platform down, data breach, or payment system failure | 15 minutes |
| **P1 — High** | Core feature unavailable for >10% of users | 1 hour |
| **P2 — Medium** | Degraded performance or non-critical feature failure | 4 hours |
| **P3 — Low** | Minor bugs, cosmetic issues, non-urgent alerts | Next business day |

### Incident Response Steps

1. **Detect** — Alert triggered via CloudWatch, Sentry, or user report.
2. **Acknowledge** — On-call engineer acknowledges within SLA window.
3. **Assess** — Determine severity, scope, and blast radius.
4. **Contain** — Isolate affected components; enable maintenance mode if required.
5. **Remediate** — Deploy fix or roll back to last known good state.
6. **Communicate** — Notify affected users and stakeholders via status page.
7. **Post-mortem** — Blameless post-mortem written within 48 hours of resolution.

For security incidents (suspected breach, credential compromise):
- Immediately revoke all active refresh tokens.
- Rotate the compromised secret.
- Notify the Data Protection Officer (DPO) within 24 hours.
- Notify affected users within 72 hours as required by NDPR/GDPR.

---

## 16. Data Retention Policy

| Data Category | Retention Period | Deletion Method |
|---|---|---|
| Active user accounts | Indefinite while account is active | Soft-delete on deactivation |
| Closed/deleted accounts | 90 days after closure | Hard-delete personal data; anonymize transaction records |
| Order records | 7 years | Anonymize personal fields after retention period |
| Payment records | 7 years | Retained for financial audit compliance |
| Chat messages | 2 years | Purged after retention period |
| Audit logs | 7 years | Write-once; immutable until expiry |
| Application logs | 90 days (API/worker) / 365 days (access) | Auto-deleted via CloudWatch retention policy |
| KYB documents | Duration of account + 5 years | Secure deletion from S3 |
