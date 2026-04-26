# WP-001-BE: Catalog + Stock API — Backend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/inventory-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the full inventory-service API: product catalogue (list, detail, create), stock management (query, reserve, deduct, release), and observability endpoints (`/health`, `/ready`, `/metrics`).

This is the foundational data service — storefront-app reads catalogue data from `GET /products` and `GET /products/{productId}`, and order-service calls stock operations (`POST /stock/reserve`, `POST /stock/deduct`, `POST /stock/reservations/{id}/release`) during the checkout saga. `POST /products` is used for seeding test/demo data (no AC, but required for test setup).

All external service calls: none — inventory-service has no outbound dependencies on other services.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-001 — Product catalogue page

The guest sees a paginated list of products from the inventory-service catalogue. Each product card shows the product name, price, and thumbnail image. The guest can filter the list to show only products with available stock.

**Testable:** `GET /products` returns `200` with a `ProductPage` containing `data` (product array) and `meta` (pagination). `GET /products?in_stock_only=true` returns only products where at least one SKU has `stock_level > 0`. UI renders a product grid with name, price, and image for each product.

---

### AC-002 — Product detail page

The guest views an individual product page showing the product name, description, all available variants (SKUs) with their prices, and stock availability per variant.

**Testable:** `GET /products/{productId}` returns `200` with a `Product` containing `name`, `description`, and `skus` array where each SKU has `label`, `price_minor`, and `stock_level`. UI displays all fields and indicates which variants are in stock.

---

### AC-003 — Empty catalogue

When the catalogue contains no products, the catalogue page shows an informative empty state.

**Testable:** When `GET /products` returns `200` with an empty `data` array, then the UI displays "No products available" instead of an empty grid.

---

### AC-004 — Product not found

When the guest navigates to a URL for a product that does not exist, a not-found page is displayed.

**Testable:** `GET /products/{nonExistentId}` returns `404`. UI displays a "Product not found" message with a link back to the catalogue.

---

## Test Scenarios (from TS-001)

### TS-001-001 — Paginated product list returns products with metadata
- **Preconditions:** inventory-service running; DB has 25 seeded products for test tenant, each with ≥1 SKU with `stock_level > 0`
- **Action:** `GET /products?page=1&per_page=20` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` has 20 `Product` objects; `meta.total` = 25, `meta.page` = 1, `meta.per_page` = 20; each product has `id`, `name`, `skus` (non-empty), `created_at`; each SKU has `id`, `label`, `price_minor`, `stock_level`

### TS-001-002 — In-stock filter returns only products with available stock
- **Preconditions:** DB has 5 products: 3 with ≥1 SKU `stock_level > 0`, 2 with all SKUs `stock_level = 0`
- **Action:** `GET /products?in_stock_only=true` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` has exactly 3 products; every product in `data` has ≥1 SKU where `stock_level > 0`

### TS-001-003 — Pagination second page returns remaining products
- **Preconditions:** DB has 25 products for test tenant
- **Action:** `GET /products?page=2&per_page=20` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` has 5 products; `meta.page` = 2, `meta.total` = 25

### TS-001-004 — Product detail returns product with all SKU variants
- **Preconditions:** DB has product `{productId}` named "Classic T-Shirt", description "A comfortable cotton tee", 3 SKUs (Small, Medium, Large) with varying `price_minor` and `stock_level`
- **Action:** `GET /products/{productId}` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `name` = "Classic T-Shirt", `description` = "A comfortable cotton tee"; `skus` has 3 items each with `id`, `label`, `price_minor`, `stock_level`; SKU with `stock_level = 0` is present (not filtered)

### TS-001-005 — Product detail shows variant stock levels accurately
- **Preconditions:** Product exists with 2 SKUs: SKU-A `stock_level = 10`, SKU-B `stock_level = 0`
- **Action:** `GET /products/{productId}` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; SKU-A has `stock_level = 10`; SKU-B has `stock_level = 0`

### TS-001-006 — Empty catalogue returns empty data array
- **Preconditions:** DB has zero products for test tenant
- **Action:** `GET /products` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` = `[]`; `meta.total` = 0

