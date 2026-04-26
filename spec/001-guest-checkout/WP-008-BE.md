# WP-008-BE: Order Detail API — Backend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/order-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the admin order detail endpoint in order-service and the notification status synchronization mechanism. Admin users can view full order details including line items, shipping address, and confirmation email delivery status. This WP delivers the `GET /orders/{orderId}` endpoint with tenant-scoped lookup, eager-loaded order lines, and a lazy-fetch notification status sync service that polls notification-service when a stale `notification_status` is detected.

All external service calls (notification-service `GET /notifications/{notificationId}`) are made via a configurable HTTP client so they can be mocked in tests.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-028 — Order detail view

The admin views full order details including order lines, shipping address, order status, and timestamps.

**Testable:** `GET /orders/{orderId}` with valid admin JWT returns `200` with `Order` containing `lines` (each with `product_name`, `variant_label`, `quantity`, `unit_price`), `shipping_address`, `status`, `reference`, and `created_at`. UI displays all fields in a structured layout.

### AC-029 — Notification status indicator

When the confirmation email for an order has failed delivery, the order detail page shows a warning indicator. When the email was delivered successfully, no indicator is shown.

**Testable:** When the notification associated with an order has `status: failed`, the order detail UI shows a "Confirmation email not delivered" warning. When `status` is `sent` or `delivered`, no warning is displayed.

### AC-031 — Order not found

Navigating to a non-existent order shows a not-found state.

**Testable:** `GET /orders/{nonExistentId}` returns `404`. UI displays "Order not found" with a link back to the order list.

---

## Test Scenarios (verbatim from TS-001)

### TS-001-051 — Order detail returns full order with all fields
- **Preconditions:** order-service running; an order exists with `id: {orderId}`, 2 line items, shipping address, status `confirmed`; admin JWT fully authenticated
- **Action:** `GET /orders/{orderId}` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; response matches `Order` schema with: `id`, `reference`, `status: "confirmed"`, `created_at`; `lines` array with 2 entries, each having `product_name`, `variant_label`, `quantity`, `unit_price`, `subtotal`; `shipping_address` with `line1`, `city`, `postal_code`, `country_code`; `total` (integer, minor currency units)

### TS-001-052 — Failed notification shows warning on order detail
- **Preconditions:** order-service running; order exists with `notification_status: "failed"` (order-service polled notification-service and stored the failed status); admin JWT fully authenticated
- **Action:** `GET /orders/{orderId}` with admin JWT
- **Expected (BE):** HTTP 200; order response includes `notification_status: "failed"`

### TS-001-053 — Successful notification shows no warning
- **Preconditions:** order-service running; order exists with `notification_status: "sent"` or `"delivered"`; admin JWT fully authenticated
- **Action:** `GET /orders/{orderId}` with admin JWT
- **Expected (BE):** HTTP 200; order response includes `notification_status: "sent"` (or `"delivered"`)

### TS-001-055 — Non-existent order returns 404
- **Preconditions:** order-service running; no order exists with `id: {nonExistentId}` for the test tenant; admin JWT fully authenticated
- **Action:** `GET /orders/{nonExistentId}` with headers `Authorization: Bearer {jwt}` and `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 404; response matches `ErrorResponse` schema with a descriptive `message`

---

## Relevant Contracts

### workspaces/order-service/docs/api/openapi.json (excerpts)

#### Security Schemes

```yaml
securitySchemes:
  bearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
```

#### GET /orders/{orderId}

```yaml
/orders/{orderId}:
  get:
    operationId: getOrder
    summary: Get an order by ID
    tags: [Orders]
    description: >
      Returns full order details. Accessible to the owning Customer,
      any merchant admin role, and platform ops.
    security:
      - bearerAuth: []
    parameters:
      - name: X-Tenant-ID
        in: header
        required: true
        schema:
          type: string
          format: uuid
      - name: orderId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    responses:
      "200":
        description: Order found.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/Order"
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
      "404":
        description: The requested resource does not exist.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
