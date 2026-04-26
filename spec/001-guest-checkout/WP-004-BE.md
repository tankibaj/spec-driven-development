# WP-004-BE: MVP Guest Shopping — Checkout Validation + Post-Order Notification

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/order-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Add input validation to the order placement endpoint (422 responses with field-level errors) and implement the post-order notification step (call notification-service after successful order confirmation). This WP builds on top of WP-003-BE — the saga endpoint already exists; this WP adds the validation layer that runs BEFORE the saga and the notification dispatch that runs AFTER it.

All external service calls (notification-service) are made via configurable HTTP clients so they can be mocked in tests.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-014 — Checkout validation errors

Missing or invalid fields in the checkout form produce clear, field-level error messages.

**Testable:** `POST /checkout/guest/orders` with missing required fields (e.g. missing `postal_code` in `shipping_address`) returns `422` with a `details` array containing `field` and `issue` entries. UI displays field-level validation messages on the corresponding form fields.

---

### AC-018 — Confirmation email dispatched

After successful order placement, the order-service sends an order confirmation email via the notification-service.

**Testable:** After `POST /checkout/guest/orders` returns `201`, order-service calls `POST /notifications` on notification-service with `channel: email`, `template_id: order_confirmation`, and `recipient.address` set to the guest's email address, within 30 seconds of order confirmation.

---

### AC-019 — Email contains order details

The confirmation email includes the order reference, line items, and order total so the guest can verify their order.

