# TradeLink Global

**Digital Trade Infrastructure for African Businesses**

TradeLink Global is a B2B marketplace platform designed to facilitate secure and transparent cross-border and local trade across Africa. It connects verified buyers and suppliers through a trusted digital infrastructure that supports the full trade lifecycle — from product discovery and negotiation to order fulfillment.

---

## Key Features

- **Marketplace** — Searchable product feed with categories, pricing, and Minimum Order Quantity (MOQ).
- **Secure Messaging** — Real-time negotiation chat between buyers and suppliers.
- **Order Management** — End-to-end order lifecycle from RFQ (Request for Quotation) to completion.
- **Trust System** — KYB (Know Your Business) verification badges and user ratings.
- **Multi-role Access** — Dedicated portals for Buyers, Suppliers, and Admins.

---

## Documentation

| Document | Description |
|---|---|
| [Software Design Document](docs/SDD.md) | Full system architecture and technical specifications |
| [Database Schema](docs/DATABASE_SCHEMA.md) | Relational and document store schemas |
| [API Overview](docs/API.md) | REST API design and endpoint reference |
| [Security Design](docs/SECURITY.md) | Authentication, encryption, and threat mitigation |
| [Roadmap](docs/ROADMAP.md) | Phased development and feature rollout plan |
| [Deployment Guide](docs/DEPLOYMENT.md) | Infrastructure setup, CI/CD, and operations runbook |
| [Contributing Guide](docs/CONTRIBUTING.md) | Developer setup, code standards, and PR process |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Web Frontend | React |
| Mobile | React Native / Flutter |
| Backend | Node.js (Express) or Django |
| Primary Database | PostgreSQL |
| Chat / Logs | MongoDB |
| Hosting | AWS or Google Cloud |
| Containerization | Docker |
| CI/CD | GitHub Actions / Cloud Pipelines |

---

## User Roles

| Role | Description |
|---|---|
| **Buyer** | Businesses seeking to procure products |
| **Supplier** | Verified businesses listing products and managing orders |
| **Admin** | System operators managing verification and platform health |

---

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) 24.x or later
- [Docker Compose](https://docs.docker.com/compose/) v2.x
- [Node.js](https://nodejs.org/) 20 LTS (for local development without Docker)
- [Git](https://git-scm.com/)

### Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/addexdi/Tradelink.git
cd Tradelink

# 2. Set up environment variables
cp .env.example .env
# Edit .env and fill in required values

# 3. Start all services
docker compose up --build

# 4. Apply database migrations
docker compose exec api npm run migrate

# 5. (Optional) Seed development data
docker compose exec api npm run seed
```

The API will be available at `http://localhost:3000/api/v1`.

For full developer setup instructions, code standards, and the PR process, see the [Contributing Guide](docs/CONTRIBUTING.md).  
For infrastructure setup and CI/CD configuration, see the [Deployment Guide](docs/DEPLOYMENT.md).

---

## License

This project and its documentation are proprietary to TradeLink Global. All rights reserved.