```

#### Key Schemas

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
      description: >
        ID of the confirmation notification sent via notification-service.
        Null if notification has not been dispatched yet.
    notification_status:
      type: string
      enum: [queued, sent, delivered, failed]
      nullable: true
      description: >
        Delivery status of the order confirmation notification, synced
        from notification-service. Null if no notification exists.
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
      description: Human-readable variant description (e.g. "Blue / XL").
    quantity:
      type: integer
      minimum: 1
    unit_price:
      type: integer
      description: Unit price in minor currency units at time of order.
    subtotal:
      type: integer

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

### workspaces/notification-service/docs/api/openapi.json (excerpts)

#### GET /notifications/{notificationId}

```yaml
/notifications/{notificationId}:
  get:
    operationId: getNotification
    summary: Get notification delivery status
    tags: [Notifications]
    parameters:
      - name: X-Tenant-ID
        in: header
        required: true
        schema:
          type: string
          format: uuid
      - name: notificationId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    responses:
      "200":
        description: Notification status returned.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/NotificationReceipt"
      "404":
        description: Resource not found.
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ErrorResponse"
```

#### NotificationReceipt Schema

```yaml
NotificationReceipt:
  type: object
  required: [id, status, created_at]
  properties:
    id:
      type: string
      format: uuid
    status:
      type: string
      enum: [queued, sent, delivered, failed]
    channel:
      type: string
    template_id:
      type: string
    created_at:
      type: string
      format: date-time
    delivered_at:
      type: string
      format: date-time
      nullable: true
```

### workspaces/order-service/docs/schema/entities.md (key constraints)

#### Table: `orders` (notification-related columns)

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `tenant_id` | `UUID` | NO | Foreign key → tenants. Scopes the order to a merchant. |
| `reference` | `VARCHAR(30)` | NO | Human-readable order reference. Unique per tenant. |
| `status` | `ENUM` | NO | `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`. |
| `guest_email` | `VARCHAR(254)` | YES | Email provided during guest checkout. |
| `shipping_address` | `JSONB` | NO | Snapshot of the shipping address at time of order. |
| `shipping_method_id` | `UUID` | NO | Foreign key → shipping_methods. |
| `shipping_cost_minor` | `INT` | NO | Shipping cost in minor currency units. |
| `subtotal_minor` | `INT` | NO | Sum of all order line subtotals. |
| `tax_minor` | `INT` | NO | Tax amount. |
| `total_minor` | `INT` | NO | Grand total: subtotal + shipping + tax. |
| `notification_id` | `UUID` | YES | ID of the confirmation notification in notification-service. NULL until notification dispatched. |
| `notification_status` | `VARCHAR(20)` | YES | Cached delivery status (`queued`, `sent`, `delivered`, `failed`). NULL until notification dispatched. Synced from notification-service. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp of order creation. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp of last status update. |

#### Table: `order_lines`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `order_id` | `UUID` | NO | Foreign key → orders. |
| `sku_id` | `UUID` | NO | SKU ID snapshotted from inventory-service. |
| `product_name` | `VARCHAR(200)` | NO | Product name snapshot. |
| `variant_label` | `VARCHAR(200)` | YES | Variant description snapshot (e.g. "Blue / XL"). |
| `quantity` | `INT` | NO | Units ordered. Minimum 1. |
| `unit_price_minor` | `INT` | NO | Price per unit in minor currency units at time of order. |
| `subtotal_minor` | `INT` | NO | `quantity × unit_price_minor`. |

**Constraints:**
- `quantity >= 1`
- `unit_price_minor >= 0`

#### Indexes

```sql
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);
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
      order.py                  # OrderResponse, OrderLineResponse, AddressResponse (from WP-007-BE — extend if needed)
    repositories/
      order_repo.py             # Order detail query (add get_by_id method)
    services/
      notification_sync.py      # Notification status lazy-fetch service (NEW)
    clients/
      notification_client.py    # HTTP client for notification-service (reuse from WP-004-BE or create)
    api/
      v1/
        orders.py               # GET /orders/{orderId} endpoint (add to existing router)
    dependencies.py             # Auth guard (from WP-006-BE — already exists)
  tests/
    conftest.py                 # Shared fixtures
    integration/
      test_order_detail.py      # TS-001-051, TS-001-052, TS-001-053, TS-001-055
