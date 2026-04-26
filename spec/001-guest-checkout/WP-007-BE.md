# WP-007-BE: Order List API — Backend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/order-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the admin order list endpoint in order-service: paginated order listing scoped to tenant, free-text search by order reference or guest email, status filtering, and empty state handling. This WP delivers the `GET /orders` endpoint with query parameters (`status`, `q`, `page`, `per_page`), a repository layer with tenant-scoped queries, and integration with the auth guard from WP-006-BE.

All external service calls (none — order list is a read-only query within order-service) are internal. The auth guard dependency from WP-006-BE is reused here.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-025 — Paginated order list

An authenticated admin user sees a paginated list of orders scoped to their tenant.

**Testable:** `GET /orders` with valid admin JWT and `X-Tenant-ID` returns `200` with an `OrderPage` containing `data` (array of `Order` objects) and `meta` (`total`, `page`, `per_page`). UI renders an order table with columns for reference, status, customer email, date, and total.

### AC-026 — Order search

The admin can search orders by order reference or customer email.

**Testable:** `GET /orders?q={searchTerm}` *(search parameter to be added to contract)* returns orders where `reference` or `guest_email` contains the search term. UI: the admin types in a search box and the order list updates to show matching results.

### AC-027 — Order status filter

The admin can filter the order list by order status.

**Testable:** `GET /orders?status=confirmed` returns only orders with `status: confirmed`. UI: the admin selects a status from a dropdown and the list updates to show only matching orders.

### AC-030 — Empty order list

When no orders exist or no orders match the current search/filter criteria, an informative empty state is shown.

**Testable:** When `GET /orders` returns `200` with an empty `data` array, the UI displays "No orders found" with guidance to adjust filters or wait for incoming orders.

---

## Test Scenarios (verbatim from TS-001)

### TS-001-046 — Paginated order list returns orders with metadata
- **Preconditions:** order-service running; 25 orders exist for the test tenant; admin user has fully authenticated JWT (viewer, admin, or owner with MFA complete)
- **Action:** `GET /orders?page=1&per_page=20` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; response matches `OrderPage` schema: `data` has 20 `Order` objects, `meta.total` is 25, `meta.page` is 1, `meta.per_page` is 20; each order has `id`, `reference`, `status`, `total`, `created_at`

### TS-001-047 — Orders are scoped to the authenticated tenant
- **Preconditions:** order-service running; Tenant A has 5 orders, Tenant B has 3 orders; admin JWT is for Tenant A
- **Action:** `GET /orders` with headers `Authorization: Bearer {tenantAJwt}` and `X-Tenant-ID: {tenantAId}`
- **Expected:** HTTP 200; `data` contains exactly 5 orders (all from Tenant A); `meta.total` is 5; no orders from Tenant B appear in the response

### TS-001-048 — Search by order reference returns matching orders
- **Preconditions:** order-service running; tenant has orders including one with `reference: "ORD-20260411-A3K9"`; admin JWT fully authenticated
- **Action:** `GET /orders?q=A3K9` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` contains the order with reference "ORD-20260411-A3K9"; orders not matching the search term are excluded

### TS-001-049 — Search by guest email returns matching orders
- **Preconditions:** order-service running; tenant has orders including 2 placed by `grace@example.com`; admin JWT fully authenticated
- **Action:** `GET /orders?q=grace@example.com` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` contains exactly 2 orders where `guest_email` matches; `meta.total` is 2

### TS-001-050 — Status filter returns only matching orders
- **Preconditions:** order-service running; tenant has 10 orders: 6 with `status: confirmed`, 4 with `status: pending`; admin JWT fully authenticated
- **Action:** `GET /orders?status=confirmed` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` contains exactly 6 orders, all with `status: "confirmed"`; `meta.total` is 6

### TS-001-054 — Empty order list returns empty data array
- **Preconditions:** order-service running; no orders exist for the test tenant (or current filters match nothing); admin JWT fully authenticated
- **Action:** `GET /orders` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; `data` is an empty array `[]`; `meta.total` is 0

---

## Relevant Contracts

### order-service.openapi.yaml (excerpts)

#### Security Schemes

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
```

#### GET /orders

