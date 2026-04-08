# WP-002-BE: Product Catalog & Stock API — Backend

**Feature:** Story-0002-product-catalog
**Derived from:** FS-002, TS-002
**Target workspace:** `workspaces/inventory-service`
**Status:** Approved
**Last updated:** 2026-04-04

---

## Objective

Scaffold and implement the `inventory-service` — a containerised FastAPI backend providing the product catalogue and stock management API. This WP delivers all 8 ACs from FS-002 and all 29 test scenarios from TS-002.

This WP is self-contained. An implementer can complete it reading only this file.

---

## Acceptance Criteria (copied verbatim from FS-002)

**AC-001 — Create a product with SKUs**
A merchant admin can create a product with one or more SKUs. Each SKU has a price and an initial stock quantity. `POST /products` returns HTTP 201 with the created product including all assigned IDs. Initial stock is immediately queryable via `GET /stock/{skuId}`.

**AC-002 — List products with pagination**
`GET /products` returns HTTP 200 with `data` array and `meta` object (total, page, per_page). Supports `in_stock_only=true` filter and `page`/`per_page` query params.

**AC-003 — Get a single product by ID**
`GET /products/{productId}` returns HTTP 200 with full product details. Returns HTTP 404 for non-existent or cross-tenant IDs.

**AC-004 — Query stock level for a SKU**
`GET /stock/{skuId}` returns `total`, `reserved`, and `available` (= total − reserved). Returns HTTP 404 for unknown SKU IDs.

**AC-005 — Reserve stock (internal)**
`POST /stock/reserve` places a temporary hold. Returns HTTP 200 with `reservation_id` and `expires_at` (~15 min). Returns HTTP 409 with `STOCK_CONFLICT` if any SKU has insufficient stock. No partial reservations. Concurrent requests must not allow overselling.

**AC-006 — Deduct reserved stock on order confirmation**
`POST /stock/deduct` converts a reservation to a permanent deduction. Returns HTTP 200. Second call with same `reservation_id` returns HTTP 409 (idempotency guard).

**AC-007 — Release a reservation on saga compensation**
`POST /stock/reservations/{reservationId}/release` returns HTTP 204. Returns HTTP 404 for unknown/already-released reservations (callers treat 404 as no-op success).

**AC-008 — Service health and observability**
`GET /health` → 200 always. `GET /ready` → 200 when DB available, 503 when DB unreachable. `GET /metrics` → valid Prometheus text format.

---

## Test Scenarios (copied verbatim from TS-002)

**TS-002-001** — `POST /products` single SKU → HTTP 201, product `id` present, SKU `id` present, `GET /stock/{sku.id}` → `total: 50, reserved: 0, available: 50`.

**TS-002-002** — `POST /products` three SKUs → HTTP 201, three SKUs with unique IDs, each independently queryable.

**TS-002-003** — `POST /products` missing `name` → HTTP 422.

**TS-002-004** — `POST /products` empty `skus` array → HTTP 422.

**TS-002-005** — `POST /products` SKU with `price_minor: -1` → HTTP 422.

**TS-002-006** — `GET /products` with 25 products → 20 returned, `meta.total: 25, meta.page: 1`.

**TS-002-007** — `GET /products?page=2&per_page=20` with 25 products → 5 returned, `meta.page: 2`.

**TS-002-008** — `GET /products?in_stock_only=true` with mixed stock → zero-stock products excluded.

**TS-002-009** — `GET /products` with no products → `data: [], meta.total: 0`.

**TS-002-010** — `GET /products` with Tenant B token → only Tenant B products returned.

**TS-002-011** — `GET /products/{id}` existing product → HTTP 200 with `skus` array.

**TS-002-012** — `GET /products/{nonExistentId}` → HTTP 404.

**TS-002-013** — `GET /products/{id}` cross-tenant → HTTP 404.

**TS-002-014** — `GET /stock/{skuId}` → `total: 100, reserved: 15, available: 85`.

**TS-002-015** — `GET /stock/{nonExistentId}` → HTTP 404.

**TS-002-016** — `GET /stock/{skuId}` after reservation of qty 5 → `reserved: 5, available: 15, total: 20`.

