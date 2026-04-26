# WP-003-BE: MVP Guest Shopping — Order Placement Saga

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/order-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the order placement saga in order-service: validate the request, reserve stock (inventory-service), capture payment (Stripe), deduct stock (inventory-service), persist the order, and return a confirmed order with a human-readable reference. Handle compensation on failure (release stock reservations on payment failure). This WP also scaffolds the order-service workspace and implements observability endpoints (`/health`, `/ready`, `/metrics`).

All external service calls (inventory-service, Stripe, notification-service) are made via configurable HTTP clients so they can be mocked in tests.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-012 — Successful order placement

After form submission, the system reserves stock, captures payment, and confirms the order atomically (saga pattern). The response includes a confirmed order with a human-readable reference.

**Testable:** `POST /checkout/guest/orders` with valid `X-Guest-Session-Token`, complete `shipping_address`, valid `shipping_method_id`, valid `payment_method` token, and at least one order `line` returns `201` with an `Order` containing `status: confirmed`, a non-empty `reference` (e.g. `ORD-20240415-A3K9`), and `lines` matching the submitted items.

---

### AC-015 — Stock conflict at checkout

When one or more SKUs have insufficient stock at the time of order placement, the order is rejected with details of which items are affected.

**Testable:** `POST /checkout/guest/orders` when requested quantities exceed available stock returns `409` with `code: STOCK_CONFLICT` and a `conflicts` array listing each affected `sku_id` with `requested` and `available` quantities. UI identifies which items could not be fulfilled and their available quantities.

---

### AC-016 — Payment failure releases reservations

When payment capture fails during the checkout saga, all stock reservations are released and no order record is created. The guest sees an error.

**Testable:** When payment capture fails after stock has been reserved, order-service calls `POST /stock/reservations/{reservationId}/release` on inventory-service within 30 seconds. No `Order` record is persisted. UI displays an error indicating payment could not be processed.

---

## Test Scenarios (verbatim from TS-001)

### TS-001-021 — Successful order returns 201 with confirmed order
- **Preconditions:** order-service running. Valid guest session. inventory-service mock: `POST /stock/reserve` → 200 with `reservation_id` and `expires_at`. Payment gateway mock: capture succeeds. inventory-service mock: `POST /stock/deduct` → 200. notification-service mock: `POST /notifications` → 202.
- **Action:** `POST /checkout/guest/orders` with headers `X-Guest-Session-Token`, `X-Tenant-ID` and body: `{ "email": "grace@example.com", "shipping_address": { "line1": "123 Main St", "city": "London", "postal_code": "SW1A 1AA", "country_code": "GB" }, "shipping_method_id": "{validMethodId}", "payment_method": { "type": "card", "token": "tok_visa_test" }, "lines": [{ "sku_id": "{skuId}", "quantity": 2 }] }`.
- **Expected:** HTTP 201. Response matches `Order` schema: `status: "confirmed"`, `reference` non-empty (pattern like `ORD-XXXXXXXX-XXXX`), `lines` array has 1 entry matching submitted SKU with `quantity: 2`. `total` is a positive integer. `shipping_address` matches submitted address. `created_at` is recent.

### TS-001-022 — Saga executes steps in correct order: reserve → capture → deduct
- **Preconditions:** order-service running. inventory-service mock: `POST /stock/reserve` → 200 with `reservation_id`. Payment gateway mock: capture succeeds. inventory-service mock: `POST /stock/deduct` → 200. notification-service mock: `POST /notifications` → 202.
- **Action:** `POST /checkout/guest/orders` with valid request body.
- **Expected:** inventory-service `POST /stock/reserve` called exactly once with `order_id` (UUID) and `lines` matching submitted items. Payment gateway called exactly once after reserve. inventory-service `POST /stock/deduct` called exactly once with `reservation_id` after payment. notification-service `POST /notifications` called exactly once after order confirmed. Call order: reserve → capture → deduct → notify (sequential).