### TS-001-007 — Non-existent product returns 404
- **Preconditions:** No product exists with `id = {nonExistentId}` for test tenant
- **Action:** `GET /products/{nonExistentId}` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 404; response matches `ErrorResponse` with descriptive `message`

### TS-001-056 — inventory-service GET /health returns 200
- **Preconditions:** inventory-service running
- **Action:** `GET /health`
- **Expected:** HTTP 200; body `{"status": "ok"}`

### TS-001-057 — inventory-service GET /ready returns 200
- **Preconditions:** inventory-service running; DB connection healthy
- **Action:** `GET /ready`
- **Expected:** HTTP 200; body contains `"status": "ready"`

### TS-001-058 — inventory-service GET /metrics returns Prometheus format
- **Preconditions:** inventory-service running
- **Action:** `GET /metrics`
- **Expected:** HTTP 200; `Content-Type` contains `text/plain`; body contains Prometheus-formatted metrics

---

## Relevant Contracts

### inventory-service.openapi.yaml (full excerpt)

**Base URL:** `https://example.com/inventory-service/api/v1/`

#### Parameters (shared)

```yaml
X-Tenant-ID:
  in: header
  required: true
  schema:
    type: string
    format: uuid

Idempotency-Key:
  in: header
  required: false
  schema:
    type: string
    maxLength: 64

productId:
  in: path
  required: true
  schema:
    type: string
    format: uuid
```

#### `GET /products` — List products

```
Parameters:
  - X-Tenant-ID (header, required, UUID)
  - in_stock_only (query, boolean, default: false)
  - page (query, integer, default: 1)
  - per_page (query, integer, default: 20, max: 100)

Response 200 → ProductPage
```

#### `POST /products` — Create a product

```
Parameters:
  - X-Tenant-ID (header, required, UUID)

Request body → CreateProductRequest (required)

Response 201 → Product
Response 400 → ErrorResponse
```

#### `GET /products/{productId}` — Get product by ID

```
Parameters:
  - X-Tenant-ID (header, required, UUID)
  - productId (path, required, UUID)

Response 200 → Product
Response 404 → ErrorResponse
```

#### `GET /stock/{skuId}` — Get stock level for a SKU

```
Parameters:
  - X-Tenant-ID (header, required, UUID)
  - skuId (path, required, UUID)

Response 200 → StockLevel
Response 404 → ErrorResponse
```

#### `POST /stock/reserve` — Reserve stock for SKUs

```
Description: Temporary hold on requested quantities. Called by order-service
during checkout. Reservations expire after 15 minutes if not converted.

Parameters:
  - X-Tenant-ID (header, required, UUID)
  - Idempotency-Key (header, optional, string max 64)

Request body → ReserveStockRequest (required)

Response 200 → ReserveStockResponse
Response 409 → StockConflictError
```

#### `POST /stock/deduct` — Convert reservations to permanent deductions

```
Parameters:
  - X-Tenant-ID (header, required, UUID)
  - Idempotency-Key (header, optional, string max 64)

Request body → DeductStockRequest (required)

Response 200 → (empty — stock deducted)
Response 409 → Reservation not found or expired
```

#### `POST /stock/reservations/{reservationId}/release` — Release reservation

```
Description: Saga compensation. Called by order-service when payment fails.
Decrements reserved counter, deletes reservation record.

Parameters:
  - X-Tenant-ID (header, required, UUID)
  - reservationId (path, required, UUID)

Response 204 → Released successfully
Response 404 → Not found (treat as success — already released or never existed)
```

### Schemas

#### Product

```yaml
type: object
required: [id, name, skus, created_at]
properties:
  id:          { type: string, format: uuid }
  name:        { type: string }
  description: { type: string }
  image_url:   { type: string, format: uri, maxLength: 500, nullable: true }
  skus:        { type: array, items: SKU }
  created_at:  { type: string, format: date-time }
```

#### SKU

```yaml
type: object
required: [id, label, price_minor, stock_level]
properties:
  id:          { type: string, format: uuid }
  label:       { type: string }
  price_minor: { type: integer }   # Price in minor currency units (cents)
  stock_level: { type: integer, minimum: 0 }
```

#### CreateProductRequest