**TS-002-017** — `POST /stock/reserve` two SKUs in stock → HTTP 200, `reservation_id`, `expires_at` ~15 min, stock levels updated.

**TS-002-018** — `POST /stock/reserve` one SKU insufficient → HTTP 409, `STOCK_CONFLICT`, one entry in `conflicts`, no reservation created, other SKU unchanged.

**TS-002-019** — `POST /stock/reserve` both SKUs at zero → HTTP 409, both in `conflicts`.

**TS-002-020** — Two concurrent `POST /stock/reserve` each requesting qty 4 of a SKU with available 5 → exactly one succeeds (HTTP 200), the other returns HTTP 409; final `reserved` is 4 not 8.

**TS-002-021** — `POST /stock/deduct` valid `reservation_id` → HTTP 200; `total` and `reserved` both decremented.

**TS-002-022** — `POST /stock/deduct` same `reservation_id` twice → second call HTTP 409.

**TS-002-023** — `POST /stock/deduct` non-existent `reservation_id` → HTTP 409.

**TS-002-024** — `POST /stock/reservations/{id}/release` active reservation → HTTP 204; `reserved` decremented.

**TS-002-025** — `POST /stock/reservations/{nonExistent}/release` → HTTP 404.

**TS-002-026** — `GET /health` → HTTP 200, `{ "status": "ok" }`.

**TS-002-027** — `GET /ready` DB available → HTTP 200.

**TS-002-028** — `GET /ready` DB unreachable → HTTP 503.

**TS-002-029** — `GET /metrics` → HTTP 200, `Content-Type: text/plain`, Prometheus metric lines present.

---

## API Contract

All endpoints are defined in `contracts/api/inventory-service.openapi.yaml`. Implement exactly as specified. Do not add, remove, or rename fields from the contract schemas.

### Endpoints to implement

| Method | Path | Operation ID | Notes |
|---|---|---|---|
| `POST` | `/products` | `createProduct` | |
| `GET` | `/products` | `listProducts` | pagination + in_stock_only |
| `GET` | `/products/{productId}` | `getProduct` | |
| `GET` | `/stock/{skuId}` | `getStockLevel` | |
| `POST` | `/stock/reserve` | `reserveStock` | uses DB row-level locking |
| `POST` | `/stock/deduct` | `deductStock` | |
| `POST` | `/stock/reservations/{reservationId}/release` | `releaseReservation` | |
| `GET` | `/health` | — | not in OpenAPI; see observability standards |
| `GET` | `/ready` | — | not in OpenAPI; see observability standards |
| `GET` | `/metrics` | — | Prometheus; instrumentator auto-generates |

---

## Database Schema

Implement using SQLAlchemy async ORM. Tables match `contracts/data-schema/product.schema.md`.

### `products`
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
tenant_id   UUID NOT NULL
name        VARCHAR(200) NOT NULL
description TEXT
is_active   BOOLEAN NOT NULL DEFAULT TRUE
created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()

INDEX idx_products_tenant_id (tenant_id)
```

### `skus`
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
product_id  UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE
tenant_id   UUID NOT NULL
label       VARCHAR(200) NOT NULL
price_minor INTEGER NOT NULL CHECK (price_minor >= 0)
is_active   BOOLEAN NOT NULL DEFAULT TRUE
created_at  TIMESTAMPTZ NOT NULL DEFAULT now()

INDEX idx_skus_product_id (product_id)
INDEX idx_skus_tenant_id (tenant_id)
```

### `stock_levels`
```sql
sku_id      UUID PRIMARY KEY REFERENCES skus(id) ON DELETE CASCADE
total       INTEGER NOT NULL DEFAULT 0 CHECK (total >= 0)
reserved    INTEGER NOT NULL DEFAULT 0 CHECK (reserved >= 0)
-- available = total - reserved (computed, not stored)
```

### `stock_reservations`
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid()
order_id    UUID NOT NULL
sku_id      UUID NOT NULL REFERENCES skus(id)
tenant_id   UUID NOT NULL
quantity    INTEGER NOT NULL CHECK (quantity > 0)
status      VARCHAR(20) NOT NULL DEFAULT 'active'
            -- values: active | converted | released | expired
