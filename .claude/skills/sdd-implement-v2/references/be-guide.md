# Backend Implementation Guide (Python / FastAPI)

Reference for the sdd-implement skill when executing backend Work Packages. This guide fills gaps when the WP doesn't specify how to approach a checkpoint. **The WP always takes precedence over this guide.**

Load this file when the target workspace has `type: backend` in `registry/routes.yaml`.

---

## Project Structure

Follow the structure from `contracts/architecture/workspace-bootstrap.md`. Key directories:

```
src/
  config.py                   # Settings via pydantic-settings + .env
  main.py                     # FastAPI app factory, lifespan, middleware
  dependencies.py             # FastAPI dependency injection
  models/                     # SQLAlchemy ORM models (one file per aggregate root)
  schemas/                    # Pydantic request/response schemas
  repositories/               # Data access layer (one file per aggregate root)
  services/                   # Business logic / orchestration (no DB calls here)
  clients/                    # External HTTP client wrappers (configurable base URLs)
  api/
    health.py                 # GET /health, /ready, /metrics
    v1/                       # Versioned route groups
tests/
  conftest.py                 # Shared fixtures (app, db, client)
  unit/                       # Pure logic tests — no I/O
  integration/                # DB + service layer tests (real DB in Docker)
  contract/                   # Schemathesis + Pact tests
```

---

## Per-Checkpoint Procedures

These are fallback procedures. If the WP's Implementation Order specifies steps for a checkpoint, follow the WP. Use these only when the WP is silent on how to approach a checkpoint.

### scaffold

**Goal:** Working project skeleton that starts and responds to health checks.

1. Create `pyproject.toml` per `workspace-bootstrap.md` toolchain spec
2. Create `Dockerfile` (multi-stage) and `docker-compose.yml` (app + postgres + redis)
3. Create `.env.example` with all required environment variables
4. Create `src/main.py` with FastAPI app factory and lifespan handler
5. Create `src/config.py` with pydantic-settings (reads from env vars)
6. Create `src/dependencies.py` for DB session and Redis dependency injection
7. Create `src/api/health.py` with `/health` endpoint (liveness only — readiness comes later when DB is wired)
8. Create `tests/conftest.py` with app and client fixtures
9. Create `.github/workflows/ci.yml` per `workspace-bootstrap.md`
10. **Verify:** `docker compose up` starts, `GET /health` returns 200

```bash
# Verification commands
docker compose up -d --wait
curl http://localhost:8000/health
```

### models

**Goal:** Database schema matches the WP's data schema excerpts. Migrations run cleanly.

1. Read the data schema excerpts from the WP's "Relevant Contracts" section
2. Create ORM model files in `src/models/` — one file per aggregate root (e.g., `order.py`, `order_line.py`)
3. Create `src/models/__init__.py` re-exporting all models
4. Initialize Alembic if not already done: `alembic init alembic`
5. Configure `alembic/env.py` to use async engine and import all models
6. Generate migration: `alembic revision --autogenerate -m "add {model_name} tables"`
7. Review the generated migration — verify columns, types, constraints, indexes match the WP's schema excerpts
8. Run migration: `alembic upgrade head`
9. Wire `/ready` endpoint to check DB connectivity
10. If the WP includes seed data instructions, create a seed script
11. **Verify:** migration applies cleanly, `/ready` returns 200 with `"database": "ok"`

```bash
# Verification commands
docker compose run --rm app alembic upgrade head
curl http://localhost:8000/ready
```

### routes

**Goal:** All API endpoints exist, accept the right request shapes, return the right response shapes.

1. Create Pydantic schemas in `src/schemas/` for every request and response body in the WP's contract excerpts
2. Create repository files in `src/repositories/` — data access functions only (no business logic)
3. Create service files in `src/services/` — business logic and orchestration (calls repositories, not DB directly)
4. Create API route files in `src/api/v1/` — thin handlers that call services and return responses
5. Wire routes into the app via `src/api/router.py`
6. Add middleware: request logging, tenant validation (`X-Tenant-ID`), auth (if WP specifies)
7. Add `/metrics` endpoint using `prometheus-fastapi-instrumentator`
8. **Verify:** each endpoint responds with the correct status code for valid and invalid inputs

```bash
# Verification commands
curl -X POST http://localhost:8000/v1/{endpoint} -H "Content-Type: application/json" -d '{...}'
curl http://localhost:8000/metrics
```

**Sub-ordering within routes:** schemas → repositories → services → routes → middleware → metrics. This ensures each layer's dependencies exist before it's built.

### integration

**Goal:** External service calls work with configurable URLs and propagate tracing headers.

1. Create HTTP client wrappers in `src/clients/` — one file per external service
2. Each client uses `httpx.AsyncClient` with a configurable `base_url` from `src/config.py`
3. Add `X-Request-ID` propagation: read from inbound request, attach to all outbound calls
4. Set up structured JSON logging with `python-json-logger` (see `contracts/architecture/observability-standards.md`)
5. Add `trace_id` to every log line via middleware
6. Add custom Prometheus metrics for domain events if the WP specifies them
7. **Verify:** client calls work against mock targets, logs are structured JSON, `X-Request-ID` echoed in response headers

