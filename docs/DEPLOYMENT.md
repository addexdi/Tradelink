# Deployment Guide

**Project:** TradeLink Global  
**Version:** 1.0  

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Docker Setup](#2-docker-setup)
3. [Environment Configuration](#3-environment-configuration)
4. [Database Setup and Migrations](#4-database-setup-and-migrations)
5. [CI/CD Pipeline](#5-cicd-pipeline)
6. [Cloud Infrastructure (AWS)](#6-cloud-infrastructure-aws)
7. [Secrets Management](#7-secrets-management)
8. [Monitoring and Alerting](#8-monitoring-and-alerting)
9. [Scaling](#9-scaling)
10. [Rollback Procedure](#10-rollback-procedure)
11. [Maintenance and Backups](#11-maintenance-and-backups)

---

## 1. Infrastructure Overview

TradeLink Global runs on a fully containerized, cloud-hosted infrastructure. All components are deployed into a **Virtual Private Cloud (VPC)** with public-facing traffic terminating at the load balancer or CDN edge.

```
                        ┌─────────────┐
                        │  Internet   │
                        └──────┬──────┘
                               │
              ┌────────────────▼───────────────────┐
              │        CDN (CloudFront / GCP CDN)   │
              │   Static assets, DDoS protection    │
              └────────────────┬───────────────────┘
                               │
              ┌────────────────▼───────────────────┐
              │    Application Load Balancer (ALB)  │
              │    TLS termination · Health checks  │
              └──────────┬─────────────┬───────────┘
                         │             │
              ┌──────────▼──┐   ┌──────▼──────────┐
              │  API Server │   │  WebSocket Server│
              │  (ECS/K8s)  │   │    (ECS/K8s)    │
              └──────┬──────┘   └──────┬──────────┘
                     │                 │
      ┌──────────────┼─────────────────┤
      │              │                 │
┌─────▼──────┐ ┌─────▼──────┐ ┌───────▼──────┐
│ PostgreSQL │ │  MongoDB   │ │    Redis     │
│  (RDS)    │ │ (Atlas/EC2)│ │(ElastiCache) │
└────────────┘ └────────────┘ └──────────────┘
                     │
         ┌───────────▼───────────┐
         │    Object Storage     │
         │  (S3 / GCS Bucket)    │
         └───────────────────────┘
```

### Environments

| Environment | Branch | Purpose |
|---|---|---|
| `development` | `feature/*` | Local developer machines |
| `staging` | `main` | QA, integration testing, stakeholder preview |
| `production` | Tagged releases | Live user traffic |

---

## 2. Docker Setup

### Dockerfile (API)

The API image is built from an official slim base image, copies only production dependencies, and runs as a non-root user.

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-slim
WORKDIR /app
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
COPY --from=builder /app/node_modules ./node_modules
COPY . .
USER appuser
EXPOSE 3000
CMD ["node", "src/server.js"]
```

### docker-compose.yml (Local Development)

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    env_file:
      - .env
    depends_on:
      - postgres
      - mongo
      - redis
    volumes:
      - ./src:/app/src   # hot reload in dev

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: tradelink
      POSTGRES_USER: tradelink
      POSTGRES_PASSWORD: tradelink_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
  mongo_data:
```

### Build and Push Image to Registry

```bash
# Build
docker build -t tradelink-api:$(git rev-parse --short HEAD) .

# Tag and push to ECR
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin <account_id>.dkr.ecr.eu-west-1.amazonaws.com

docker tag tradelink-api:<sha> <account_id>.dkr.ecr.eu-west-1.amazonaws.com/tradelink-api:<sha>
docker push <account_id>.dkr.ecr.eu-west-1.amazonaws.com/tradelink-api:<sha>
```

---

## 3. Environment Configuration

Environment variables are never committed to source control. They are injected at runtime from:

- **Local development** — `.env` file (copied from `.env.example`)
- **Staging / Production** — AWS Secrets Manager or Parameter Store, injected into ECS task definitions

### Full Variable Reference

| Variable | Required | Description |
|---|---|---|
| `NODE_ENV` | ✅ | `development`, `staging`, or `production` |
| `PORT` | ✅ | API server port (default: `3000`) |
| `DATABASE_URL` | ✅ | PostgreSQL connection string |
| `MONGODB_URI` | ✅ | MongoDB connection string |
| `REDIS_URL` | ✅ | Redis connection string |
| `JWT_SECRET` | ✅ | HS256 secret for access tokens (≥ 64 bytes) |
| `JWT_REFRESH_SECRET` | ✅ | HS256 secret for refresh tokens (≥ 64 bytes) |
| `JWT_ACCESS_EXPIRY` | | Access token TTL (default: `15m`) |
| `JWT_REFRESH_EXPIRY` | | Refresh token TTL (default: `30d`) |
| `AWS_REGION` | ✅ | AWS region for S3 and KMS |
| `AWS_S3_BUCKET` | ✅ | S3 bucket name for file uploads |
| `AWS_KMS_KEY_ID` | ✅ | KMS key ARN for field-level encryption |
| `PAYSTACK_SECRET_KEY` | ✅ | Paystack live/test secret key |
| `FLUTTERWAVE_SECRET_KEY` | | Flutterwave secret key (optional gateway) |
| `SMTP_HOST` | ✅ | SMTP server hostname |
| `SMTP_PORT` | ✅ | SMTP server port |
| `SMTP_USER` | ✅ | SMTP username |
| `SMTP_PASS` | ✅ | SMTP password |
| `EMAIL_FROM` | ✅ | Sender address for transactional emails |
| `CORS_ORIGINS` | ✅ | Comma-separated list of allowed origins |
| `LOG_LEVEL` | | Logging verbosity (default: `info`) |
| `SENTRY_DSN` | | Sentry error tracking DSN |

---

## 4. Database Setup and Migrations

### PostgreSQL

Migrations are managed with a dedicated migration tool (e.g., `node-pg-migrate` or Django's built-in ORM migrations).

```bash
# Create a new migration
npm run migrate:create -- --name add_disputes_table

# Run all pending migrations
npm run migrate:up

# Roll back the last migration
npm run migrate:down
```

**Production migration checklist:**
1. Test migration on staging database first.
2. Take a manual snapshot of the production RDS instance before running.
3. Run migrations with the API in read-only mode if the migration involves existing data.
4. Verify row counts and spot-check data after migration.

### MongoDB

MongoDB schema changes are managed through application-level versioned data migrations stored in `migrations/mongo/`.

```bash
npm run migrate:mongo
```

---

## 5. CI/CD Pipeline

The CI/CD pipeline runs on **GitHub Actions** and consists of three stages:

### Stage 1 — Continuous Integration (on every push)

```
Checkout → Install dependencies → Lint → Run unit tests → Run integration tests → Build Docker image → Security scan (Trivy)
```

### Stage 2 — Staging Deployment (on merge to `main`)

```
Pull CI image → Push to ECR → Update ECS task definition → Deploy to staging → Run smoke tests → Notify team
```

### Stage 3 — Production Deployment (on version tag `v*`)

```
Pull release image → Tag as production → Deploy to production (blue/green) → Health check → Swap traffic → Notify team
```

### Sample GitHub Actions Workflow

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
  release:
    types: [published]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run lint
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t tradelink-api:${{ github.sha }} .
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: tradelink-api:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1

  deploy-staging:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS staging
        run: |
          aws ecs update-service \
            --cluster tradelink-staging \
            --service api \
            --force-new-deployment
```

---

## 6. Cloud Infrastructure (AWS)

### Key Services Used

| Service | Purpose |
|---|---|
| **ECS Fargate** | Serverless container hosting for API and worker services |
| **RDS PostgreSQL** | Managed PostgreSQL with Multi-AZ failover |
| **ElastiCache (Redis)** | Session cache and rate limiting store |
| **S3** | Object storage for product images and KYB documents |
| **CloudFront** | CDN for static assets and API acceleration |
| **Route 53** | DNS management and health-based routing |
| **ACM** | Managed TLS certificate provisioning and renewal |
| **KMS** | Encryption key management for sensitive data fields |
| **Secrets Manager** | Runtime secret injection for ECS tasks |
| **SQS** | Message queue for async job processing |
| **SES** | Transactional email delivery |
| **CloudWatch** | Logs, metrics, and alarms |
| **Shield Standard** | Always-on DDoS protection |

### VPC Design

- **Public subnets** — Load balancers and NAT gateways only.
- **Private subnets** — All application containers and databases.
- **Database subnet group** — Isolated subnet with no internet route.
- Security groups enforce least-privilege port access between components.

---

## 7. Secrets Management

All secrets are stored in **AWS Secrets Manager** and referenced by ARN in ECS task definitions. They are never embedded in Docker images or environment files in the repository.

```bash
# Store a secret
aws secretsmanager create-secret \
  --name tradelink/production/jwt-secret \
  --secret-string "$(openssl rand -hex 64)"

# Rotate a secret
aws secretsmanager rotate-secret \
  --secret-id tradelink/production/jwt-secret
```

Secret rotation is configured for:
- Database credentials — every 30 days
- JWT secrets — every 90 days
- Payment gateway keys — on-demand after compromise

---

## 8. Monitoring and Alerting

### Metrics

Key metrics are tracked via **CloudWatch** and visualized in a Grafana dashboard:

| Metric | Alert Threshold |
|---|---|
| API error rate (5xx) | > 1% over 5 minutes |
| API p95 response time | > 500 ms |
| CPU utilization (ECS) | > 80% sustained |
| Memory utilization (ECS) | > 85% sustained |
| RDS free storage | < 20% |
| Redis memory usage | > 75% |
| Failed login rate | > 50/minute (abuse signal) |

### Logging

All application logs are structured JSON and forwarded to **CloudWatch Logs**. Log groups:

| Group | Retention |
|---|---|
| `/tradelink/api` | 90 days |
| `/tradelink/worker` | 90 days |
| `/tradelink/access` | 365 days |
| `/tradelink/audit` | 7 years |

### Error Tracking

Runtime errors are captured and grouped in **Sentry** with release tracking tied to the Git SHA.

---

## 9. Scaling

### Horizontal Scaling (ECS)

API services scale automatically based on CPU and request count:

```
Minimum tasks:  2  (always available across 2 AZs)
Maximum tasks:  20
Scale-out:      CPU > 70% for 2 minutes → add 2 tasks
Scale-in:       CPU < 30% for 5 minutes → remove 1 task
```

### Database Scaling

- **Read replicas** — One replica in each active region for read-heavy traffic (product search, listing browse).
- **Connection pooling** — PgBouncer in transaction pooling mode sits between the API and RDS to cap open connections.

### Caching Strategy

| Data | Cache TTL | Invalidation |
|---|---|---|
| Product listing (single) | 5 minutes | On update or deactivation |
| Product search results | 60 seconds | On any new listing in category |
| User profile | 10 minutes | On profile update |
| Supplier rating summary | 15 minutes | On new rating submission |

---

## 10. Rollback Procedure

### Application Rollback

ECS maintains the previous task definition revision. To roll back:

```bash
# List recent task definition revisions
aws ecs list-task-definitions \
  --family-prefix tradelink-api \
  --sort DESC

# Update service to previous revision
aws ecs update-service \
  --cluster tradelink-production \
  --service api \
  --task-definition tradelink-api:<previous_revision>
```

### Database Rollback

If a migration causes data corruption:

1. Stop API traffic by setting the service task count to `0`.
2. Restore the pre-migration RDS snapshot:
   ```bash
   aws rds restore-db-instance-to-point-in-time \
     --source-db-instance-identifier tradelink-prod \
     --target-db-instance-identifier tradelink-prod-restored \
     --restore-time <timestamp_before_migration>
   ```
3. Update `DATABASE_URL` in Secrets Manager to point to the restored instance.
4. Resume API traffic.

---

## 11. Maintenance and Backups

### Automated Backups

| Data Store | Backup Mechanism | Retention | RTO | RPO |
|---|---|---|---|---|
| PostgreSQL (RDS) | Automated daily snapshots + continuous WAL | 30 days | < 1 hour | < 5 minutes |
| MongoDB | Daily snapshots to S3 | 30 days | < 2 hours | < 24 hours |
| S3 (uploads) | Cross-region replication | Indefinite | Immediate | 0 |

### Maintenance Windows

- **RDS** — Tuesdays 02:00–04:00 UTC (lowest traffic period for Africa/Europe time zones).
- **ECS / OS patches** — Rolling updates with no-downtime deployment on Wednesdays.

### Health Checks

The `/health` endpoint returns the status of all critical dependencies:

```json
GET /api/v1/health

{
  "status": "ok",
  "timestamp": "2026-05-06T18:00:00Z",
  "services": {
    "postgres": "ok",
    "mongo": "ok",
    "redis": "ok"
  },
  "version": "1.4.2",
  "sha": "a3f9c1d"
}
```

A `"degraded"` status means at least one non-critical dependency is unavailable. A `"critical"` status means the API cannot serve requests reliably and triggers an immediate on-call alert.