### TS-001-023 — Order reference is human-readable and unique
- **Preconditions:** order-service running. Two successful orders placed in sequence.
- **Action:** Place two orders via `POST /checkout/guest/orders` with valid payloads.
- **Expected:** Both orders return `reference` fields that are non-empty strings. References match human-readable format (e.g., `ORD-20260411-XXXX`). The two references are different from each other.

### TS-001-027 — Insufficient stock returns 409 with conflict details
- **Preconditions:** order-service running. Valid guest session. inventory-service mock: `POST /stock/reserve` → 409 with `StockConflictError`: `code: "STOCK_CONFLICT"`, `conflicts: [{ sku_id: "{skuId}", requested: 5, available: 2 }]`.
- **Action:** `POST /checkout/guest/orders` with valid body where `lines` includes `{ sku_id: "{skuId}", quantity: 5 }`.
- **Expected:** HTTP 409. Response has `code: "STOCK_CONFLICT"` and `conflicts` array with one entry: `sku_id`, `requested: 5`, `available: 2`.

### TS-001-028 — Stock conflict prevents payment capture
- **Preconditions:** order-service running. Valid guest session. inventory-service mock: `POST /stock/reserve` → 409 (stock conflict). Payment gateway mock configured.
- **Action:** `POST /checkout/guest/orders` with valid body requesting more than available stock.
- **Expected:** Payment gateway mock called 0 times. No `Order` record persisted. inventory-service `POST /stock/deduct` called 0 times.

### TS-001-029 — UI displays stock conflict with affected items
- **Preconditions:** storefront-app loaded, guest session active, cart has items. Mock: `POST /checkout/guest/orders` → 409 with conflicts.
- **Action:** User submits checkout form.
- **Expected:** *(FE scenario — included for traceability only. For this BE WP, focus is on returning the 409 response with conflict details, which is covered by TS-001-027.)*

### TS-001-030 — Payment failure triggers stock reservation release
- **Preconditions:** order-service running. Valid guest session. inventory-service mock: `POST /stock/reserve` → 200 with `reservation_id: "{resId}"`. Payment gateway mock: capture fails (error/declined). inventory-service mock: `POST /stock/reservations/{resId}/release` → 204.
- **Action:** `POST /checkout/guest/orders` with valid body.
- **Expected:** inventory-service `POST /stock/reservations/{resId}/release` called exactly once within 30 seconds. `reservationId` in release call matches the one from reserve response. HTTP response to client indicates payment failure (not 201).

### TS-001-031 — Payment failure does not persist an order
- **Preconditions:** Same as TS-001-030.
- **Action:** `POST /checkout/guest/orders` with valid body (payment will fail).
- **Expected:** No `Order` record exists in the database. inventory-service `POST /stock/deduct` called 0 times.

### TS-001-059 — order-service GET /health returns 200
- **Preconditions:** order-service is running.
- **Action:** `GET /health`.
- **Expected:** HTTP 200. Response body: `{"status": "ok"}`.

### TS-001-060 — order-service GET /ready returns 200
- **Preconditions:** order-service is running. Database connection is healthy.
- **Action:** `GET /ready`.
- **Expected:** HTTP 200. Response body contains `"status": "ready"`.

### TS-001-061 — order-service GET /metrics returns Prometheus format
- **Preconditions:** order-service is running.
- **Action:** `GET /metrics`.
- **Expected:** HTTP 200. `Content-Type` header contains `text/plain`. Response body contains Prometheus-formatted metrics.

---

## Relevant Contracts

### order-service.openapi.yaml (excerpts)

#### POST /checkout/guest/orders

```yaml
path: /checkout/guest/orders
method: POST
operationId: placeGuestOrder
tags: [Checkout]
description: >
  Validates the cart, reserves stock, creates a Payment Intent, and
  places the order atomically. Requires a valid guest session token.
  Idempotent when the same Idempotency-Key header is provided.
security:
  - guestSession: []    # X-Guest-Session-Token header
parameters:
  - name: Idempotency-Key
    in: header
    required: false
    schema:
      type: string
      maxLength: 64
  - name: X-Tenant-ID
    in: header
    required: true
    schema:
      type: string
      format: uuid
responses:
  201:
    description: Order placed successfully.
    schema: Order
  400: ErrorResponse
  401:
    description: Guest session token is missing, invalid, or expired.
    schema: ErrorResponse
  409:
    description: Conflict — one or more items are out of stock.
    schema: StockConflictError
  422: ErrorResponse
```