```yaml
type: object
required: [name, skus]
properties:
  name:        { type: string, maxLength: 200 }
  description: { type: string, maxLength: 2000 }
  image_url:   { type: string, format: uri, maxLength: 500, nullable: true }
  skus:
    type: array
    minItems: 1
    items:
      type: object
      required: [label, price_minor, initial_stock]
      properties:
        label:         { type: string }
        price_minor:   { type: integer, minimum: 0 }
        initial_stock: { type: integer, minimum: 0 }
```

#### ProductPage

```yaml
type: object
properties:
  data: { type: array, items: Product }
  meta:
    type: object
    properties:
      total:    { type: integer }
      page:     { type: integer }
      per_page: { type: integer }
```

#### StockLevel

```yaml
type: object
required: [sku_id, available, reserved]
properties:
  sku_id:    { type: string, format: uuid }
  available: { type: integer }   # Derived: total - reserved
  reserved:  { type: integer }
  total:     { type: integer }   # Total physical stock on hand
```

#### ReserveStockRequest

```yaml
type: object
required: [order_id, lines]
properties:
  order_id: { type: string, format: uuid }
  lines:
    type: array
    minItems: 1
    items:
      type: object
      required: [sku_id, quantity]
      properties:
        sku_id:   { type: string, format: uuid }
        quantity: { type: integer, minimum: 1 }
```

#### ReserveStockResponse

```yaml
type: object
required: [reservation_id, expires_at]
properties:
  reservation_id: { type: string, format: uuid }
  expires_at:     { type: string, format: date-time }
```

#### DeductStockRequest

```yaml
type: object
required: [reservation_id]
properties:
  reservation_id: { type: string, format: uuid }
```

#### StockConflictError

```yaml
type: object
required: [code, message, conflicts]
properties:
  code:    { type: string, example: "STOCK_CONFLICT" }
  message: { type: string }
  conflicts:
    type: array
    items:
      type: object
      properties:
        sku_id:    { type: string, format: uuid }
        requested: { type: integer }
        available: { type: integer }
```

#### ErrorResponse

```yaml
type: object
required: [code, message]
properties:
  code:    { type: string }
  message: { type: string }
```

### product.schema.md (full data schema)

#### Table: `products`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key |
| `tenant_id` | `UUID` | NO | Foreign key → tenants |
| `name` | `VARCHAR(200)` | NO | Display name |
| `description` | `TEXT` | YES | Rich-text description |
| `image_url` | `VARCHAR(500)` | YES | Product thumbnail URL. NULL if no image |
| `is_active` | `BOOLEAN` | NO | Storefront visibility. Default: `true` |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp |

**Constraints:** `UNIQUE (tenant_id, name)`

#### Table: `skus`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key |
| `product_id` | `UUID` | NO | FK → products |
| `tenant_id` | `UUID` | NO | FK → tenants (denormalised for fast tenant scoping) |
| `label` | `VARCHAR(200)` | NO | Variant description (e.g. "Blue / XL") |
| `price_minor` | `INT` | NO | Selling price in minor currency units. Min 0 |
| `is_active` | `BOOLEAN` | NO | Whether variant is purchasable. Default: `true` |
| `created_at` | `TIMESTAMPTZ` | NO | UTC |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC |

**Constraints:** `UNIQUE (product_id, label)`, `price_minor >= 0`

#### Table: `stock_levels`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `sku_id` | `UUID` | NO | PK + FK → skus. One row per SKU |
| `total` | `INT` | NO | Total physical units |
| `reserved` | `INT` | NO | Units held by active reservations |
| `updated_at` | `TIMESTAMPTZ` | NO | Last stock movement time |

**Derived field (not stored):** `available = total - reserved`

**Constraints:** `total >= 0`, `reserved >= 0`, `reserved <= total`

#### Table: `stock_reservations`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key |
| `order_id` | `UUID` | NO | Pending order this reservation belongs to |
| `sku_id` | `UUID` | NO | FK → skus |
| `quantity` | `INT` | NO | Units reserved |
| `status` | `ENUM` | NO | `active`, `converted`, `expired`, `released` |
| `expires_at` | `TIMESTAMPTZ` | NO | UTC expiry (15 min from creation) |
| `created_at` | `TIMESTAMPTZ` | NO | UTC |

