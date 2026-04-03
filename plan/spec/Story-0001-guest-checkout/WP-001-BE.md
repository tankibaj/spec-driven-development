# WP-001-BE: Guest Checkout — Backend

**Feature:** Story-0001-guest-checkout
**Derived from:** FS-001, TS-001
**Target workspace:** `workspaces/order-service`
**Status:** Approved
**Last updated:** 2026-04-03

---

## Objective

Implement the backend API for guest checkout in the `order-service`. This covers:

1. Guest session creation and management.
2. Order placement with stock reservation, payment capture, and notification dispatch.
3. Idempotent request handling.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (copied verbatim from FS-001)

**AC-001 — Guest session creation**
A shopper who has not signed in can initiate a Guest Checkout session without providing any personal details upfront. `POST /checkout/guest/sessions` returns a session token and expiry timestamp.

**AC-004 — Payment entry**
The system must not store raw card numbers. A valid card token from the payment gateway SDK is passed to the order placement API. Raw card data never appears in any service log or database.

**AC-005 — Order placement — happy path**
When the shopper submits valid shipping and payment details, the system:
1. Validates the cart is non-empty and all items are in stock.
2. Reserves stock for all cart items.
3. Creates a Payment Intent and captures the payment.
4. Creates an Order record with status `confirmed`.
5. Sends an order confirmation email to the provided guest email address.
6. Returns the Order Reference to the storefront.

**AC-006 — Out-of-stock handling**
If any item in the cart is out of stock at the moment of order placement, the system MUST: not charge the shopper, not create an Order record, return HTTP 409 with `code: STOCK_CONFLICT` listing the conflicting SKUs.

**AC-007 — Idempotent order placement**
A retry with the same `Idempotency-Key` header MUST return the original response without creating a duplicate order or charging the card twice.

**AC-008 — Order confirmation email**
After successful order placement, call the Notification Service with `template_id: order_confirmation` and order details. The notification must dispatch within 2 minutes.

---

## Test Scenarios (copied verbatim from TS-001)

**TS-001-001** — `POST /checkout/guest/sessions` with empty body → HTTP 201, response includes `id`, `token`, `expires_at` (~24 h from now).

**TS-001-002** — `POST /checkout/guest/sessions` with valid `cart_id` → HTTP 201, session associated with cart.

**TS-001-003** — `POST /checkout/guest/sessions` with non-existent `cart_id` → HTTP 422.

**TS-001-010** — Valid card tokenised; token (not raw card data) is sent to `POST /checkout/guest/orders`; service logs and orders DB do NOT contain card number or CVV.

**TS-001-012** — Successful `POST /checkout/guest/orders` → HTTP 201, `status: "confirmed"`, `reference` matches `ORD-\d{8}-[A-Z0-9]{4}`, order record in DB, stock reduced, payment captured.

**TS-001-013** — Order total calculation: 2×€10.00 + 1×€25.00 + €4.99 shipping + 20% tax = €58.99 total.

**TS-001-014** — SKU-A qty 2 requested, available 1 → HTTP 409, `code: STOCK_CONFLICT`, `conflicts[0].sku_id = SKU-A, requested: 2, available: 1`, no order, no charge.

**TS-001-015** — Both SKUs available 0 → HTTP 409, both listed in `conflicts`.

**TS-001-017** — Two identical requests with `Idempotency-Key: abc-123` → both HTTP 201, identical responses, exactly one order in DB, one payment capture.

**TS-001-018** — Same payload, different idempotency key → two distinct orders created.

**TS-001-019** — After successful order, Notification Service called within 2 minutes with `template_id: order_confirmation`, correct payload.

**TS-001-020** — After out-of-stock failure, Notification Service is NOT called.

---

## API Contract

Implement the following endpoints exactly as defined in `contracts/api/order-service.openapi.yaml`:

| Method | Path | Operation ID |
|---|---|---|
| `POST` | `/checkout/guest/sessions` | `createGuestSession` |
| `POST` | `/checkout/guest/orders` | `placeGuestOrder` |

### Guest Session

Request (optional):
```json
{ "cart_id": "uuid" }
```

Response (201):
```json
{
  "id": "uuid",
  "token": "opaque-string",
  "expires_at": "2026-04-04T21:48:00Z"
}
```

### Place Guest Order