#### PlaceGuestOrderRequest schema

```yaml
PlaceGuestOrderRequest:
  type: object
  required: [email, shipping_address, shipping_method_id, payment_method, lines]
  properties:
    email:
      type: string
      format: email
    shipping_address:
      $ref: Address
    shipping_method_id:
      type: string
      format: uuid
    payment_method:
      $ref: PaymentMethodInput
    lines:
      type: array
      minItems: 1
      items:
        type: object
        required: [sku_id, quantity]
        properties:
          sku_id:
            type: string
            format: uuid
          quantity:
            type: integer
            minimum: 1
```

#### Order schema

```yaml
Order:
  type: object
  required: [id, reference, status, lines, total, created_at]
  properties:
    id:
      type: string
      format: uuid
    reference:
      type: string
      example: "ORD-20240415-A3K9"
    status:
      $ref: OrderStatus    # enum: [pending, confirmed, processing, shipped, delivered, cancelled]
    guest_email:
      type: string
      format: email
      nullable: true
    lines:
      type: array
      items:
        $ref: OrderLine
    subtotal:
      type: integer          # Minor currency units
    shipping_cost:
      type: integer
    tax:
      type: integer
    total:
      type: integer          # Minor currency units
    shipping_address:
      $ref: Address
    notification_id:
      type: string
      format: uuid
      nullable: true
    notification_status:
      type: string
      enum: [queued, sent, delivered, failed]
      nullable: true
    created_at:
      type: string
      format: date-time
```

#### OrderLine schema

```yaml
OrderLine:
  type: object
  required: [sku_id, quantity, unit_price, subtotal]
  properties:
    sku_id:
      type: string
      format: uuid
    product_name:
      type: string
    variant_label:
      type: string
    quantity:
      type: integer
      minimum: 1
    unit_price:
      type: integer          # Minor currency units at time of order
    subtotal:
      type: integer
```

#### OrderStatus enum

```yaml
OrderStatus:
  type: string
  enum: [pending, confirmed, processing, shipped, delivered, cancelled]
```

#### StockConflictError schema

```yaml
StockConflictError:
  type: object
  required: [code, message, conflicts]
  properties:
    code:
      type: string
      example: "STOCK_CONFLICT"
    message:
      type: string
    conflicts:
      type: array
      items:
        type: object
        properties:
          sku_id:
            type: string
            format: uuid
          requested:
            type: integer
          available:
            type: integer
```

#### Address schema

```yaml
Address:
  type: object
  required: [line1, city, postal_code, country_code]
  properties:
    line1:
      type: string
      maxLength: 100
    line2:
      type: string
      maxLength: 100
    city:
      type: string
      maxLength: 100
    state:
      type: string
      maxLength: 100
    postal_code:
      type: string
      maxLength: 20
    country_code:
      type: string
      minLength: 2
      maxLength: 2
      description: ISO 3166-1 alpha-2 country code.
```

#### PaymentMethodInput schema

```yaml
PaymentMethodInput:
  type: object
  required: [type, token]
  properties:
    type:
      type: string
      enum: [card, digital_wallet]
    token:
      type: string
      description: Tokenised payment instrument from the payment gateway SDK.
```

#### ErrorResponse schema

```yaml
ErrorResponse:
  type: object
  required: [code, message]
  properties:
    code:
      type: string
    message:
      type: string
    details:
      type: array
      items:
        type: object
        properties:
          field:
            type: string
          issue:
            type: string
```

### inventory-service.openapi.yaml (excerpts)

#### POST /stock/reserve