```yaml
/orders:
  get:
    operationId: listOrders
    summary: List orders (merchant / platform roles only)
    tags: [Orders]
    security:
      - bearerAuth: []
    parameters:
      - name: X-Tenant-ID
        in: header
        required: true
        schema:
          type: string
          format: uuid
      - name: status
        in: query
        schema:
          $ref: "#/components/schemas/OrderStatus"
      - name: q
        in: query
        description: >
          Free-text search across order reference and guest email.
          Returns orders where `reference` or `guest_email` contains the
          search term (case-insensitive).
        schema:
          type: string
          maxLength: 200
      - name: page
        in: query
        schema:
          type: integer
          default: 1
      - name: per_page
        in: query
        schema:
          type: integer
          default: 20
          maximum: 100
    responses:
      "200":
        description: Paginated list of orders.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/OrderPage"
      "401":
        description: Missing or invalid authentication token.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
      "403":
        description: Insufficient permissions (e.g., pre-MFA JWT).
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
```

#### Key Schemas

```yaml
OrderPage:
  type: object
  required: [data, meta]
  properties:
    data:
      type: array
      items:
        $ref: "#/components/schemas/Order"
    meta:
      type: object
      properties:
        total:
          type: integer
        page:
          type: integer
        per_page:
          type: integer

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
      $ref: "#/components/schemas/OrderStatus"
    guest_email:
      type: string
      format: email
      nullable: true
    lines:
      type: array
      items:
        $ref: "#/components/schemas/OrderLine"
    subtotal:
      type: integer
    shipping_cost:
      type: integer
    tax:
      type: integer
    total:
      type: integer
      description: Order total in minor currency units.
    shipping_address:
      $ref: "#/components/schemas/Address"
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
      type: integer
    subtotal:
      type: integer

OrderStatus:
  type: string
  enum: [pending, confirmed, processing, shipped, delivered, cancelled]

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
| `notification_id` | `UUID` | YES | ID of the order confirmation notification in notification-service. |
| `notification_status` | `VARCHAR(20)` | YES | Cached delivery status (`queued`, `sent`, `delivered`, `failed`). |
| `payment_intent_id` | `VARCHAR(100)` | YES | External payment gateway intent ID. |
| `idempotency_key` | `VARCHAR(64)` | YES | Client-supplied idempotency key for safe retries. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp of order creation. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp of last status update. |

**Constraints:**
- `UNIQUE (tenant_id, reference)`
- `CHECK (customer_id IS NOT NULL OR guest_email IS NOT NULL)`
- `UNIQUE (tenant_id, idempotency_key)`

#### Table: `order_lines`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `order_id` | `UUID` | NO | Foreign key → orders. |
| `sku_id` | `UUID` | NO | SKU ID snapshotted from inventory-service. |
| `product_name` | `VARCHAR(200)` | NO | Product name snapshot. |
| `variant_label` | `VARCHAR(200)` | YES | Variant description snapshot. |
| `quantity` | `INT` | NO | Units ordered. Minimum 1. |
| `unit_price_minor` | `INT` | NO | Price per unit in minor currency units. |
| `subtotal_minor` | `INT` | NO | `quantity × unit_price_minor`. |

#### Indexes

```sql
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);
CREATE INDEX idx_orders_guest_email ON orders (guest_email) WHERE guest_email IS NOT NULL;
CREATE INDEX idx_order_lines_order_id ON order_lines (order_id);
```

---

## Implementation Notes

### Project structure

```
workspaces/order-service/
  src/
    models/
      order.py                  # Order + OrderLine ORM models (from WP-003-BE — already exists)
    schemas/
      order.py                  # OrderResponse, OrderLineResponse, OrderPage Pydantic schemas
    repositories/
      order_repo.py             # Order list query with search, filter, pagination, tenant scoping
    api/
      v1/
        orders.py               # GET /orders endpoint
    dependencies.py             # Auth guard (from WP-006-BE — already exists)
  tests/
    conftest.py                 # Shared fixtures (test DB, seeded orders)
    integration/
      test_order_list.py        # All TS-001-046 through TS-001-050 + TS-001-054 scenarios