**Testable:** The `POST /notifications` request `payload` object includes `order_reference` (string matching the order's `reference`), line item details (product name, quantity, unit price per line), and `total` (formatted currency string).

---

## Test Scenarios (verbatim from TS-001)

### TS-001-025 — Missing required field returns 422 with field-level error
- **Preconditions:** order-service running. Valid guest session exists.
- **Action:** `POST /checkout/guest/orders` with header `X-Guest-Session-Token`, `X-Tenant-ID` and body missing `postal_code` in `shipping_address`: `{ "email": "grace@example.com", "shipping_address": { "line1": "123 Main St", "city": "London", "country_code": "GB" }, "shipping_method_id": "{validMethodId}", "payment_method": { "type": "card", "token": "tok_visa_test" }, "lines": [{ "sku_id": "{skuId}", "quantity": 1 }] }`.
- **Expected:** HTTP 422. Response body contains `details` array with at least one entry where `field` contains "postal_code" and `issue` describes the missing field. No stock reservation was attempted (inventory-service mock `POST /stock/reserve` called 0 times).

### TS-001-026 — Multiple validation errors returned in single response
- **Preconditions:** order-service running. Valid guest session exists.
- **Action:** `POST /checkout/guest/orders` with body missing `email`, `shipping_address.line1`, and `shipping_address.postal_code`.
- **Expected:** HTTP 422. Response `details` array contains at least 3 entries, covering each missing field. No stock reservation was attempted.

### TS-001-034 — Order-service calls notification-service after successful order
- **Preconditions:** order-service running. All saga mocks succeed (stock reserve, payment capture, stock deduct). notification-service mock: `POST /notifications` → 202 with receipt.
- **Action:** `POST /checkout/guest/orders` with valid body including `email: "grace@example.com"`.
- **Expected:** notification-service mock `POST /notifications` called exactly once. Request has `channel: "email"`, `template_id: "order_confirmation"`, `recipient.address: "grace@example.com"`. Call occurs after order confirmation (after stock deduct succeeds). Order returned with `status: confirmed` regardless of notification result.

### TS-001-035 — Notification not sent when order placement fails
- **Preconditions:** order-service running. inventory-service mock: `POST /stock/reserve` → 409 (stock conflict). notification-service mock configured.
- **Action:** `POST /checkout/guest/orders` with valid body.
- **Expected:** notification-service mock `POST /notifications` called 0 times. No notification dispatched because the order was never confirmed.

### TS-001-036 — Notification payload includes order reference, items, and total
- **Preconditions:** order-service running. All saga mocks succeed. notification-service mock: `POST /notifications` → 202. Order placed with 2 line items.
- **Action:** `POST /checkout/guest/orders` with valid body containing 2 order lines.
- **Expected:** notification-service mock received `POST /notifications` with `payload` containing: `order_reference` matching the `reference` in the order response (e.g., `"ORD-20260411-A3K9"`), line item details (product name and quantity for each line), `total` (formatted currency string matching the order total).

---

## Relevant Contracts

### order-service.openapi.yaml (excerpts)

#### POST /checkout/guest/orders — 422 response

```yaml
path: /checkout/guest/orders
method: POST
# Full endpoint defined in WP-003-BE. This WP adds the validation layer.
responses:
  422:
    description: The request is well-formed but contains semantic errors.
    content:
      application/json:
        schema:
          $ref: ErrorResponse
```

#### ErrorResponse schema (with details array)

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
            description: >
              Dot-notation path to the invalid field
              (e.g. "shipping_address.postal_code", "email").
          issue:
            type: string
            description: >
              Human-readable description of the validation error
              (e.g. "This field is required").
```

#### PlaceGuestOrderRequest schema (required fields reference)

```yaml
PlaceGuestOrderRequest:
  type: object
  required: [email, shipping_address, shipping_method_id, payment_method, lines]
  properties:
    email:
      type: string
      format: email
    shipping_address:
      $ref: Address     # required: [line1, city, postal_code, country_code]
    shipping_method_id:
      type: string
      format: uuid
    payment_method:
      $ref: PaymentMethodInput    # required: [type, token]
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

PaymentMethodInput:
  type: object
  required: [type, token]
  properties:
    type:
      type: string
      enum: [card, digital_wallet]
    token:
      type: string
```

#### Order schema (notification fields)

```yaml
Order:
  type: object
  required: [id, reference, status, lines, total, created_at]
  properties:
    # ... (full schema in WP-003-BE)
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
```

### notification-service.openapi.yaml (excerpts)

#### POST /notifications

```yaml
path: /notifications
method: POST
operationId: sendNotification
description: >
  Dispatches a notification to the specified recipient using the
  chosen channel and template. Idempotent when the same
  Idempotency-Key is provided.
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
requestBody:
  required: true
  content:
    application/json:
      schema:
        $ref: SendNotificationRequest
responses:
  202:
    description: Notification accepted for delivery.
    schema: NotificationReceipt
  400: ErrorResponse
  422: ErrorResponse
```

#### SendNotificationRequest schema

```yaml
SendNotificationRequest:
  type: object
  required: [channel, template_id, recipient, payload]
  properties:
    channel:
      type: string
      enum: [email, sms]
    template_id:
      type: string
      description: >
        Identifier of the notification template to use.
        For order confirmation: "order_confirmation"
      example: order_confirmation
    recipient:
      type: object
      required: [address]
      properties:
        address:
          type: string
          description: Email address for email channel.
        name:
          type: string
          description: Optional display name (used in email salutation).
    payload:
      type: object
      additionalProperties: true
      description: >
        Template variable values. For order_confirmation:
        {
          "order_reference": "ORD-20240415-A3K9",
          "lines": [{"product_name": "...", "quantity": 2, "unit_price": "$29.99"}],
          "total": "$74.97"
        }
```

#### NotificationReceipt schema

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

### order.schema.md (notification columns on orders table)

The `orders` table includes these notification-related columns (full table definition in WP-003-BE):

| Column | Type | Nullable | Description |
|---|---|---|---|
| `notification_id` | `UUID` | YES | ID of the order confirmation notification in notification-service. NULL until notification dispatched. |
| `notification_status` | `VARCHAR(20)` | YES | Cached delivery status of the confirmation notification (`queued`, `sent`, `delivered`, `failed`). NULL until notification dispatched. Synced from notification-service. |

After successful notification dispatch, update the order record:
- Set `notification_id` to the `id` from the `NotificationReceipt` response.
- Set `notification_status` to the `status` from the `NotificationReceipt` response (typically `"queued"`).

---

## Implementation Notes

### Project structure

This WP adds/modifies files in the order-service workspace (scaffolded by WP-003-BE):

```
workspaces/order-service/
  src/
    schemas/
      order.py                  # Add validation logic to PlaceGuestOrderRequest
    services/
      order_saga.py             # Add validation step BEFORE saga, notification step AFTER saga
      validation.py             # (NEW) Request validation: check all required fields, return 422 details
    clients/
      notification_client.py    # (NEW) HTTP client for notification-service
  tests/
    integration/
      test_validation.py        # (NEW) TS-001-025, TS-001-026
      test_notification.py      # (NEW) TS-001-034, TS-001-035, TS-001-036
```

### Tech stack

- Python 3.12+, FastAPI, SQLAlchemy 2.x (async), Alembic
- pytest + pytest-asyncio + httpx (TestClient) + respx (HTTP mocking)
- Pydantic v2 for request/response schemas
- `httpx` for notification-service HTTP calls

### Key patterns

**Request validation (422 responses):**

The validation layer runs BEFORE the saga starts. It checks:

1. `email` — required, must be valid email format
2. `shipping_address.line1` — required
3. `shipping_address.city` — required
4. `shipping_address.postal_code` — required
5. `shipping_address.country_code` — required, must be 2 characters
6. `shipping_method_id` — required, must be a valid UUID
7. `payment_method.type` — required, must be `"card"` or `"digital_wallet"`
8. `payment_method.token` — required, non-empty
9. `lines` — required, must have at least 1 item
10. Each line: `sku_id` required (UUID), `quantity` required (>= 1)

If any validation fails, return 422 with ALL errors collected (not fail-fast):

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request contains invalid fields",
  "details": [
    { "field": "email", "issue": "This field is required" },
    { "field": "shipping_address.postal_code", "issue": "This field is required" },
    { "field": "shipping_address.line1", "issue": "This field is required" }
  ]
}
```

Implementation options:
- **Option A (Recommended):** Use Pydantic model validation with a custom exception handler. Catch `ValidationError`, transform into the `ErrorResponse` format with `details` array. Pydantic already collects all errors.
- **Option B:** Write manual validation in the service layer. More control but more boilerplate.

The key requirement from TS-001-025 and TS-001-026 is that **no stock reservation is attempted** when validation fails. This means validation must happen BEFORE calling inventory-service.

**Post-order notification dispatch:**

After the saga completes successfully (order status = `confirmed`):

1. Build `SendNotificationRequest`:
   ```json
   {
     "channel": "email",
     "template_id": "order_confirmation",
     "recipient": {
       "address": "{guest_email}"
     },
     "payload": {
       "order_reference": "ORD-20260411-A3K9",
       "lines": [
         { "product_name": "Classic T-Shirt", "quantity": 2, "unit_price": "$29.99" }
       ],
       "total": "$74.97"
     }
   }
   ```
2. Call `POST /notifications` on notification-service with `X-Tenant-ID` header.
3. On 202 response: extract `id` from `NotificationReceipt`, update order record: `notification_id = receipt.id`, `notification_status = receipt.status`.
4. On failure (network error, 4xx, 5xx): log the error, set `notification_id = NULL`, `notification_status = NULL`. **Do NOT affect the order.** The order is still returned as `confirmed` to the client.

**Notification failure handling:**
- Notification failure must NOT affect the order response. The client always gets the confirmed order.
- Log the notification failure at WARNING level for operational visibility.
- The `notification_id` being NULL on the order record indicates notification was not dispatched.

**External HTTP client for notification-service:**
- Use `httpx.AsyncClient` with base URL from `NOTIFICATION_SERVICE_URL` env var.
- Timeout: 10 seconds (notification is non-critical).
- Wrap in try/except: catch `httpx.HTTPError` and log, do not re-raise.

### Key constraints

- Validation runs BEFORE any external calls (stock, payment). This is the critical assertion in TS-001-025 and TS-001-026.
- Notification dispatch must not block or fail the order response.
- No PII in logs: do not log guest email addresses or payment tokens. Log `notification_id` and `order_reference` for correlation.
- The `notification_status` field on the Order API response will show the status from the initial dispatch (`"queued"` typically). Polling for final status (sent/delivered/failed) is out of scope for this WP.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-025, TS-001-026, TS-001-034, TS-001-035, TS-001-036)
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints still return expected responses (no regression)
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (endpoint responses match OpenAPI schemas)

---

## Implementation Order

Implement in this order:
1. Request validation layer (Pydantic custom exception handler or manual validation) returning 422 with field `details` array
2. Notification-service HTTP client (`NotificationClient` using `httpx` with `NOTIFICATION_SERVICE_URL`)
3. Post-order notification dispatch (integrated into saga flow, after order confirmation)
4. Store `notification_id` and `notification_status` on order record after successful dispatch
5. Tests for all TS scenarios (TS-001-025, TS-001-026, TS-001-034, TS-001-035, TS-001-036)
6. Run DoD checklist

---

## Dependency Info

**Wave:** 4
**Depends on:** WP-003-BE (order saga endpoint must exist — this WP adds validation and notification on top of it).
**Depended on by:** No BE WPs depend on this directly. WP-005-BE (notification-service) must be deployed for real integration. WP-004-FE (storefront checkout UI) consumes the 422 validation response and displays field errors.