```yaml
path: /stock/reserve
method: POST
operationId: reserveStock
description: >
  Places a temporary hold on the requested quantities. Called by the
  Order Service during checkout. Reservations expire after 15 minutes
  if not converted to a deduction.
parameters:
  - name: X-Tenant-ID (required, UUID)
  - name: Idempotency-Key (optional, string max 64)
requestBody:
  schema: ReserveStockRequest
responses:
  200:
    description: Reservations created for all requested SKUs.
    schema: ReserveStockResponse
  409:
    description: One or more SKUs have insufficient stock.
    schema: StockConflictError
```

#### POST /stock/deduct

```yaml
path: /stock/deduct
method: POST
operationId: deductStock
description: Convert reservations to permanent deductions on order confirmation.
parameters:
  - name: X-Tenant-ID (required, UUID)
  - name: Idempotency-Key (optional, string max 64)
requestBody:
  schema: DeductStockRequest
responses:
  200:
    description: Stock deducted successfully.
  409:
    description: Reservation not found or already expired.
```

#### POST /stock/reservations/{reservationId}/release

```yaml
path: /stock/reservations/{reservationId}/release
method: POST
operationId: releaseReservation
description: >
  Saga compensation step. Called by order-service when payment fails
  after stock has been reserved. Decrements the reserved counter and
  deletes the reservation record.
parameters:
  - name: X-Tenant-ID (required, UUID)
  - name: reservationId (path, required, UUID)
responses:
  204:
    description: Reservation released successfully.
  404:
    description: Reservation not found (already released or never existed — treat as success).
```

#### ReserveStockRequest schema

```yaml
ReserveStockRequest:
  type: object
  required: [order_id, lines]
  properties:
    order_id:
      type: string
      format: uuid
      description: The pending order ID this reservation is for.
    lines:
      type: array
      minItems: 1
      items:
        type: object
        required: [sku_id, quantity]
        properties:
          sku_id:
            type: string
            format: uuid
          quantity:
            type: integer
            minimum: 1
```

#### ReserveStockResponse schema

```yaml
ReserveStockResponse:
  type: object
  required: [reservation_id, expires_at]
  properties:
    reservation_id:
      type: string
      format: uuid
    expires_at:
      type: string
      format: date-time
```

#### DeductStockRequest schema

```yaml
DeductStockRequest:
  type: object
  required: [reservation_id]
  properties:
    reservation_id:
      type: string
      format: uuid
```

#### StockConflictError schema (inventory-service)

```yaml
StockConflictError:
  type: object
  required: [code, message, conflicts]
  properties:
    code:
      type: string
      example: "STOCK_CONFLICT"
    message:
      type: string
    conflicts:
      type: array
      items:
        type: object
        properties:
          sku_id:
            type: string
            format: uuid
          requested:
            type: integer
          available:
            type: integer
```

### order.schema.md (key constraints)

#### Table: `orders`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `tenant_id` | `UUID` | NO | Foreign key → tenants. Scopes the order to a merchant. |
| `reference` | `VARCHAR(30)` | NO | Human-readable order reference (e.g. `ORD-20240415-A3K9`). Unique per tenant. |
| `status` | `ENUM` | NO | `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`. |
| `customer_id` | `UUID` | YES | Foreign key → customers. NULL for guest orders. |
| `guest_email` | `VARCHAR(254)` | YES | Email provided during guest checkout. NULL for authenticated orders. |
| `shipping_address` | `JSONB` | NO | Snapshot of the shipping address at time of order. |
| `shipping_method_id` | `UUID` | NO | Foreign key → shipping_methods. |
| `shipping_cost_minor` | `INT` | NO | Shipping cost in minor currency units at time of order. |
| `subtotal_minor` | `INT` | NO | Sum of all order line subtotals in minor currency units. |
| `tax_minor` | `INT` | NO | Tax amount in minor currency units. |
| `total_minor` | `INT` | NO | Grand total: subtotal + shipping + tax. |
| `notification_id` | `UUID` | YES | ID of the order confirmation notification. NULL until notification dispatched. |
| `notification_status` | `VARCHAR(20)` | YES | Cached delivery status (`queued`, `sent`, `delivered`, `failed`). NULL until dispatched. |
| `payment_intent_id` | `VARCHAR(100)` | YES | External payment gateway intent ID. NULL until payment step. |
| `idempotency_key` | `VARCHAR(64)` | YES | Client-supplied idempotency key for safe retries. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp of order creation. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp of last status update. |