expires_at  TIMESTAMPTZ NOT NULL
created_at  TIMESTAMPTZ NOT NULL DEFAULT now()

INDEX idx_reservations_sku_id (sku_id)
INDEX idx_reservations_order_id (order_id)
INDEX idx_reservations_expires_at (expires_at)
```

---

## Implementation Notes

### Project structure

Follow the layout defined in `contracts/architecture/workspace-bootstrap.md`:

```
workspaces/inventory-service/
├── src/
│   ├── main.py
│   ├── config.py
│   ├── api/
│   │   ├── router.py
│   │   ├── health.py
│   │   └── v1/
│   │       ├── products.py
│   │       └── stock.py
│   ├── services/
│   │   ├── product_service.py
│   │   └── stock_service.py
│   ├── repositories/
│   │   ├── product_repository.py
│   │   └── stock_repository.py
│   ├── models/
│   │   └── inventory.py        # All ORM models in one file (small service)
│   ├── schemas/
│   │   ├── product.py
│   │   └── stock.py
│   └── dependencies.py
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_product_service.py
│   │   └── test_stock_service.py
│   └── integration/
│       ├── test_products_api.py
│       └── test_stock_api.py
├── alembic/
│   └── versions/
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
└── .env.example
```

### Stock reservation — concurrency safety

**Critical:** Use PostgreSQL row-level locking to prevent overselling.

In `stock_repository.py`, the reserve operation must:
1. `SELECT ... FOR UPDATE` on all `stock_levels` rows for the requested SKUs within a single transaction.
2. Check `total - reserved >= requested_quantity` for each SKU.
3. If any check fails, roll back and return the conflicts — do not update any row.
4. If all checks pass, `UPDATE stock_levels SET reserved = reserved + quantity` for all rows.
5. Insert the `stock_reservations` record.

```python
# Pseudocode for reserve operation (single transaction)
async with session.begin():
    rows = await session.execute(
        select(StockLevel)
        .where(StockLevel.sku_id.in_(sku_ids))
        .with_for_update()          # row-level lock
    )
    conflicts = [r for r in rows if r.total - r.reserved < requested[r.sku_id]]
    if conflicts:
        raise StockConflictError(conflicts)
    for row in rows:
        row.reserved += requested[row.sku_id]
    session.add(StockReservation(...))
```

This pattern ensures atomicity under concurrent load (satisfies TS-002-020).

### Deduct stock

Within a single transaction:
1. Load the reservation by ID, lock it with `SELECT ... FOR UPDATE`.
2. If not found or status != `active`, raise a conflict error (HTTP 409).
3. Set reservation status to `converted`.
4. `UPDATE stock_levels SET total = total - qty, reserved = reserved - qty`.

### Release reservation

Within a single transaction:
1. Load reservation by ID.
2. If not found or status != `active`, return HTTP 404.
3. Set reservation status to `released`.
4. `UPDATE stock_levels SET reserved = reserved - qty`.

### Tenant isolation

All queries MUST filter by `tenant_id` extracted from the `X-Tenant-ID` request header. Implement a FastAPI dependency:

```python
# dependencies.py
def get_tenant_id(x_tenant_id: str = Header(...)) -> uuid.UUID:
    return uuid.UUID(x_tenant_id)
```

Every repository method accepts `tenant_id: uuid.UUID` as a required parameter. Never omit this filter.

### Error response format

All 4xx errors MUST use the `ErrorResponse` schema: `{ "code": "SNAKE_CASE_CODE", "message": "Human-readable string" }`.

Map domain exceptions to HTTP status codes in the route layer only:
- `ProductNotFoundError` → 404
- `SKUNotFoundError` → 404
- `StockConflictError` → 409 with `StockConflictError` schema
- `AlreadyDeductedError` → 409
- `ValidationError` (Pydantic) → 422 (FastAPI handles automatically)

### Observability

Follow `contracts/architecture/observability-standards.md`:

- `GET /health` — always returns `{"status": "ok"}` from `api/health.py`. No DB check.
- `GET /ready` — attempts a `SELECT 1` against the DB. Returns `{"status": "ready"}` on success, `{"status": "unavailable", "detail": "database"}` with HTTP 503 on failure.
- `GET /metrics` — provided automatically by `prometheus-fastapi-instrumentator`. Mount in `main.py` lifespan:

```python
from prometheus_fastapi_instrumentator import Instrumentator

