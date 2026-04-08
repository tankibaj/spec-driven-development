# WP-001-BE: Guest Checkout — Backend Work Package

**Feature:** Story-0001-guest-checkout
**Target workspace:** `workspaces/order-service`
**Status:** approved
**Generated:** 2026-04-04

---

## Objective

Implement the backend for guest checkout in the `order-service`. This covers:
- Guest session creation (Redis-backed, 24 h TTL).
- Order placement saga (reserve stock → capture payment → create order → notify).
- Idempotent order placement via `Idempotency-Key` header.
- Order retrieval.
- Full observability (`/health`, `/ready`, `/metrics`).

All external service calls (inventory-service, payment gateway, notification-service) are made via configurable HTTP clients so they can be mocked in tests with `respx`.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-001 — Guest session creation
A shopper who has not signed in can initiate a Guest Checkout session without providing any personal details upfront.
**Testable:** `POST /checkout/guest/sessions` returns a session token and expiry timestamp.

### AC-002 — Collect shipping information
During guest checkout, the shopper must provide: full name, email address, shipping address (line1, city, postal_code, country_code required; line2 and state optional). The form MUST validate the email format and all required address fields before allowing progress to the next step.
**Testable:** Submitting the checkout form with a missing required field prevents progression and displays a field-level error.

### AC-003 — Shipping method selection
The shopper must be presented with at least one shipping method and its cost before proceeding to payment. The selected shipping method and cost must be reflected in the order total.
**Testable:** Selecting a shipping method updates the displayed order total to include the shipping cost.

### AC-004 — Payment entry
The shopper can enter card details via the payment gateway's embedded widget. The system must not store raw card numbers.
**Testable:** A valid card token is returned by the payment gateway SDK and passed to the order placement API. Raw card data never appears in any service log or database.

### AC-005 — Order placement — happy path
When the shopper submits valid shipping and payment details, the system:
1. Validates the cart is non-empty and all items are in stock.
2. Reserves stock for all cart items.
3. Creates a Payment Intent and captures the payment.
4. Creates an Order record with status `confirmed`.
5. Sends an order confirmation email to the provided guest email address.
6. Returns the Order Reference to the storefront.
**Testable:** A POST to `POST /checkout/guest/orders` with valid inputs returns HTTP 201 and an Order object with `status: confirmed` and a non-empty `reference`.

### AC-006 — Out-of-stock handling
If any item in the cart is out of stock at the moment of order placement, the system MUST:
- Not charge the shopper.
- Not create an Order record.
- Return a clear error identifying the out-of-stock SKU(s).
**Testable:** A POST to `POST /checkout/guest/orders` where one SKU has zero available stock returns HTTP 409 with a `STOCK_CONFLICT` error body listing the conflicting SKUs.

### AC-007 — Idempotent order placement
If the network fails after the server processes the request, a retry with the same `Idempotency-Key` header MUST return the original response (HTTP 201 + same Order) without creating a duplicate order or charging the card twice.
**Testable:** Two identical POST requests to `POST /checkout/guest/orders` with the same `Idempotency-Key` return identical responses and produce exactly one Order record in the database.

### AC-008 — Order confirmation email
After a successful order placement, the shopper receives a transactional email containing: Order Reference, list of items, shipping address, order total, estimated delivery window.
**Testable:** After order placement, the Notification Service is called with `template_id: order_confirmation` and a payload matching the order details.

---

## Test Scenarios (verbatim from TS-001)

### TS-001-001 — Create guest session happy path
- **Preconditions:** order-service running; Redis healthy; valid X-Tenant-ID header.
- **Action:** `POST /checkout/guest/sessions` with empty body.
- **Expected:** HTTP 201; body contains `id` (UUID), `token` (non-empty string), `expires_at` (ISO datetime ~24 h from now); token stored in Redis with TTL ≤86400 s.

### TS-001-002 — Two consecutive sessions produce unique tokens
- **Preconditions:** order-service running; Redis healthy.
- **Action:** `POST /checkout/guest/sessions` twice with same tenant.
- **Expected:** Both 201; `token` values differ.

### TS-001-003 — Order with invalid email format → 422
- **Preconditions:** Valid guest session token.
- **Action:** `POST /checkout/guest/orders` with `email: "not-an-email"`, otherwise valid.
- **Expected:** HTTP 422; `email` field identified as invalid.

### TS-001-004 — Order missing postal_code → 422
- **Preconditions:** Valid guest session token.
- **Action:** `POST /checkout/guest/orders` with `shipping_address` missing `postal_code`.
- **Expected:** HTTP 422.