Request (required headers: `X-Guest-Session-Token`, `X-Tenant-ID`; optional: `Idempotency-Key`):
```json
{
  "email": "grace@example.com",
  "shipping_address": {
    "line1": "12 High Street",
    "city": "London",
    "postal_code": "EC1A 1BB",
    "country_code": "GB"
  },
  "shipping_method_id": "uuid",
  "payment_method": {
    "type": "card",
    "token": "tok_gateway_xyz"
  }
}
```

Success response (201):
```json
{
  "id": "uuid",
  "reference": "ORD-20260403-A3K9",
  "status": "confirmed",
  "lines": [...],
  "subtotal": 4500,
  "shipping_cost": 499,
  "tax": 900,
  "total": 5899,
  "shipping_address": {...},
  "created_at": "2026-04-03T21:48:00Z"
}
```

Conflict response (409):
```json
{
  "code": "STOCK_CONFLICT",
  "message": "One or more items are out of stock.",
  "conflicts": [
    { "sku_id": "uuid", "requested": 2, "available": 1 }
  ]
}
```

---

## Implementation Notes

### Guest Session

- Store sessions in Redis with key `guest_session:{id}` and TTL 86400 seconds (24 h).
- Session value: JSON `{ tenant_id, cart_id?, created_at }`.
- Generate `token` as a cryptographically random 32-byte value encoded as URL-safe base64.
- Validate the session token on every checkout request via middleware. Return 401 if missing or expired.

### Order Placement — Saga Pattern

Use a local Saga to coordinate the following steps atomically. If any step fails, compensate by releasing reservations and voiding the Payment Intent before returning the error to the client.

```
1. Validate cart (non-empty, all SKUs belong to the tenant)
2. Call inventory-service POST /stock/reserve  ← compensatable
3. Call payment gateway: Create Payment Intent
4. Call payment gateway: Capture Payment Intent ← compensatable (void)
5. Create Order record in DB (status: confirmed)
6. Deduct stock: inventory-service POST /stock/deduct
7. Call notification-service POST /notifications (async, fire-and-forget)
8. Return Order to client
```

If step 2 returns 409 (stock conflict), return 409 to the client immediately — do not proceed.
If step 4 fails (payment declined), release the reservation (call `POST /stock/reservations/{id}/release` — add to inventory-service if not present) and return a 402 or 422.

### Idempotency

- Store idempotency key in the `orders` table: `UNIQUE (tenant_id, idempotency_key)`.
- On receiving a request with an `Idempotency-Key` header:
  - Look up the key in the DB.
  - If a matching row exists, return the stored response immediately without re-running the saga.
  - If not, proceed with the saga and store the key alongside the new order.

### Reference Generation

Generate Order Reference as `ORD-YYYYMMDD-{4 random uppercase alphanumeric chars}`. Verify uniqueness per tenant before persisting. Retry up to 3 times on collision.

### Security — No Raw Card Data

- Never log, store, or pass card numbers, CVVs, or full PANs.
- The `payment_method.token` field is the only payment-related value that touches the order-service.
- Ensure structured logging frameworks are configured to mask the `payment_method` field in request logs.

### Notification Dispatch

Call the Notification Service asynchronously after the order record is committed. Use a background task / message queue so that a notification delivery failure does NOT roll back the order. Log failures for alerting.

Notification payload for `order_confirmation`:
```json
{
  "order_reference": "ORD-20260403-A3K9",
  "items": [
    { "name": "Widget Pro", "variant": "Blue / XL", "qty": 2, "unit_price": "€10.00", "subtotal": "€20.00" }
  ],
  "subtotal": "€45.00",
  "shipping_cost": "€4.99",
  "tax": "€9.00",
  "total": "€58.99",
  "shipping_address": { "line1": "...", ... },
  "estimated_delivery": "3–5 business days"
}
```

### Error Handling Conventions

All 4xx error responses MUST use the `ErrorResponse` schema:
```json
{ "code": "SNAKE_CASE_CODE", "message": "Human-readable message" }
```

---

## Related Contracts

- `contracts/api/order-service.openapi.yaml` — full OpenAPI spec for the endpoints above.
- `contracts/api/inventory-service.openapi.yaml` — `reserveStock`, `deductStock` operations.
- `contracts/api/notification-service.openapi.yaml` — `sendNotification` operation.
- `contracts/architecture/ADR-001.md` — guest session design decision (Redis, opaque token).
- `contracts/data-schema/order.schema.md` — `orders` and `order_lines` table schemas.
