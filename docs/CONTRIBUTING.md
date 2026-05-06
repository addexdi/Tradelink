# Contributing Guide

**Project:** TradeLink Global  

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Local Development Setup](#2-local-development-setup)
3. [Project Structure](#3-project-structure)
4. [Environment Variables](#4-environment-variables)
5. [Running the Application](#5-running-the-application)
6. [Running Tests](#6-running-tests)
7. [Code Standards](#7-code-standards)
8. [Git Workflow](#8-git-workflow)
9. [Pull Request Process](#9-pull-request-process)
10. [Reporting Bugs](#10-reporting-bugs)

---

## 1. Prerequisites

Ensure the following tools are installed before setting up the project:

| Tool | Minimum Version | Purpose |
|---|---|---|
| Node.js | 20 LTS | Backend runtime (Express variant) |
| Python | 3.11 | Backend runtime (Django variant) |
| Docker | 24.x | Containerization |
| Docker Compose | 2.x | Local multi-service orchestration |
| PostgreSQL client | 15 | Database inspection (`psql`) |
| Git | 2.40 | Version control |

---

## 2. Local Development Setup

### 2.1 Clone the Repository

```bash
git clone https://github.com/addexdi/Tradelink.git
cd Tradelink
```

### 2.2 Copy Environment Files

```bash
cp .env.example .env
```

Edit `.env` and fill in all required values (see [Environment Variables](#4-environment-variables)).

### 2.3 Start All Services with Docker Compose

```bash
docker compose up --build
```

This starts:
- **API server** on `http://localhost:3000`
- **PostgreSQL** on `localhost:5432`
- **MongoDB** on `localhost:27017`
- **Redis** on `localhost:6379`

### 2.4 Apply Database Migrations

```bash
# Node.js / Express variant
docker compose exec api npm run migrate

# Django variant
docker compose exec api python manage.py migrate
```

### 2.5 Seed Development Data (optional)

```bash
# Node.js
docker compose exec api npm run seed

# Django
docker compose exec api python manage.py loaddata fixtures/dev_seed.json
```

---

## 3. Project Structure

```
Tradelink/
├── docs/               # Project documentation
├── src/
│   ├── api/            # Route handlers / controllers
│   ├── services/       # Business logic layer
│   ├── models/         # Database models / ORM definitions
│   ├── middleware/      # Auth, validation, rate limiting
│   ├── workers/        # Background job processors
│   └── utils/          # Shared utilities
├── tests/
│   ├── unit/           # Unit tests
│   ├── integration/    # Integration tests
│   └── e2e/            # End-to-end tests
├── migrations/         # Database migration files
├── docker-compose.yml
├── Dockerfile
└── .env.example
```

---

## 4. Environment Variables

All environment variables are documented in `.env.example`. Key variables:

| Variable | Description | Example |
|---|---|---|
| `NODE_ENV` | Runtime environment | `development` |
| `PORT` | API server port | `3000` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/tradelink` |
| `MONGODB_URI` | MongoDB connection string | `mongodb://localhost:27017/tradelink` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` |
| `JWT_SECRET` | Secret for signing access tokens | (generate with `openssl rand -hex 64`) |
| `JWT_REFRESH_SECRET` | Secret for signing refresh tokens | (generate with `openssl rand -hex 64`) |
| `AWS_S3_BUCKET` | S3 bucket for file uploads | `tradelink-uploads-dev` |
| `AWS_REGION` | AWS region | `eu-west-1` |
| `PAYSTACK_SECRET_KEY` | Paystack payment gateway secret | `sk_test_...` |
| `SMTP_HOST` | Email server host | `smtp.mailhog.io` |
| `SMTP_PORT` | Email server port | `1025` |

> **Never commit secrets to source control.** Use `.env` for local development only. Production secrets are managed via AWS Secrets Manager or GCP Secret Manager (see [DEPLOYMENT.md](DEPLOYMENT.md)).

---

## 5. Running the Application

### Development Mode (with hot reload)

```bash
# Node.js / Express
npm run dev

# Django
python manage.py runserver
```

### Production Build

```bash
docker compose -f docker-compose.prod.yml up --build
```

---

## 6. Running Tests

```bash
# Run all tests
npm test

# Run unit tests only
npm run test:unit

# Run integration tests only
npm run test:integration

# Run with coverage report
npm run test:coverage
```

All tests must pass before a pull request will be merged. Maintain **≥ 80% code coverage** for all new business logic.

---

## 7. Code Standards

### 7.1 General

- Write **self-documenting code** with clear variable and function names.
- Keep functions small and single-purpose (≤ 30 lines as a guide).
- Prefer `async/await` over callbacks or raw `.then()` chains.
- Handle all errors explicitly — never swallow exceptions silently.

### 7.2 Linting and Formatting

The project uses ESLint (Node.js) or flake8 + Black (Django) for linting and formatting.

```bash
# Node.js
npm run lint
npm run lint:fix

# Django
flake8 .
black .
```

All code must pass linting checks before merging. The CI pipeline enforces this automatically.

### 7.3 API Layer

- Validate all request inputs at the route level before passing to services.
- Return consistent error envelopes (see [API.md](API.md) — Error Handling).
- Never expose internal stack traces to API consumers.
- Log all errors with sufficient context for debugging.

### 7.4 Database

- Use **parameterized queries** exclusively — never interpolate user input into SQL.
- All schema changes must be done via migration files, not manual DDL.
- New tables require both `created_at` and `updated_at` columns.

### 7.5 Security

- Never log credentials, tokens, or PII.
- Sanitize all user-supplied data before use.
- Follow the principle of least privilege for all new database roles and IAM permissions.
- Review the full [Security Design](SECURITY.md) before implementing auth-related features.

---

## 8. Git Workflow

TradeLink Global follows **GitHub Flow**:

1. Branch from `main` using a descriptive name:
   ```
   feature/kyb-verification-workflow
   fix/order-status-race-condition
   docs/api-endpoint-reference
   ```

2. Make small, focused commits with clear messages:
   ```
   feat: add KYB document upload endpoint
   fix: prevent duplicate order submission on double-click
   docs: add payment gateway integration guide
   ```

3. Keep branches short-lived — merge within a few days of opening.

4. Rebase on `main` before opening a pull request:
   ```bash
   git fetch origin
   git rebase origin/main
   ```

### Commit Message Format

Follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

| Type | When to use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code restructuring without behavior change |
| `test` | Adding or updating tests |
| `chore` | Build scripts, CI, dependencies |
| `security` | Security fix or hardening |

---

## 9. Pull Request Process

1. **Self-review** your diff before opening a PR. Check for debug code, hardcoded values, and missing tests.
2. Fill in the PR template completely.
3. Ensure all **CI checks pass** (linting, tests, security scan).
4. Request **at least one reviewer** from the core team.
5. Address all reviewer comments before merging.
6. **Squash-merge** into `main` with a clean commit message.
7. Delete the source branch after merging.

### PR Checklist

- [ ] Code follows project style and conventions
- [ ] All new logic has corresponding unit tests
- [ ] Integration tests updated if API contracts changed
- [ ] No secrets or credentials included
- [ ] Documentation updated if behavior changed
- [ ] Changelog entry added (if user-facing change)

---

## 10. Reporting Bugs

1. Search existing GitHub Issues to avoid duplicates.
2. Open a new issue using the **Bug Report** template.
3. Include:
   - Steps to reproduce
   - Expected vs. actual behavior
   - Environment details (OS, Node/Python version, browser)
   - Relevant log output or screenshots
4. For **security vulnerabilities**, do **not** open a public issue. Follow the responsible disclosure policy described in [SECURITY.md](SECURITY.md).