### TS-001-005 — Order missing address line1 → 422
- **Preconditions:** Valid guest session token.
- **Action:** `POST /checkout/guest/orders` with `shipping_address` missing `line1`.
- **Expected:** HTTP 422.

### TS-001-006 — Order with empty lines array → 422
- **Preconditions:** Valid guest session token.
- **Action:** `POST /checkout/guest/orders` with `lines: []`.
- **Expected:** HTTP 422.

### TS-001-007 — Order missing lines field → 422
- **Preconditions:** Valid guest session token.
- **Action:** `POST /checkout/guest/orders` with no `lines` field.
- **Expected:** HTTP 422.

### TS-001-008 — Valid shipping_method_id → shipping cost in total
- **Preconditions:** Shipping method `SM-STANDARD` (500 minor units) seeded in DB; valid session; inventory + payment mocks return success.
- **Action:** `POST /checkout/guest/orders` with `shipping_method_id`: SM-STANDARD ID; one line (unit_price=1000, qty=1).
- **Expected:** HTTP 201; `shipping_cost == 500`; `total == 1500`.

### TS-001-009 — Unknown shipping_method_id → 400
- **Preconditions:** Valid guest session.
- **Action:** `POST /checkout/guest/orders` with random UUID for `shipping_method_id`.
- **Expected:** HTTP 400; no order created.

### TS-001-010 — No raw card data in response or DB
- **Preconditions:** Valid session; mocks return success.
- **Action:** `POST /checkout/guest/orders` with `payment.token: "tok_test_abc123"`.
- **Expected:** HTTP 201; response body has no `card_number`/`pan`/`cvv`/`expiry` field; DB orders table has no such column.

### TS-001-011 — Happy path: 201, confirmed, valid reference
- **Preconditions:** Valid session; inventory mock → 200 with `reservation_id`; payment mock → 200 with `payment_intent_id`; notification mock → 202.
- **Action:** `POST /checkout/guest/orders` with all valid fields.
- **Expected:** HTTP 201; `status == "confirmed"`; `reference` matches `^ORD-\d{8}-[A-Z0-9]{4}$`; `id` is UUID; `lines` present; `total` correct.

### TS-001-012 — Placed order retrievable via GET /orders/{id}
- **Preconditions:** Order placed successfully.
- **Action:** `GET /orders/{orderId}`.
- **Expected:** HTTP 200; body matches placed order.

### TS-001-013 — Inventory STOCK_CONFLICT → 409, no charge
- **Preconditions:** Valid session; inventory mock → 409 STOCK_CONFLICT; payment mock configured but must not be called.
- **Action:** `POST /checkout/guest/orders` with conflicting SKU.
- **Expected:** HTTP 409; `code == "STOCK_CONFLICT"`; `conflicts` non-empty; payment mock received 0 calls.

### TS-001-014 — No order record created on stock conflict
- **Preconditions:** Same as TS-001-013.
- **Action:** `POST /checkout/guest/orders` with conflicting SKU.
- **Expected:** HTTP 409; order count in DB unchanged.

### TS-001-015 — Same Idempotency-Key → identical response, one DB row
- **Preconditions:** Valid session; mocks return success; `Idempotency-Key: "idem-key-001"`.
- **Action:** POST twice with same key and payload.
- **Expected:** Both 201; bodies identical (same `id`, `reference`); exactly one DB row with that key; payment mock called once.

### TS-001-016 — Different keys → two distinct orders
- **Preconditions:** Mocks return success.
- **Action:** POST with `key-A`, then POST with `key-B` (separate sessions).
- **Expected:** Both 201; different `id` values; two DB rows.

### TS-001-017 — Notification service called with correct template
- **Preconditions:** Valid session; mocks return success; notification mock captures requests.
- **Action:** `POST /checkout/guest/orders` with `email: "guest@example.com"`.
- **Expected:** HTTP 201; notification mock received exactly one call; call body has `channel: "email"`, `template_id: "order_confirmation"`, `recipient.address: "guest@example.com"`, `payload.order_reference` matches placed order reference.

### TS-001-018 — GET /health → 200
- **Action:** `GET /health`.
- **Expected:** HTTP 200; `{"status": "ok"}`.

### TS-001-019 — GET /ready → 200 when DB and Redis available
- **Action:** `GET /ready`.
- **Expected:** HTTP 200; `body.status == "ready"`.

### TS-001-020 — GET /metrics → 200 Prometheus format
- **Action:** `GET /metrics`.
- **Expected:** HTTP 200; `Content-Type` contains `text/plain`.

---

## Relevant Contracts

### order-service.openapi.yaml (excerpts)

