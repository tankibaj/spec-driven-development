# WP-006-FE: storefront-app — Checkout Error Handling

**Feature:** story-0001-mvp-guest-shopping
**Target workspace:** `workspaces/storefront-app`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Handle checkout error states in storefront-app: stock conflict (409 response with affected item details), payment failure (re-enable form for retry), and expired guest session (401 redirect to restart checkout). After this WP, all checkout failure modes degrade gracefully with clear user messaging.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-015 — Stock conflict at checkout

When one or more SKUs have insufficient stock at the time of order placement, the order is rejected with details of which items are affected.

**Testable:** `POST /checkout/guest/orders` when requested quantities exceed available stock returns `409` with `code: STOCK_CONFLICT` and a `conflicts` array listing each affected `sku_id` with `requested` and `available` quantities. UI identifies which items could not be fulfilled and their available quantities.

---

### AC-016 — Payment failure releases reservations

When payment capture fails during the checkout saga, all stock reservations are released and no order record is created. The guest sees an error.

**Testable:** When payment capture fails after stock has been reserved, order-service calls `POST /stock/reservations/{reservationId}/release` on inventory-service within 30 seconds. No `Order` record is persisted. UI displays an error indicating payment could not be processed.

---

### AC-017 — Expired guest session rejected

An expired or invalid guest session token cannot be used for checkout.

**Testable:** `POST /checkout/guest/orders` with an expired or invalid `X-Guest-Session-Token` returns `401` *(to be added to contract — not currently specified in order-service.openapi.yaml)*. UI redirects the guest to restart the checkout process.

---

## Test Scenarios (from TS-001)

### TS-001-029 — UI displays stock conflict with affected items
- **Preconditions:** storefront-app loaded. Guest session active. Cart has items. MSW mock: `POST /checkout/guest/orders` returns 409 with `{ code: "STOCK_CONFLICT", message: "Insufficient stock", conflicts: [{ sku_id: "sku-1", requested: 5, available: 2 }] }`.
- **Action:** User clicks "Place Order".
- **Expected:** Checkout page displays stock conflict error. The affected item is identified with message like "Only 2 units available (you requested 5)". User can navigate back to cart to adjust quantities.

### TS-001-030 — Payment failure triggers stock reservation release (FE aspect)
- **Preconditions:** storefront-app loaded. Checkout form completed. MSW mock: `POST /checkout/guest/orders` returns an error indicating payment failure (e.g., 502 or a custom error response with `code: "PAYMENT_FAILED"`).
- **Action:** User clicks "Place Order".
- **Expected:** UI displays "Payment could not be processed. Please try again or use a different payment method." "Place Order" button re-enabled. User can retry or change payment details.

### TS-001-032 — Expired session token returns 401 (FE aspect)
- **Preconditions:** storefront-app loaded. Checkout form completed. MSW mock: `POST /checkout/guest/orders` returns 401 with `{ code: "SESSION_EXPIRED", message: "Guest session has expired" }`.
- **Action:** User clicks "Place Order".
- **Expected:** UI displays "Your session has expired." User is redirected to restart checkout (navigate to `/cart`). Checkout state is reset.

### TS-001-033 — Invalid session token returns 401 (FE aspect)
- **Preconditions:** storefront-app loaded. Checkout form completed. MSW mock: `POST /checkout/guest/orders` returns 401 with `{ code: "INVALID_SESSION", message: "Invalid session token" }`.
- **Action:** User clicks "Place Order".
- **Expected:** UI displays "Your session has expired." User is redirected to `/cart`. Checkout state is reset.

---

## UI Flow

```
[Checkout: "Place Order"] → POST /checkout/guest/orders →
  409 (stock conflict):
    → [Stock Conflict Error View]
      → shows affected items + available quantities
      → "Return to Cart" button → /cart

  Payment failure:
    → [Payment Error Message]
      → "Payment could not be processed. Please try again or use a different payment method."
      → "Place Order" button re-enabled for retry

  401 (expired/invalid session):
    → [Session Expired Message]
      → "Your session has expired."
      → auto-redirect to /cart after brief display
```

