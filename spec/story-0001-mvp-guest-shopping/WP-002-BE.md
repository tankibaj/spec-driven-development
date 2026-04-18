# WP-002-BE: MVP Guest Shopping — Guest Sessions + Shipping Methods

**Feature:** story-0001-mvp-guest-shopping
**Target workspace:** `workspaces/order-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the guest checkout infrastructure in order-service: guest session creation and validation, shipping methods listing, and session expiry enforcement. This WP delivers the foundation that WP-003-BE (order placement saga) builds on.

All external service calls (none in this WP — self-contained within order-service) are made via configurable HTTP clients so they can be mocked in tests.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-010 — Guest session creation

The system creates a guest checkout session when the guest begins checkout. The session token is used for all subsequent checkout API calls.

**Testable:** `POST /checkout/guest/sessions` returns `201` with a `GuestSession` containing `id` (UUID), `token` (opaque string), and `expires_at` (future timestamp).

---

### AC-011 — Checkout form

The guest completes a multi-step checkout form: (1) shipping address with required fields (line1, city, postal_code, country_code), (2) shipping method selection from available options, and (3) payment details.

**Testable:** When the guest proceeds to checkout, then a multi-step form is presented with shipping address fields, a shipping method selector showing available options with costs, and a payment details form. All required fields must be completed before the order can be submitted.

> **Note:** AC-011 has both BE and FE components. This WP covers the BE component: the `GET /checkout/shipping-methods` endpoint that returns available shipping options. The checkout form UI is in WP-004-FE.

---

### AC-017 — Expired guest session rejected

An expired or invalid guest session token cannot be used for checkout.

**Testable:** `POST /checkout/guest/orders` with an expired or invalid `X-Guest-Session-Token` returns `401` *(to be added to contract — not currently specified in order-service.openapi.yaml)*. UI redirects the guest to restart the checkout process.

---

## Test Scenarios (verbatim from TS-001)

### TS-001-016 — POST /checkout/guest/sessions creates a guest session
- **Preconditions:** order-service is running and healthy.
- **Action:** `POST /checkout/guest/sessions` with header `X-Tenant-ID: {tenantId}` and empty JSON body `{}`.
- **Expected:** HTTP 201. Response matches `GuestSession` schema: `id` (UUID), `token` (non-empty string), `expires_at` (ISO 8601 datetime). `expires_at` is in the future (at least 23 hours from now).

### TS-001-017 — Guest session token is usable in subsequent requests
- **Preconditions:** order-service is running. A guest session was created and the `token` was captured.
- **Action:** `POST /checkout/guest/orders` with header `X-Guest-Session-Token: {capturedToken}` and a valid request body.
- **Expected:** The request is NOT rejected with 401 (session token is accepted). The request proceeds to validation (may return 201, 409, or 422 depending on payload — the point is the session token is valid).

### TS-001-018 — Shipping methods returned from API
- **Preconditions:** order-service is running. `shipping_methods` table is seeded with at least 2 methods (e.g., "Standard Shipping" $5.99, "Express Shipping" $14.99).
- **Action:** `GET /checkout/shipping-methods` with header `X-Tenant-ID: {tenantId}`.
- **Expected:** HTTP 200. Response is an array of shipping method objects, each with `id` (UUID), `name` (string), `description` (string), `cost_minor` (integer), and `estimated_days` (string or integer range). At least 2 methods returned.

### TS-001-019 — Checkout form renders multi-step UI with shipping methods
- **Preconditions:** storefront-app is loaded, guest session created, cart has at least one item. Mock: `GET /checkout/shipping-methods` returns 2 shipping options.
- **Action:** User navigates to checkout.
- **Expected:** *(FE scenario — included for traceability only. Not tested in this BE WP.)*

### TS-001-020 — Checkout form enforces required fields
- **Preconditions:** storefront-app is loaded, guest session active, cart has items, user is on shipping address step.
- **Action:** User leaves required fields empty and attempts to proceed.
- **Expected:** *(FE scenario — included for traceability only. Not tested in this BE WP.)*

### TS-001-032 — Expired session token returns 401
- **Preconditions:** order-service is running. A guest session was created but has expired (either by time or by test manipulation).
- **Action:** `POST /checkout/guest/orders` with header `X-Guest-Session-Token: {expiredToken}` and valid body.
- **Expected:** HTTP 401. Response indicates session has expired. No stock reservation attempted, no order created.

### TS-001-033 — Invalid session token returns 401
- **Preconditions:** order-service is running.
- **Action:** `POST /checkout/guest/orders` with header `X-Guest-Session-Token: invalid-garbage-token` and valid body.
- **Expected:** HTTP 401. Response indicates invalid session. No stock reservation attempted, no order created.

---

## Relevant Contracts

### order-service.openapi.yaml (excerpts)

#### POST /checkout/guest/sessions

```yaml
path: /checkout/guest/sessions
method: POST
operationId: createGuestSession
tags: [Checkout]
description: >
  Initialises a new Guest Session. Returns a session token to be included
  in subsequent checkout requests. The session expires after 24 hours of
  inactivity.