```python
# Quick verification in test
import respx
from httpx import Response

with respx.mock(base_url="http://inventory-service:8001") as mock:
    mock.post("/stock/reserve").mock(return_value=Response(200, json={...}))
    # Call your client, verify it works
```

### tests

**Goal:** Every TS scenario in the WP has a passing test.

1. Read the WP's "Test Scenarios" section — this is your test list
2. Create test files in `tests/integration/` — one file per feature area or AC group
3. Each TS scenario becomes one test function:
   - Name: `test_ts_{ts_number}_{sequence}_{short_description}`
   - Set up preconditions (DB fixtures, mock responses)
   - Execute the action (HTTP call via `httpx.AsyncClient`)
   - Assert expected outcomes (status code, response body, DB state, mock call counts)
4. Use `respx` for mocking external HTTP calls — verify request shapes match contract
5. Use `httpx.AsyncClient(transport=ASGITransport(app=app))` for testing endpoints
6. **Verify:** `pytest tests/` passes with all green

```bash
# Verification commands
docker compose run --rm app uv run pytest tests/ -v --tb=short
```

**Counting rule:** If the WP lists N test scenarios, you must have N test functions. Count before and after.

### dod

**Goal:** The WP's Definition of Done checklist passes completely.

1. Read the DoD section from the WP — run each check listed there
2. Fix any failures before marking the WP done
3. Common checks (verify only if the WP lists them):
   - `uv run ruff check src/` — zero errors
   - `uv run mypy src/` — zero errors
   - Schemathesis against the OpenAPI spec
   - Pact consumer tests (if WP calls another service)
   - No secrets in code (`grep -r "password\|secret\|token" src/` should find only variable names, not values)

```bash
# Common verification commands
docker compose run --rm app uv run ruff check src/
docker compose run --rm app uv run mypy src/
docker compose run --rm app uv run schemathesis run ../../contracts/api/{service}.openapi.yaml --base-url http://localhost:8000 --checks all
```

---

## Testing Patterns

### Async test setup (conftest.py)

```python
import pytest
from httpx import ASGITransport, AsyncClient
from src.main import create_app

@pytest.fixture
async def app():
    app = create_app()
    yield app

@pytest.fixture
async def client(app):
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac
```

### Mocking external services (respx)

```python
import respx
from httpx import Response

@pytest.fixture
def mock_inventory():
    with respx.mock(base_url="http://inventory-service:8001") as mock:
        yield mock

async def test_stock_reservation(client, mock_inventory):
    mock_inventory.post("/stock/reserve").mock(
        return_value=Response(200, json={"reservation_id": "uuid", "expires_at": "..."})
    )
    response = await client.post("/checkout/guest/orders", json={...})
    assert response.status_code == 201
    assert mock_inventory.post("/stock/reserve").called
```

### DB fixtures (real PostgreSQL in Docker)

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@pytest.fixture
async def db_session():
    engine = create_async_engine(settings.test_database_url)
    async with AsyncSession(engine) as session:
        yield session
        await session.rollback()
```

### Testing saga/orchestration flows

```python
async def test_order_saga_stock_conflict(client, mock_inventory, mock_payment):
    # Stock reservation fails
    mock_inventory.post("/stock/reserve").mock(
        return_value=Response(409, json={"code": "STOCK_CONFLICT", "conflicts": [...]})
    )

    response = await client.post("/checkout/guest/orders", json={...})

    assert response.status_code == 409
    # Payment should NOT have been called
    assert not mock_payment.post("/capture").called
```

---

## Common Pitfalls

| Pitfall | Fix |
|---|---|
| Hardcoded service URLs | Use `pydantic-settings` with env vars. Every external URL is configurable. |
| Sync DB calls in async routes | Use `sqlalchemy[asyncio]` + `asyncpg`. Never use sync engine in async context. |
| Tests that depend on execution order | Each test creates its own data. Use fixtures, not shared state. |
| Missing `X-Tenant-ID` header validation | Every endpoint must validate the tenant header. Add middleware or dependency. |
| Logging PII (emails, tokens) | Structured logging with field allowlists. Ruff rule `S320` for enforcement. |
| Forgetting to propagate `X-Request-ID` | Use the `TracingTransport` pattern from `contracts/architecture/observability-standards.md`. |
| Alembic migration that doesn't match the model | Generate migration from model diff: `alembic revision --autogenerate`. Verify the generated SQL. |
| Schemathesis failures on auth endpoints | Configure Schemathesis with `--header "X-Tenant-ID: ..."` and auth fixtures in `tests/contract/conftest.py`. |
| Committing to the spec-hub repo | All implementation code goes in the workspace submodule. The only spec-hub commit is the submodule reference. |