```
POST /checkout/guest/sessions → 201 GuestSession {id, token, expires_at}
POST /checkout/guest/orders
  Security: X-Guest-Session-Token
  Headers: X-Tenant-ID (required), Idempotency-Key (optional, max 64 chars)
  Body: PlaceGuestOrderRequest {email, shipping_address, shipping_method_id, payment_method, lines}
  200 → Order; 409 → StockConflictError
GET /orders/{orderId} → 200 Order

PlaceGuestOrderRequest.lines: minItems: 1
Address required fields: line1, city, postal_code, country_code
PaymentMethodInput: {type: enum[card, digital_wallet], token: string}
Order: {id, reference, status, lines, subtotal, shipping_cost, tax, total, shipping_address, created_at}
OrderStatus: enum[pending, confirmed, processing, shipped, delivered, cancelled]
```

### order.schema.md (key constraints)
- `orders.idempotency_key`: VARCHAR(64); UNIQUE(tenant_id, idempotency_key).
- `orders.reference`: VARCHAR(30); format `ORD-YYYYMMDD-{4 uppercase alphanumeric}`; UNIQUE(tenant_id, reference); retry up to 3× on collision.
- `orders.guest_email`: nullable; set for guest orders.
- `orders.customer_id`: NULL for guest orders.
- No `card_number`, `pan`, `cvv`, or `expiry` column anywhere in the schema.
- `orders.payment_intent_id`: external gateway reference (string), NOT raw card data.

### inventory-service.openapi.yaml (relevant endpoints)
```
POST /stock/reserve
  Body: {order_id: UUID, lines: [{sku_id: UUID, quantity: int}]}
  200 → {reservation_id: UUID, expires_at: datetime}
  409 → StockConflictError {code, message, conflicts: [{sku_id, requested, available}]}

POST /stock/reservations/{reservationId}/release → 204
```

### notification-service.openapi.yaml (relevant endpoint)
```
POST /notifications
  Headers: X-Tenant-ID, Idempotency-Key
  Body: {channel: "email"|"sms", template_id: string, recipient: {address, name?}, payload: object}
  202 → NotificationReceipt {id, status, ...}
```

### ADR-001 (guest session design)
- Sessions stored in Redis, NOT in PostgreSQL.
- Token is opaque (not a JWT).
- Session key: `session:{tenant_id}:{token}`.
- TTL: 86400 seconds (24 hours).
- Session invalidated (deleted from Redis) on successful order placement.

---

## Implementation Notes

### Project structure
```
workspaces/order-service/
  src/
    config.py                   # Settings: DATABASE_URL, REDIS_URL, INVENTORY_SERVICE_URL,
                                #   PAYMENT_GATEWAY_URL, NOTIFICATION_SERVICE_URL
    main.py                     # FastAPI app; Instrumentator outside lifespan
    dependencies.py             # DbSession, TenantID, RedisClient, GuestSessionToken deps
    models/order.py             # Order, OrderLine, ShippingMethod ORM models
    schemas/
      checkout.py               # GuestSessionResponse, PlaceGuestOrderRequest, OrderResponse
    repositories/
      order_repository.py       # CRUD for orders + order_lines
      shipping_repository.py    # Lookup shipping_methods by id
      session_repository.py     # Redis get/set/delete for guest sessions
    services/
      checkout_service.py       # Saga orchestration
    clients/
      inventory_client.py       # httpx.AsyncClient wrapper for inventory-service
      payment_client.py         # httpx.AsyncClient wrapper for payment gateway
      notification_client.py    # httpx.AsyncClient wrapper for notification-service
    api/
      health.py                 # GET /health, GET /ready
      router.py
      v1/
        checkout.py             # POST /checkout/guest/sessions, POST /checkout/guest/orders
        orders.py               # GET /orders/{orderId}
  tests/
    conftest.py
    integration/
      test_checkout_api.py      # All 20 TS-001 scenarios
  alembic/
    versions/
      0001_initial_schema.py    # creates orders, order_lines, shipping_methods; seeds 2 methods
  Dockerfile
  docker-compose.yml
  pyproject.toml
  .env
  .env.example
  alembic.ini
```

### Tech stack (from workspace-bootstrap.md)
- Python 3.13, FastAPI 0.135, SQLAlchemy 2.0 async, asyncpg, Alembic 1.18
- `redis[asyncio]` (aioredis) for session storage
- `respx` for mocking external HTTP calls in tests
- pytest 9, pytest-asyncio 1.3, asyncio_mode=auto
- PostgreSQL 17 (orders), Redis 7 (sessions)

### Saga flow (checkout_service.py)