requestBody:
  required: false
  content:
    application/json:
      schema:
        type: object
        properties:
          cart_id:
            type: string
            format: uuid
            description: ID of an existing anonymous cart to attach to this session.
responses:
  201:
    schema: GuestSession
  400: ErrorResponse
  422: ErrorResponse
```

#### GET /checkout/shipping-methods

```yaml
path: /checkout/shipping-methods
method: GET
operationId: listShippingMethods
tags: [Checkout]
description: >
  Returns the available shipping methods for the tenant. Used by the
  storefront during checkout to populate the shipping method selector.
parameters:
  - name: X-Tenant-ID
    in: header
    required: true
    schema:
      type: string
      format: uuid
responses:
  200:
    schema:
      type: array
      items: ShippingMethod
```

#### POST /checkout/guest/orders (401 response only — full endpoint in WP-003-BE)

```yaml
path: /checkout/guest/orders
method: POST
security:
  - guestSession: []    # X-Guest-Session-Token header
parameters:
  - name: X-Tenant-ID (required, UUID)
  - name: Idempotency-Key (optional, string max 64)
responses:
  401:
    description: Guest session token is missing, invalid, or expired.
    schema: ErrorResponse
```

#### GuestSession schema

```yaml
GuestSession:
  type: object
  required: [id, token, expires_at]
  properties:
    id:
      type: string
      format: uuid
    token:
      type: string
      description: Opaque token to include in X-Guest-Session-Token.
    expires_at:
      type: string
      format: date-time
```

#### ShippingMethod schema

```yaml
ShippingMethod:
  type: object
  required: [id, name, cost_minor]
  properties:
    id:
      type: string
      format: uuid
    name:
      type: string
      example: "Standard Shipping"
    description:
      type: string
      example: "Delivery in 3-5 business days"
    cost_minor:
      type: integer
      description: Shipping cost in minor currency units (e.g. cents).
      example: 599
    estimated_days_min:
      type: integer
      description: Minimum estimated delivery days.
      example: 3
    estimated_days_max:
      type: integer
      description: Maximum estimated delivery days.
      example: 5
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

#### Security schemes

```yaml
securitySchemes:
  guestSession:
    type: apiKey
    in: header
    name: X-Guest-Session-Token
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

### order.schema.md (key constraints)

#### Table: `shipping_methods`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `tenant_id` | `UUID` | NO | Foreign key → tenants. Scopes shipping methods to a merchant. |
| `name` | `VARCHAR(100)` | NO | Display name (e.g. "Standard Shipping"). |
| `description` | `VARCHAR(500)` | YES | Human-readable description (e.g. "Delivery in 3-5 business days"). |
| `cost_minor` | `INT` | NO | Shipping cost in minor currency units. |
| `estimated_days_min` | `INT` | YES | Minimum estimated delivery days. |
| `estimated_days_max` | `INT` | YES | Maximum estimated delivery days. |
| `is_active` | `BOOLEAN` | NO | Whether this method is available for selection. Default: `true`. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp. |

**Constraints:**
- `UNIQUE (tenant_id, name)`
- `cost_minor >= 0`

**Notes:**
- The `orders.shipping_method_id` column references `shipping_methods.id`.
- Shipping methods are seeded via migration fixture. No public management UI in MVP.

#### Table: `guest_sessions` (designed for this WP)

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `tenant_id` | `UUID` | NO | Scopes the session to a merchant. |
| `token` | `VARCHAR(255)` | NO | Opaque session token (random string or UUID). Unique. |
| `expires_at` | `TIMESTAMPTZ` | NO | Expiry time. Sessions expire after 24 hours of inactivity. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp. |

**Constraints:**
- `UNIQUE (token)` — token lookup must be fast and unambiguous.
- Token is opaque: use `secrets.token_urlsafe(32)` or `uuid4()` string.

**Indexes:**
```sql
CREATE UNIQUE INDEX idx_guest_sessions_token ON guest_sessions (token);
CREATE INDEX idx_guest_sessions_expires ON guest_sessions (expires_at);
```

---

## Implementation Notes

### Project structure

```
workspaces/order-service/
  src/
    config.py                   # Settings and env vars
    main.py                     # FastAPI app setup
    dependencies.py             # Dependency injection
    models/
      guest_session.py          # GuestSession ORM model
      shipping_method.py        # ShippingMethod ORM model
    schemas/
      guest_session.py          # Pydantic request/response schemas
      shipping_method.py        # Pydantic response schema
    repositories/
      guest_session_repo.py     # Data access for guest sessions
      shipping_method_repo.py   # Data access for shipping methods
    services/
      guest_session_service.py  # Create, validate, expire sessions
    api/
      v1/
        checkout.py             # POST /checkout/guest/sessions, GET /checkout/shipping-methods
  tests/
    conftest.py                 # Shared fixtures (DB, client, tenant)
    integration/
      test_guest_sessions.py    # TS-001-016, TS-001-017, TS-001-032, TS-001-033
      test_shipping_methods.py  # TS-001-018
  alembic/
    versions/
      001_guest_sessions.py     # guest_sessions table
      002_shipping_methods.py   # shipping_methods table + seed data