### Stock Conflict View

When the API returns 409 with `StockConflictError`:

- Display a prominent error banner at the top of the checkout form.
- For each entry in the `conflicts` array, display:
  - Product identification — match `sku_id` from the conflict to the cart items to get `product_name` and `variant_label`.
  - Message: **"Only {available} units of {product_name} ({variant_label}) available (you requested {requested})."**
- Display a **"Return to Cart"** button that navigates to `/cart` so the user can adjust quantities.
- "Place Order" button remains visible but re-enabled (user might adjust and retry, though returning to cart is recommended).

### Payment Failure View

When the API returns an error indicating payment failure (the BE returns an error after the saga's payment step fails — stock reservations are released server-side):

- Display an inline error message: **"Payment could not be processed. Please try again or use a different payment method."**
- "Place Order" button is re-enabled.
- User stays on the payment step (Step 3).
- User can change card details in Stripe Elements and retry.

### Expired Session View

When the API returns 401:

- Display a message: **"Your session has expired."**
- Reset checkout state (`checkoutStore.resetCheckout()`).
- Navigate to `/cart` (the user will create a new guest session when they click "Checkout" again).

---

## Error Handling

### Stock conflict (409)

**When:** `POST /checkout/guest/orders` returns 409 with `StockConflictError`.
**Display:** Error banner listing each affected item: "Only {available} units of {product_name} ({variant_label}) available (you requested {requested})." "Return to Cart" button.
**Behavior:** "Place Order" button re-enabled. User can return to cart to adjust quantities and re-attempt checkout.

### Payment failure

**When:** `POST /checkout/guest/orders` returns an error indicating payment failure (e.g., response body contains `code: "PAYMENT_FAILED"`, or a non-201/non-409/non-422/non-401 error).
**Display:** "Payment could not be processed. Please try again or use a different payment method."
**Behavior:** User stays on Step 3 (Payment). "Place Order" button re-enabled. Stripe CardElement remains interactive for card changes.

### Expired or invalid guest session (401)

**When:** `POST /checkout/guest/orders` returns 401.
**Display:** "Your session has expired." Brief display (toast or banner), then redirect.
**Behavior:** Reset checkout state. Navigate to `/cart`. User re-initiates checkout to get a new session.

### Distinguishing error types

The order submission response handler (from WP-005-FE's `usePlaceOrder` mutation) must route errors based on HTTP status code:

| Status | Handler |
|---|---|
| 201 | Success → confirmation page (WP-005-FE) |
| 401 | Expired session → redirect to `/cart` (this WP) |
| 409 | Stock conflict → display conflict details (this WP) |
| 422 | Validation errors → field-level messages (WP-005-FE) |
| Other (4xx/5xx) | Payment failure or generic error (this WP) |

---

## State Management

### Checkout Error State

Extend the checkout store (from WP-004-FE) or use component-local state for error display:

```typescript
// Added to checkout store or managed via component state
interface CheckoutError {
  type: "stock_conflict" | "payment_failed" | "session_expired" | "generic";
  message: string;
  conflicts?: Array<{
    sku_id: string;
    product_name: string;  // resolved from cart items
    variant_label: string; // resolved from cart items
    requested: number;
    available: number;
  }>;
}
```

**Resolving product names for stock conflicts:** When a 409 response is received, the `conflicts` array contains `sku_id` values. Match these against `cartStore.items` (which contains `sku_id`, `product_name`, `variant_label`) to display human-readable names. If a `sku_id` from the conflict is not found in the cart (edge case), display the `sku_id` as a fallback.

### Cart Store (read-only)

Used to resolve `sku_id` → `product_name` / `variant_label` for stock conflict display. No modifications to cart in this WP (the user returns to cart manually to adjust quantities).

---

## Routing

No new routes. This WP enhances error handling on existing routes:

| Route | Component | Notes |
|---|---|---|
| `/checkout` | `CheckoutPage` | Enhanced with stock conflict, payment failure, and session expiry error handling |
| `/cart` | `CartPage` | Target of redirects from session expiry and stock conflict |

---

## Related Contracts

- `contracts/api/order-service.openapi.yaml` — `POST /checkout/guest/orders` error responses (401, 409)

**Schemas used:**

- `StockConflictError` (409 response):
  - `code` — `"STOCK_CONFLICT"`
  - `message` — descriptive string
  - `conflicts` — array of `{ sku_id: UUID, requested: integer, available: integer }`

- `ErrorResponse` (401 response):
  - `code` — `"SESSION_EXPIRED"` or `"INVALID_SESSION"`
  - `message` — descriptive string

**Mock strategy:** Use MSW to mock 401, 409, and payment failure responses during development and testing.

---

## Implementation Notes

### Files to Create

1. `src/components/checkout/StockConflictError.tsx` — Stock conflict error banner with affected items list and "Return to Cart" button.
2. `src/components/checkout/PaymentError.tsx` — Payment failure inline message.
3. `src/components/checkout/SessionExpiredError.tsx` — Session expired message with redirect logic.

### Files to Modify

4. `src/hooks/usePlaceOrder.ts` (from WP-005-FE) — Enhance error handling to classify errors by status code (401, 409, 422, other) and expose structured error state.
5. `src/pages/CheckoutPage.tsx` — Render the appropriate error component based on error type. Handle redirect to `/cart` on 401.
6. `src/components/checkout/PaymentForm.tsx` — Show `PaymentError` inline below the card element. Re-enable "Place Order" on payment failure.
7. MSW handlers — Add mock handlers for 409 (stock conflict), 401 (expired session), and payment failure scenarios.

### Error Classification Logic

In the `usePlaceOrder` mutation's `onError` callback or response handler:

```typescript
function classifyOrderError(response: Response, body: any): CheckoutError {
  if (response.status === 401) {
    return { type: "session_expired", message: "Your session has expired." };
  }
  if (response.status === 409 && body.code === "STOCK_CONFLICT") {
    return {
      type: "stock_conflict",
      message: body.message,
      conflicts: body.conflicts.map((c) => ({
        ...c,
        product_name: resolveProductName(c.sku_id),
        variant_label: resolveVariantLabel(c.sku_id),
      })),
    };
  }
  if (response.status === 422) {
    // Handled by WP-005-FE — field-level validation
    return { type: "generic", message: body.message };
  }
  // All other errors treated as payment/generic failure
  return {
    type: "payment_failed",
    message: "Payment could not be processed. Please try again or use a different payment method.",
  };
}
```

### Implementation Order

1. Define `CheckoutError` type and error classification logic
2. Enhance `usePlaceOrder` to classify and expose structured errors
3. Create `StockConflictError` component (affected items list + "Return to Cart")
4. Create `PaymentError` component (inline message + re-enable button)
5. Create `SessionExpiredError` component (message + redirect to `/cart`)
6. Wire error components into `CheckoutPage` / `PaymentForm`
7. Handle 401 redirect: reset checkout state → navigate to `/cart`
8. Add MSW handlers for 409, 401, and payment failure responses
9. Tests for all TS scenarios (TS-001-029, TS-001-030, TS-001-032, TS-001-033)
10. Run DoD checklist

---

## Dependency Info

**Wave 4** — parallel with WP-005-FE. Both build on the checkout form from WP-004-FE and the order submission flow.

In practice, implement WP-005-FE first (basic order submission and confirmation), then WP-006-FE (error cases). The error classification logic in `usePlaceOrder` may be initially stubbed in WP-005-FE and fully implemented in this WP.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-029, TS-001-030, TS-001-032, TS-001-033)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