```
async def place_guest_order(tenant_id, session_token, payload, idempotency_key) -> Order:
    1. Look up session in Redis → 401 if not found
    2. Validate shipping_method_id in DB → 400 if not found
    3. Check idempotency: SELECT orders WHERE tenant_id=X AND idempotency_key=Y
       - If found: return existing order (HTTP 201)
    4. Build reservation request from payload.lines
    5. inventory_client.reserve_stock(tenant_id, order_id, lines)
       - On 409: raise StockConflictError → endpoint returns 409
    6. payment_client.create_and_capture_intent(amount=total, token=payload.payment.token)
       - On failure: release reservation (best-effort), raise PaymentError
    7. BEGIN DB TRANSACTION
       - Generate reference (ORD-YYYYMMDD-{4 chars}); retry up to 3× on UNIQUE violation
       - INSERT order (status=confirmed, idempotency_key, payment_intent_id, ...)
       - INSERT order_lines
       - COMMIT
    8. Delete session from Redis (invalidate on success)
    9. asyncio.create_task(notification_client.send_order_confirmation(order, email))
    10. Return order
```

### Order reference generation
```python
import random, string
def _make_reference(date_str: str) -> str:
    suffix = "".join(random.choices(string.ascii_uppercase + string.digits, k=4))
    return f"ORD-{date_str}-{suffix}"
```
Retry loop: attempt insert; on `UniqueViolationError` for `reference`, regenerate and retry (max 3 attempts).

### Guest session storage (Redis)
```python
# Key format:
key = f"guest_session:{tenant_id}:{token}"
value = json.dumps({"id": str(session_id), "tenant_id": str(tenant_id), "created_at": iso_str})
await redis.set(key, value, ex=86400)
```

### External HTTP clients
Each client takes a base URL from config and uses a shared `httpx.AsyncClient`. URL config env vars:
- `INVENTORY_SERVICE_URL` — e.g. `http://inventory-service:8000`
- `PAYMENT_GATEWAY_URL` — e.g. `http://mock-payment:8000`
- `NOTIFICATION_SERVICE_URL` — e.g. `http://notification-service:8000`

In tests, `respx.mock` intercepts calls to these URLs.

### Shipping methods — DB seed
Seed two shipping methods in migration `0001_initial_schema.py` with known fixed UUIDs so tests can reference them:
```python
STANDARD_ID = "00000000-0000-0000-0000-000000000010"
EXPRESS_ID  = "00000000-0000-0000-0000-000000000011"
```
- Standard: name="Standard Delivery", cost_minor=500, estimated_days=5
- Express: name="Express Delivery", cost_minor=1500, estimated_days=2

### Test mocking pattern
```python
import respx
import httpx

@pytest.fixture
def mock_inventory_success(sku_id: str) -> respx.Router:
    with respx.mock(base_url=settings.INVENTORY_SERVICE_URL) as mock:
        mock.post("/stock/reserve").mock(
            return_value=httpx.Response(200, json={
                "reservation_id": str(uuid.uuid4()),
                "expires_at": (datetime.now(UTC) + timedelta(minutes=15)).isoformat()
            })
        )
        yield mock
```

### Key constraints from learnings
- Use `model_dump(mode="json")` in HTTPException details (UUIDs must be strings).
- Call `Instrumentator().instrument(app).expose(app)` in `create_app()`, NOT in lifespan.
- Do NOT use `@pytest.mark.anyio`; rely on `asyncio_mode = "auto"`.
- Per-test async engine fixture to avoid asyncpg loop-binding issues.
- Notification dispatch is fire-and-forget — a notification failure MUST NOT roll back the order.

### docker-compose.yml services
- `app`: runtime target, port 8002.
- `test`: test target; depends on postgres + redis.
- `migrate`: runs `alembic upgrade head`.
- `postgres`: postgres:17-alpine, DB=orders.
- `redis`: redis:7-alpine.

### Definition of Done
Before marking this WP done:
- [ ] All 20 TS-001 scenarios pass in Docker via `docker compose --profile test run --rm test`
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors (strict mode)
- [ ] `/health`, `/ready`, `/metrics` endpoints all return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures

---

## Implementation Order

Implement in this order:
1. Scaffold workspace (pyproject.toml, Dockerfile, docker-compose.yml, .env).
2. DB models and Alembic migration (orders, order_lines, shipping_methods with seed data).
3. Redis session repository.
4. HTTP clients (inventory, payment, notification) with configurable base URLs.
5. Order repository (create, get_by_id, get_by_idempotency_key).
6. Checkout service (saga).
7. API endpoints (`/checkout/guest/sessions`, `/checkout/guest/orders`, `/orders/{id}`).
8. Health/ready/metrics endpoints.
9. Tests (all 20 scenarios using respx mocks).
10. Run DoD checklist.