```

### Tech stack

- Python 3.12+, FastAPI, SQLAlchemy 2.x (async), Alembic
- pytest + pytest-asyncio + httpx (TestClient)
- Pydantic v2 for request/response schemas

### Key patterns

**Guest session creation:**
1. Generate a unique token via `secrets.token_urlsafe(32)`.
2. Set `expires_at` to `now + 24 hours`.
3. Persist to `guest_sessions` table.
4. Return `GuestSession` response with `id`, `token`, `expires_at`.

**Session validation (FastAPI dependency):**
1. Extract `X-Guest-Session-Token` header.
2. Look up the session by `token` in the `guest_sessions` table.
3. If not found → raise `HTTPException(401)` with `{"code": "INVALID_SESSION", "message": "Invalid session token"}`.
4. If `expires_at < now` → raise `HTTPException(401)` with `{"code": "SESSION_EXPIRED", "message": "Guest session has expired"}`.
5. Return the valid `GuestSession` model for downstream use.

This dependency will be used by `POST /checkout/guest/orders` in WP-003-BE.

**Shipping methods seed data (Alembic migration):**
Insert two rows per tenant (use a seed tenant ID from test fixtures):
- "Standard Shipping", description "Delivery in 3-5 business days", `cost_minor: 599`, `estimated_days_min: 3`, `estimated_days_max: 5`, `is_active: true`
- "Express Shipping", description "Delivery in 1-2 business days", `cost_minor: 1499`, `estimated_days_min: 1`, `estimated_days_max: 2`, `is_active: true`

**Shipping methods listing:**
`GET /checkout/shipping-methods` returns all active methods (`is_active = true`) for the tenant identified by `X-Tenant-ID`.

### Key constraints

- `X-Tenant-ID` header is required on all endpoints. Validate its presence and format (UUID).
- Guest session tokens must be cryptographically random and unguessable.
- Session expiry is checked at validation time — no background cleanup needed for MVP (expired sessions are simply rejected).
- No PII in logs: do not log session tokens.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-016, TS-001-017, TS-001-018, TS-001-032, TS-001-033)
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (endpoint responses match OpenAPI schemas)
- [ ] Note: `/health`, `/ready`, `/metrics` endpoints are NOT in scope for this WP — they are implemented in WP-003-BE which scaffolds the order-service workspace

---

## Implementation Order

Implement in this order:
1. DB models (`guest_sessions` table, `shipping_methods` table) + Alembic migrations
2. Shipping methods seed data (Standard $5.99, Express $14.99)
3. Guest session service (create, validate, expire)
4. Pydantic schemas (`GuestSession`, `ShippingMethod` responses)
5. API endpoints: `POST /checkout/guest/sessions`, `GET /checkout/shipping-methods`
6. Session validation FastAPI dependency for `X-Guest-Session-Token`
7. Tests for all TS scenarios (TS-001-016, TS-001-017, TS-001-018, TS-001-032, TS-001-033)
8. Run DoD checklist

---

## Dependency Info

**Wave:** 2
**Depends on:** WP-001-BE (inventory-service) being available for integration.
**Depended on by:** WP-003-BE (order placement saga) — uses guest session validation and shipping method lookup from this WP.