**Lifecycle:**
- `active`: holds stock; counts toward `stock_levels.reserved`
- `converted`: order confirmed; stock permanently deducted; `reserved` and `total` decremented
- `expired`: timer elapsed; `reserved` decremented; stock returns to available pool
- `released`: order abandoned explicitly; same effect as expired

#### Indexes

```sql
CREATE INDEX idx_products_tenant_active ON products (tenant_id, is_active);
CREATE INDEX idx_skus_product ON skus (product_id);
CREATE INDEX idx_stock_reservations_expires ON stock_reservations (expires_at)
  WHERE status = 'active';
```

---

## Implementation Notes

### Project structure

```
workspaces/inventory-service/
  src/
    __init__.py
    config.py                   # Settings via pydantic-settings + .env
    main.py                     # FastAPI app factory, lifespan, middleware
    dependencies.py             # Dependency injection (DB session, etc.)
    models/
      __init__.py
      product.py                # Product, SKU ORM models
      stock.py                  # StockLevel, StockReservation ORM models
    schemas/
      __init__.py
      product.py                # Product, SKU, CreateProductRequest, ProductPage
      stock.py                  # StockLevel, ReserveStockRequest/Response, etc.
    repositories/
      __init__.py
      product_repository.py     # CRUD + paginated queries for products/SKUs
      stock_repository.py       # Stock level + reservation data access
    services/
      __init__.py
      product_service.py        # Product creation logic
      stock_service.py          # Reserve, deduct, release, expiry business logic
    api/
      __init__.py
      health.py                 # GET /health, /ready, /metrics
      router.py                 # Mounts all route groups
      v1/
        __init__.py
        products.py             # GET /products, GET /products/{id}, POST /products
        stock.py                # GET /stock/{skuId}, POST /stock/reserve, POST /stock/deduct, POST /stock/reservations/{id}/release
  tests/
    __init__.py
    conftest.py                 # Shared fixtures: app, async DB, test client
    integration/
      test_products.py          # TS-001-001 through TS-001-007
      test_stock.py             # Stock reserve/deduct/release tests
      test_observability.py     # TS-001-056, TS-001-057, TS-001-058
  alembic/
    env.py
    versions/
  Dockerfile
  docker-compose.yml            # Service + PostgreSQL
  pyproject.toml
  .env.example
```

### Tech stack

- **Language:** Python 3.13
- **Framework:** FastAPI `^0.135`
- **ASGI:** Uvicorn `^0.43`
- **ORM:** SQLAlchemy async `^2.0`
- **DB driver:** asyncpg `^0.31`
- **Migrations:** Alembic `^1.18`
- **Validation:** Pydantic v2 `^2.12`
- **Settings:** pydantic-settings `^2.13`
- **HTTP client:** httpx `^0.28` (for any outbound calls; not needed in this WP)
- **Package manager:** uv (latest)
- **Linter:** Ruff `^0.15`
- **Type checker:** mypy `^1.20` (strict mode)
- **Test runner:** pytest `^9.0` + pytest-asyncio `^1.3`
- **Metrics:** prometheus-fastapi-instrumentator `^7.1`
- **Logging:** python-json-logger `^4.1`
- **Docker base:** `python:3.13-slim`

### Key patterns

#### Multi-tenant scoping

All queries MUST filter by `tenant_id` from the `X-Tenant-ID` header. Create a FastAPI dependency that extracts and validates this header:

```python
async def get_tenant_id(x_tenant_id: str = Header(...)) -> uuid.UUID:
    return uuid.UUID(x_tenant_id)
```

#### Stock level computation

`stock_level` in the Product/SKU API response is a **derived field**: `available = total - reserved` from the `stock_levels` table. When listing products, JOIN `stock_levels` onto `skus` and compute `available` in the query.

For the `in_stock_only=true` filter: return only products that have at least one SKU where `stock_levels.total - stock_levels.reserved > 0`.

#### Stock reservation flow

The reserve → deduct → release cycle is consumed by order-service during the checkout saga:

1. **Reserve** (`POST /stock/reserve`): Validate all SKUs have sufficient `available` stock. If any SKU fails, return `409 StockConflictError` with conflict details. If all pass, create `stock_reservations` records with `status = active`, increment `stock_levels.reserved` for each SKU, return `reservation_id` + `expires_at` (15 min from now). Use row-level locking (`SELECT ... FOR UPDATE`) on `stock_levels` to prevent race conditions.

2. **Deduct** (`POST /stock/deduct`): Find the reservation by `reservation_id`. Verify it is `active`. Decrement `stock_levels.total` and `stock_levels.reserved` by the reserved quantities. Set reservation `status = converted`.

3. **Release** (`POST /stock/reservations/{id}/release`): Find the reservation by ID. If `active`, decrement `stock_levels.reserved`, set `status = released`. If not found or already released/expired, return 204 (idempotent — treat as success).

4. **Expiry cleanup**: Reservations with `status = active` and `expires_at < now()` should be cleaned up. Implement either a background task (periodic) or lazy cleanup (check on query). For MVP, lazy cleanup on stock queries plus a periodic background task (every 60 seconds) is sufficient.

#### Idempotency

`POST /stock/reserve` and `POST /stock/deduct` accept an optional `Idempotency-Key` header. If the same key is reused, return the original response without creating duplicate reservations/deductions. Store the idempotency key → response mapping in a simple DB table or Redis cache.

#### POST /products (data seeding)

No AC covers product creation. This endpoint is for seeding catalogue data. When creating a product, also create:
- One `sku` row per entry in `skus` array
- One `stock_levels` row per SKU with `total = initial_stock`, `reserved = 0`

#### Observability

Follow `docs/architecture/observability-standards.md`:
- `GET /health` → `200 {"status": "ok", "service": "inventory-service", "version": "0.1.0"}`
- `GET /ready` → checks DB connection, returns `200` if healthy, `503` if not
- `GET /metrics` → Prometheus exposition format via `prometheus-fastapi-instrumentator`
- Structured JSON logging to stdout with `timestamp`, `level`, `service`, `trace_id`, `message`
- Propagate `X-Request-ID` header (read inbound, generate if absent, echo in response)
- No PII in logs

### Key constraints

- `stock_levels.reserved` must never exceed `stock_levels.total` — enforce in reserve logic
- `stock_levels.total` must never go negative — enforce in deduct logic
- Product names are unique per tenant: `UNIQUE (tenant_id, name)`
- SKU labels are unique per product: `UNIQUE (product_id, label)`
- Use row-level locking on `stock_levels` during reserve/deduct to prevent overselling under concurrent requests

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-001 through TS-001-007, TS-001-056, TS-001-057, TS-001-058)
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (Schemathesis against `inventory-service.openapi.yaml`)

---

## Implementation Order

Implement in this order:

1. **Scaffold workspace** — `pyproject.toml`, `Dockerfile`, `docker-compose.yml` (with PostgreSQL), `.env.example`, project directory structure per workspace-bootstrap standard
2. **DB models + Alembic migration** — `products`, `skus`, `stock_levels`, `stock_reservations` tables with all constraints, indexes, and the `stock_reservation_status` enum
3. **Repositories** — `ProductRepository` (list paginated, get by ID, create), `StockRepository` (get level, reserve, deduct, release, expire cleanup)
4. **Service layer** — `ProductService` (create product with SKUs + initial stock), `StockService` (reserve with conflict detection, deduct, release, expiry background task)
5. **API endpoints** — all 7 paths from the contract: `GET /products`, `POST /products`, `GET /products/{productId}`, `GET /stock/{skuId}`, `POST /stock/reserve`, `POST /stock/deduct`, `POST /stock/reservations/{reservationId}/release`
6. **Health/ready/metrics endpoints** — `GET /health`, `GET /ready`, `GET /metrics`
7. **Tests** — all TS scenarios (TS-001-001 through TS-001-007, TS-001-056 through TS-001-058)
8. **Run DoD checklist** — ruff, mypy, Schemathesis, secrets check

### Dependency info

No dependencies on other services. This WP can start immediately (Wave 1). storefront-app and order-service depend on this service being complete.