**Constraints:**
- `UNIQUE (tenant_id, reference)`
- `CHECK (customer_id IS NOT NULL OR guest_email IS NOT NULL)` — every order has either a customer or a guest email.
- `UNIQUE (tenant_id, idempotency_key)` — prevents duplicate order placement.

**Indexes:**
```sql
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);
CREATE INDEX idx_orders_guest_email ON orders (guest_email) WHERE guest_email IS NOT NULL;
```

#### Table: `order_lines`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `order_id` | `UUID` | NO | Foreign key → orders. |
| `sku_id` | `UUID` | NO | SKU ID snapshotted from inventory-service at order time. |
| `product_name` | `VARCHAR(200)` | NO | Product name snapshot. |
| `variant_label` | `VARCHAR(200)` | YES | Variant description snapshot (e.g. "Blue / XL"). |
| `quantity` | `INT` | NO | Units ordered. Minimum 1. |
| `unit_price_minor` | `INT` | NO | Price per unit in minor currency units at time of order. |
| `subtotal_minor` | `INT` | NO | `quantity × unit_price_minor`. |

**Constraints:**
- `quantity >= 1`
- `unit_price_minor >= 0`

**Indexes:**
```sql
CREATE INDEX idx_order_lines_order_id ON order_lines (order_id);
```

---

## Implementation Notes

### Project structure

```
workspaces/order-service/
  src/
    config.py                   # Settings: DB URL, INVENTORY_SERVICE_URL, STRIPE_API_KEY, NOTIFICATION_SERVICE_URL
    main.py                     # FastAPI app setup, include routers
    dependencies.py             # Dependency injection (DB session, clients, guest session validation)
    models/
      __init__.py
      order.py                  # Order + OrderLine ORM models
      guest_session.py          # GuestSession ORM model (from WP-002-BE)
      shipping_method.py        # ShippingMethod ORM model (from WP-002-BE)
    schemas/
      __init__.py
      order.py                  # PlaceGuestOrderRequest, Order, OrderLine, OrderPage Pydantic schemas
      common.py                 # Address, PaymentMethodInput, ErrorResponse, StockConflictError
    repositories/
      order_repo.py             # Order + OrderLine data access
    services/
      order_saga.py             # Orchestrate: validate → reserve → capture → deduct → persist
      reference_generator.py    # Generate ORD-YYYYMMDD-XXXX references
    clients/
      inventory_client.py       # HTTP client for inventory-service (reserve, deduct, release)
      stripe_client.py          # Stripe SDK wrapper (PaymentIntent.create)
      notification_client.py    # HTTP client for notification-service (placeholder for WP-004-BE)
    api/
      health.py                 # GET /health, /ready, /metrics
      v1/
        checkout.py             # POST /checkout/guest/sessions (WP-002-BE), POST /checkout/guest/orders
        shipping.py             # GET /checkout/shipping-methods (WP-002-BE)
  tests/
    conftest.py                 # Shared fixtures (DB, TestClient, respx mocks, tenant, guest session)
    integration/
      test_order_placement.py   # TS-001-021, TS-001-022, TS-001-023
      test_stock_conflict.py    # TS-001-027, TS-001-028
      test_payment_failure.py   # TS-001-030, TS-001-031
      test_health.py            # TS-001-059, TS-001-060, TS-001-061
  alembic/
    versions/
      003_orders_table.py       # orders table
      004_order_lines_table.py  # order_lines table
  Dockerfile
  docker-compose.yml            # PostgreSQL, order-service
  pyproject.toml
```

### Tech stack

- Python 3.12+, FastAPI, SQLAlchemy 2.x (async), Alembic
- pytest + pytest-asyncio + httpx (TestClient) + respx (HTTP mocking)
- Pydantic v2 for request/response schemas
- Stripe Python SDK (`stripe`) in test mode
- `httpx` for external service calls (inventory-service, notification-service)