```

### Tech stack

- **Language:** Python 3.12+
- **Framework:** FastAPI
- **ORM:** SQLAlchemy 2.x (async)
- **Migrations:** Alembic
- **Tests:** pytest + pytest-asyncio + httpx (AsyncClient)

### Key patterns

#### Tenant scoping

**Every query** in the order repository MUST filter by `tenant_id`. The tenant ID comes from the auth guard's `AdminContext.tenant_id`. Never allow cross-tenant data leakage.

```python
async def list_orders(
    db: AsyncSession,
    tenant_id: UUID,
    status: str | None = None,
    q: str | None = None,
    page: int = 1,
    per_page: int = 20,
) -> tuple[list[Order], int]:
    query = select(Order).where(Order.tenant_id == tenant_id)
    
    if status:
        query = query.where(Order.status == status)
    
    if q:
        search_term = f"%{q}%"
        query = query.where(
            or_(
                Order.reference.ilike(search_term),
                Order.guest_email.ilike(search_term),
            )
        )
    
    # Count total before pagination
    count_query = select(func.count()).select_from(query.subquery())
    total = await db.scalar(count_query)
    
    # Apply pagination
    query = query.order_by(Order.created_at.desc())
    query = query.offset((page - 1) * per_page).limit(per_page)
    
    # Eager load order lines
    query = query.options(selectinload(Order.lines))
    
    result = await db.execute(query)
    orders = result.scalars().all()
    
    return orders, total
```

#### Search (`q` parameter)

- Case-insensitive `ILIKE` on `orders.reference` and `orders.guest_email`.
- Returns the **union** of matches (OR condition).
- Combined with tenant scoping (AND condition with `tenant_id`).
- `maxLength: 200` enforced at the Pydantic schema level.

#### Status filter

- Exact match on `orders.status` enum value.
- Valid values: `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`.
- If an invalid status is provided, return 422 (validated via Pydantic/enum).

#### Pagination

- `page`: 1-based (default: 1).
- `per_page`: default 20, maximum 100. Clamp at schema validation level.
- Response `meta` includes `total` (total matching records), `page`, `per_page`.
- Offset-based: `offset = (page - 1) * per_page`.
- Default sort: `created_at DESC` (most recent first).

#### Order response with lines

- Eager load `order_lines` relationship using SQLAlchemy `selectinload` to avoid N+1 queries.
- Map DB column names to API response field names:
  - `total_minor` → `total`
  - `subtotal_minor` → `subtotal`
  - `shipping_cost_minor` → `shipping_cost`
  - `tax_minor` → `tax`
  - `unit_price_minor` → `unit_price` (on OrderLine)
  - `subtotal_minor` → `subtotal` (on OrderLine)
- Include `guest_email` and `notification_status` in the Order response.

#### Auth guard integration

- `GET /orders` uses `Depends(require_admin_auth)` from WP-006-BE.
- The guard verifies JWT validity, checks `mfa_verified`, and returns `AdminContext`.
- The endpoint extracts `tenant_id` from `AdminContext` for query scoping.

### Key constraints

- **Tenant isolation is non-negotiable.** Every query filters by `tenant_id`. The test TS-001-047 explicitly verifies this.
- **Empty results are 200, not 404.** `GET /orders` always returns 200 with `data: []` when no results match. Per AC-030.
- **No PII in logs.** Do not log `guest_email` values. Log order IDs and counts only.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios (TS-001-046, TS-001-047, TS-001-048, TS-001-049, TS-001-050, TS-001-054) have passing automated tests
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (Schemathesis against `GET /orders`)

---

## Implementation Order

Implement in this order:

1. **Order list repository method:** `order_repo.py` — query with search (`ILIKE` on reference + guest_email), status filter, pagination (offset-based), tenant scoping, eager load order lines
2. **Pydantic response schemas:** `OrderResponse`, `OrderLineResponse`, `OrderPageResponse` with `meta` (total, page, per_page)
3. **API endpoint: GET /orders:** `api/v1/orders.py` — wire query parameters (status, q, page, per_page), auth guard dependency, return `OrderPage`
4. **Test fixtures:** Seed orders for two tenants with varying statuses, references, and guest emails
5. **Tests:** All TS scenarios (TS-001-046, TS-001-047, TS-001-048, TS-001-049, TS-001-050, TS-001-054) as integration tests
6. **Run DoD checklist:** ruff, mypy, contract validation

---

## Dependencies

- **Wave:** 3
- **Depends on:** WP-003-BE (orders table + order data model), WP-006-BE (auth guard dependency)
- **Depended on by:** WP-008-FE (admin order list UI)