```

### Tech stack

- **Language:** Python 3.12+
- **Framework:** FastAPI
- **ORM:** SQLAlchemy 2.x (async)
- **HTTP client:** httpx (async) for notification-service calls
- **Tests:** pytest + pytest-asyncio + httpx (AsyncClient) + respx (HTTP mocking)

### Key patterns

#### Order detail repository

```python
async def get_order_by_id(
    db: AsyncSession,
    order_id: UUID,
    tenant_id: UUID,
) -> Order | None:
    """
    Load a single order by ID, scoped to tenant.
    Eager-loads order lines.
    Returns None if not found.
    """
    query = (
        select(Order)
        .where(Order.id == order_id, Order.tenant_id == tenant_id)
        .options(selectinload(Order.lines))
    )
    result = await db.execute(query)
    return result.scalar_one_or_none()
```

- **Tenant scoping is mandatory.** Always filter by `tenant_id` — an order that exists for Tenant B must return 404 for Tenant A.
- **Eager load `order_lines`** using `selectinload` to avoid N+1 queries.

#### Notification status sync service (lazy fetch)

Per FS-001 resolved open question (Option A): order-service stores `notification_id` when sending the notification (done in WP-004-BE). When order detail is requested, order-service checks if `notification_status` needs refreshing and fetches from notification-service if needed.

```python
class NotificationSyncService:
    """
    Lazy-fetches notification status from notification-service
    and caches it on the order record.
    """
    
    def __init__(self, notification_client: NotificationClient, db: AsyncSession):
        self.client = notification_client
        self.db = db
    
    async def sync_notification_status(self, order: Order) -> Order:
        """
        If the order has a notification_id and notification_status is 'queued',
        fetch fresh status from notification-service, update the order record,
        and return the updated order.
        
        If notification_status is already terminal (sent, delivered, failed),
        skip the fetch — status won't change.
        
        If notification_id is None, do nothing.
        """
        if order.notification_id is None:
            return order
        
        if order.notification_status in ("sent", "delivered", "failed"):
            # Terminal status — no need to re-fetch
            return order
        
        # Status is 'queued' or None — fetch fresh status
        try:
            receipt = await self.client.get_notification(
                notification_id=order.notification_id,
                tenant_id=order.tenant_id,
            )
            order.notification_status = receipt.status
            self.db.add(order)
            await self.db.commit()
        except Exception:
            # If notification-service is unreachable, return stale status
            # Do not fail the order detail request
            pass
        
        return order
```

**Key rules:**
- Only fetch when `notification_status` is `queued` (or `None`). Terminal statuses (`sent`, `delivered`, `failed`) are never re-fetched.
- If notification-service is unreachable, return the stale cached status. Do not fail the order detail request — notification status is supplementary, not critical.
- Update the order record after fetching so subsequent requests use the cached value.

#### Notification HTTP client

```python
class NotificationClient:
    """
    HTTP client for notification-service.
    Base URL is configurable via NOTIFICATION_SERVICE_URL env var.
    """
    
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = httpx.AsyncClient(base_url=base_url, timeout=5.0)
    
    async def get_notification(
        self, notification_id: UUID, tenant_id: UUID
    ) -> NotificationReceipt:
        response = await self.client.get(
            f"/notifications/{notification_id}",
            headers={"X-Tenant-ID": str(tenant_id)},
        )
        response.raise_for_status()
        return NotificationReceipt(**response.json())