### Key patterns

**Saga flow (order placement):**

```
1. Validate request (fields, guest session, shipping method exists)
2. Look up current prices from inventory-service for each SKU (resolve prices at checkout time)
3. POST /stock/reserve on inventory-service with order lines
   └── On 409 → return 409 StockConflictError to client (saga stops)
4. Stripe PaymentIntent.create with the payment token
   └── On failure → POST /stock/reservations/{id}/release on inventory-service → return error to client
5. POST /stock/deduct on inventory-service with reservation_id
6. Persist Order + OrderLines to database (status: confirmed)
7. Generate human-readable reference: ORD-YYYYMMDD-XXXX
8. Return 201 with Order response
Note: notification dispatch (step after order confirmation) is added in WP-004-BE.
```

**External HTTP clients:**
- `InventoryClient`: wraps `httpx.AsyncClient` with base URL from `INVENTORY_SERVICE_URL` env var. Methods: `reserve_stock(order_id, lines)`, `deduct_stock(reservation_id)`, `release_reservation(reservation_id)`.
- `StripeClient`: wraps `stripe.PaymentIntent.create()`. Uses `STRIPE_API_KEY` env var. Test mode with `tok_visa` for test tokens.
- All clients are injected via FastAPI `Depends()` for easy mocking.

**Test mocking pattern:**
- Use `respx` to mock inventory-service HTTP calls. Set up mock routes in `conftest.py`.
- For Stripe, mock `stripe.PaymentIntent.create` via `unittest.mock.patch` or use Stripe's test mode tokens.
- Assert call counts and call order via respx route assertions.

**Reference generator:**
- Format: `ORD-YYYYMMDD-XXXX` where `XXXX` is 4 alphanumeric characters (uppercase).
- Use `datetime.date.today().strftime("%Y%m%d")` + `secrets.token_hex(2).upper()[:4]`.
- Check uniqueness against the database (retry on collision).

**Observability endpoints:**
- `GET /health` → `{"status": "ok"}` (always 200 if app is running)
- `GET /ready` → `{"status": "ready"}` (200 if DB connection is healthy; 503 otherwise)
- `GET /metrics` → Prometheus text format (use `prometheus_client` library)

### Key constraints

- `X-Tenant-ID` required on all endpoints.
- `X-Guest-Session-Token` required on `POST /checkout/guest/orders` — validated via the dependency from WP-002-BE.
- Prices are resolved at checkout time by looking up current SKU prices from inventory-service. Cart-displayed prices may differ from final order prices.
- `notification_id` and `notification_status` are set to NULL on initial order creation (WP-004-BE adds notification dispatch).
- `tax_minor` is `0` for MVP (no tax calculation engine). Set to `0` on every order.
- No PII in logs: do not log guest email, payment tokens, or shipping addresses.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-021, TS-001-022, TS-001-023, TS-001-027, TS-001-028, TS-001-030, TS-001-031, TS-001-059, TS-001-060, TS-001-061)
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (endpoint responses match OpenAPI schemas)

---

## Implementation Order

Implement in this order:
1. Scaffold order-service workspace (pyproject.toml, Dockerfile, docker-compose.yml with PostgreSQL)
2. DB models (`orders`, `order_lines` tables) + Alembic migrations
3. External HTTP clients (inventory-service client, Stripe client) with configurable base URLs
4. Order repository (data access layer for orders + order lines)
5. Reference generator (`ORD-YYYYMMDD-XXXX`)
6. Saga service (orchestrate: validate → reserve → capture → deduct → persist)
7. API endpoint: `POST /checkout/guest/orders`
8. Health/ready/metrics endpoints (`GET /health`, `GET /ready`, `GET /metrics`)
9. Tests (all TS scenarios, using `respx` for inventory-service mocks and `mock`/test mode for Stripe)
10. Run DoD checklist

---

## Dependency Info

**Wave:** 3
**Depends on:** WP-002-BE (guest session validation, shipping methods lookup must exist in order-service).
**Depended on by:** WP-004-BE (adds validation layer and notification dispatch on top of this saga).