@asynccontextmanager
async def lifespan(app: FastAPI):
    Instrumentator().instrument(app).expose(app)
    yield
```

### Structured logging

Use `python-json-logger` configured in `main.py`. Every log line must be valid JSON. Log level from `config.py` `Settings.log_level`.

### Settings (`config.py`)

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_env: str = "development"
    log_level: str = "INFO"
    database_url: str
    default_tenant_id: str = "00000000-0000-0000-0000-000000000001"

    model_config = {"env_file": ".env"}
```

### `docker-compose.yml` (inventory-service specific)

```yaml
services:
  app:
    build: .
    ports:
      - "8001:8000"          # inventory-service on port 8001
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: inventory
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
```

### `.env` (for Docker — all hostnames are service names)

```dotenv
APP_ENV=development
LOG_LEVEL=INFO
DATABASE_URL=postgresql+asyncpg://app:app@postgres:5432/inventory
DEFAULT_TENANT_ID=00000000-0000-0000-0000-000000000001
```

---

## Test Strategy

### `tests/conftest.py`

```python
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from src.main import create_app
from src.models.inventory import Base

TEST_TENANT_ID = "00000000-0000-0000-0000-000000000001"
TEST_TENANT_ID_2 = "00000000-0000-0000-0000-000000000002"

@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"

@pytest.fixture(scope="session")
async def engine():
    # Uses DATABASE_URL from env (set by docker compose)
    from src.config import get_settings
    e = create_async_engine(get_settings().database_url)
    async with e.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield e
    async with e.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await e.dispose()

@pytest.fixture
async def session(engine):
    async with AsyncSession(engine) as s:
        yield s
        await s.rollback()

@pytest.fixture
async def client(engine):
    app = create_app()
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as c:
        yield c

@pytest.fixture
def tenant_headers():
    return {"X-Tenant-ID": TEST_TENANT_ID}

@pytest.fixture
def tenant2_headers():
    return {"X-Tenant-ID": TEST_TENANT_ID_2}
```

### Integration test pattern (map 1:1 to TS scenarios)

```python
# tests/integration/test_products_api.py

async def test_ts_002_001_create_product_single_sku(client, tenant_headers):
    """TS-002-001: POST /products single SKU → 201, IDs present, stock queryable."""
    response = await client.post("/products", json={
        "name": "Widget Pro",
        "description": "A high-quality widget.",
        "skus": [{"label": "Blue / M", "price_minor": 1999, "initial_stock": 50}],
    }, headers=tenant_headers)
    assert response.status_code == 201
    data = response.json()
    assert "id" in data
    sku_id = data["skus"][0]["id"]

    stock = await client.get(f"/stock/{sku_id}", headers=tenant_headers)
    assert stock.status_code == 200
    assert stock.json() == {"sku_id": sku_id, "total": 50, "reserved": 0, "available": 50}
```

Name every test function `test_ts_002_XXX_<description>` for direct traceability.

---

## Definition of Done

Before marking `WP-002-BE` as `done` in `status.yaml`:

- [ ] All 29 TS-002 scenarios have passing automated tests
- [ ] `uv run ruff check .` exits 0
- [ ] `uv run ruff format --check .` exits 0
- [ ] `uv run mypy src/` exits 0
- [ ] `docker compose build` succeeds
- [ ] `docker compose up -d && docker compose run --rm app uv run alembic upgrade head && docker compose run --rm app uv run pytest` exits 0
- [ ] `/health`, `/ready`, `/metrics` endpoints return correct responses
- [ ] No secrets or credentials in code or test fixtures

---

## Related Contracts

- `contracts/api/inventory-service.openapi.yaml` — full API spec (authoritative).
- `contracts/data-schema/product.schema.md` — DB table definitions.
- `contracts/architecture/observability-standards.md` — health/metrics requirements.
- `contracts/architecture/workspace-bootstrap.md` — toolchain and scaffold standard.
