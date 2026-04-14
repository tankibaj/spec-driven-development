# Product Concept: Multi-Tenant E-Commerce Platform

**Status:** Approved
**Author:** Product Team
**Date:** 2026-04-11

---

## Vision

A multi-tenant e-commerce platform that enables merchants to sell online with
minimal setup, while providing their customers with a fast, friction-free
shopping experience and operations teams with full control over orders and stock.

---

## Problem Statement

Small-to-mid-size merchants need an online storefront but lack the engineering
resources to build and operate one. Existing solutions are either too rigid
(template-based, no customisation) or too complex (enterprise platforms with
6-month onboarding). There is a gap for a platform that is:

- **Fast to set up** — a merchant can go live in days, not months.
- **Multi-tenant by default** — shared infrastructure, isolated data per tenant.
- **API-first** — every capability is an API; the storefront is one consumer,
  the admin portal is another, and merchants can build their own integrations.

---

## Target Market

| Segment | Description |
|---------|-------------|
| **Primary** | Small-to-mid-size merchants (1–50 employees) selling physical goods online |
| **Secondary** | Operations teams managing fulfilment, stock, and customer support |
| **Tertiary** | Platform engineers building and maintaining the infrastructure |

---

## Core Capabilities

These are the product pillars. Each pillar maps to one or more services.

| # | Pillar | Description | Services |
|---|--------|-------------|----------|
| 1 | **Storefront** | Customer-facing catalogue browsing, cart, and checkout | `storefront-app`, `order-service`, `inventory-service` |
| 2 | **Order Management** | Order lifecycle from placement through fulfilment, cancellation, and refunds | `order-service`, `admin-app` |
| 3 | **Inventory & Catalogue** | Product creation, SKU management, stock levels, reservations, deductions | `inventory-service`, `admin-app` |
| 4 | **Customer Accounts** | Registration, authentication, address book, order history | `order-service`, `storefront-app` |
| 5 | **Notifications** | Transactional emails and SMS across all lifecycle events | `notification-service` |
| 6 | **Operations & Observability** | Admin dashboards, real-time monitoring, health checks, metrics | `admin-app`, all backend services |

---

## Service Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Consumers                              │
│   ┌───────────────────┐          ┌───────────────────┐       │
│   │   storefront-app  │          │     admin-app     │       │
│   │   (React / TS)    │          │   (React / TS)    │       │
│   └────────┬──────────┘          └─────────┬─────────┘       │
│            │                               │                  │
├────────────┼───────────────────────────────┼──────────────────┤
│            │           API Layer            │                  │
├────────────┼───────────────────────────────┼──────────────────┤
│            │        Backend Services        │                  │
│  ┌─────────▼──────┐  ┌────────────────┐  ┌▼──────────────┐  │
│  │ order-service  │◄─┤inventory-      │  │notification-  │  │
│  │ (Python/       │  │service         │  │service        │  │
│  │  FastAPI)      │  │(Python/FastAPI)│  │(Python/FastAPI│  │
│  └────────────────┘  └────────────────┘  └───────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| Merchant onboarding time | < 1 day from signup to first product listed | Time between tenant creation and first `POST /products` |
| Checkout conversion rate | > 70% of carts result in orders | Orders placed / checkout sessions initiated |
| Order processing latency | < 2s P95 from "Place Order" to confirmation | P95 of `POST /checkout/guest/orders` response time |
| Platform uptime | 99.9% | Health endpoint monitoring across all services |
| Order confirmation delivery | > 99% within 2 minutes | `notification_sent_total` Prometheus metric |
| Guest-to-account conversion | > 15% within 30 days of guest order | Account registrations with email matching a prior guest order |

---

## Tenancy Model

See `plan/reference/roles.md` for the full role hierarchy.

- Each **Tenant** is an independent merchant with isolated data.
- Data isolation is enforced at every layer: DB row-level, API auth, service boundaries.
- Platform roles (`platform_ops`, `platform_admin`) operate across tenants for support and infrastructure only.

---

## Feature Roadmap

| Priority | Feature | Story ID | Status |
|----------|---------|----------|--------|
| 1 | **MVP Guest Shopping Experience** | **story-0001** | **PRD draft — entering Phase 1** |
| 2 | Customer Registration & Accounts | TBD | Not started |
| 3 | Order Management Dashboard (full) | TBD | Not started |

> **Note:** story-0001 (MVP) covers the full guest shopping flow end-to-end:
> product catalog (browse + stock), cart, guest checkout, order confirmation
> email, and a read-only admin order view. It exercises all 5 workspaces and
> is the first end-to-end runnable product.

---

## Technical Standards

| Area | Standard | Reference |
|------|----------|-----------|
| API Style | REST, OpenAPI 3.1 | `contracts/api/` |
| Backend | Python, FastAPI, SQLAlchemy async, Alembic, Ruff, mypy | `contracts/architecture/workspace-bootstrap.md` |
| Frontend | TypeScript, React 19, Vite, Zustand, Biome, Vitest, MSW | `contracts/architecture/workspace-bootstrap.md` |
| Observability | `/health`, `/ready`, `/metrics`, structured JSON logging | `contracts/architecture/observability-standards.md` |
| Contract Testing | Schemathesis (BE), Pact (inter-service), MSW (FE) | `contracts/architecture/contract-validation.md` |
| Branching | `spec/` for spec-hub, `feat/{story-ID}-{WP-ID}` for workspaces | `contracts/architecture/branching-strategy.md` |
| Methodology | Spec Driven Development (SDD), maturity level 2 (Spec-First) | `registry/project.yaml` |