```

- Base URL configured via `NOTIFICATION_SERVICE_URL` environment variable.
- Timeout: 5 seconds (notification status is not critical — fail fast).
- Reuse the client from WP-004-BE if it already exists. If not, create it here.

#### API endpoint

```python
@router.get("/orders/{order_id}", response_model=OrderResponse)
async def get_order_detail(
    order_id: UUID,
    admin: AdminContext = Depends(require_admin_auth),
    db: AsyncSession = Depends(get_db),
    notification_sync: NotificationSyncService = Depends(get_notification_sync),
):
    order = await order_repo.get_order_by_id(db, order_id, admin.tenant_id)
    if order is None:
        raise HTTPException(
            status_code=404,
            detail={"code": "ORDER_NOT_FOUND", "message": "Order not found"},
        )
    
    # Lazy-fetch notification status if needed
    order = await notification_sync.sync_notification_status(order)
    
    return order
```

#### Response mapping

Map DB column names to API response field names:
- `total_minor` → `total`
- `subtotal_minor` → `subtotal`
- `shipping_cost_minor` → `shipping_cost`
- `tax_minor` → `tax`
- `unit_price_minor` → `unit_price` (on OrderLine)
- `subtotal_minor` → `subtotal` (on OrderLine)
- `shipping_address` (JSONB) → `Address` object with `line1`, `line2`, `city`, `state`, `postal_code`, `country_code`

Include in the response: `id`, `reference`, `status`, `guest_email`, `lines` (with `product_name`, `variant_label`, `quantity`, `unit_price`, `subtotal`), `shipping_address`, `subtotal`, `shipping_cost`, `tax`, `total`, `notification_id`, `notification_status`, `created_at`.

### Test mocking pattern

Use `respx` to mock notification-service HTTP calls in tests:

```python
@pytest.fixture
def mock_notification_service():
    with respx.mock:
        yield respx

# Example: mock a failed notification status
def test_failed_notification(mock_notification_service):
    mock_notification_service.get(
        f"http://notification-service/notifications/{notification_id}"
    ).respond(json={"id": str(notification_id), "status": "failed", "created_at": "..."})
```

For TS-001-052 and TS-001-053, the tests should:
- Seed an order with a known `notification_id` and `notification_status`.
- For the "already synced" case (status is terminal): verify the response includes the cached status without calling notification-service.
- For the "needs sync" case (status is `queued`): mock the notification-service response and verify the order's `notification_status` is updated.

### Key constraints

- **Tenant isolation is non-negotiable.** Order detail MUST filter by `tenant_id`. An order belonging to another tenant returns 404, not 403.
- **Notification sync must not break order detail.** If notification-service is down, return stale status. Never return 500 for a notification fetch failure.
- **No PII in logs.** Do not log `guest_email`, `shipping_address`, or notification recipient details. Log order ID and notification ID only.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios (TS-001-051, TS-001-052, TS-001-053, TS-001-055) have passing automated tests
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (Schemathesis against `GET /orders/{orderId}`)

---

## Implementation Order

Implement in this order:

1. **Order detail repository method:** `order_repo.py` — `get_order_by_id(db, order_id, tenant_id)` with eager-loaded order lines, tenant scoping
2. **Notification HTTP client:** `notification_client.py` — `get_notification(notification_id, tenant_id)` with configurable base URL and 5s timeout (reuse from WP-004-BE if available)
3. **Notification sync service:** `notification_sync.py` — lazy-fetch logic: skip terminal statuses, fetch and cache for `queued`, graceful failure handling
4. **API endpoint: GET /orders/{orderId}:** `api/v1/orders.py` — wire auth guard, tenant scoping, notification sync, 404 handling
5. **Tests:** All TS scenarios (TS-001-051, TS-001-052, TS-001-053, TS-001-055) with mock notification-service via respx; seed test orders with known notification statuses
6. **Run DoD checklist:** ruff, mypy, contract validation

---

## Dependencies

- **Wave:** 3 (parallel with WP-007-BE)
- **Depends on:** WP-003-BE (orders table + order data model), WP-006-BE (auth guard dependency), WP-004-BE (notification client — reuse if available)
- **Depended on by:** WP-009-FE (admin order detail UI)
