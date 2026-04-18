# WP-005-FE: storefront-app — Order Submission + Confirmation

**Feature:** story-0001-mvp-guest-shopping
**Target workspace:** `workspaces/storefront-app`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement order submission and the order confirmation page in storefront-app: when the user clicks "Place Order", tokenize the payment via Stripe, submit the order to `POST /checkout/guest/orders`, navigate to a confirmation page on 201 success displaying order reference, items, and total, and handle 422 validation errors by displaying field-level messages on the checkout form.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-012 — Successful order placement

After form submission, the system reserves stock, captures payment, and confirms the order atomically (saga pattern). The response includes a confirmed order with a human-readable reference.

**Testable:** `POST /checkout/guest/orders` with valid `X-Guest-Session-Token`, complete `shipping_address`, valid `shipping_method_id`, valid `payment_method` token, and at least one order `line` returns `201` with an `Order` containing `status: confirmed`, a non-empty `reference` (e.g. `ORD-20240415-A3K9`), and `lines` matching the submitted items.

---

### AC-013 — Order confirmation page

After successful order placement, the guest sees a confirmation page with the order reference, ordered items, and order total.

**Testable:** When the order placement returns `201`, then the confirmation page displays the `reference`, each ordered item with product name and quantity, and the `total` formatted as currency.

---

### AC-014 — Checkout validation errors

Missing or invalid fields in the checkout form produce clear, field-level error messages.

**Testable:** `POST /checkout/guest/orders` with missing required fields (e.g. missing `postal_code` in `shipping_address`) returns `422` with a `details` array containing `field` and `issue` entries. UI displays field-level validation messages on the corresponding form fields.

---

## Test Scenarios (from TS-001)

### TS-001-021 — Successful order returns 201 with confirmed order (FE aspect)
- **Preconditions:** storefront-app loaded. Checkout form fully completed. MSW mock: `POST /checkout/guest/orders` returns 201 with `{ id: "order-uuid", reference: "ORD-20260411-A3K9", status: "confirmed", lines: [{ sku_id: "sku-1", product_name: "Classic T-Shirt", variant_label: "Small", quantity: 2, unit_price: 2999, subtotal: 5998 }], total: 7497, created_at: "2026-04-11T..." }`.
- **Action:** User clicks "Place Order".
- **Expected:** UI navigates to `/checkout/confirmation`. Order reference "ORD-20260411-A3K9" displayed prominently. Line item shown: "Classic T-Shirt", quantity 2. Order total displayed as "$74.97". Cart is cleared.

### TS-001-024 — Confirmation page displays order reference, items, and total
- **Preconditions:** storefront-app loaded. MSW mock: `POST /checkout/guest/orders` returns 201 with order reference "ORD-20260411-A3K9", one line item ("Classic T-Shirt / Small", qty 2, unit_price 2999), total 7497.
- **Action:** User submits checkout form (order placement succeeds).
- **Expected:** UI navigates to confirmation page. Order reference "ORD-20260411-A3K9" displayed prominently. Ordered item: "Classic T-Shirt", quantity 2. Order total: "$74.97". "Continue Shopping" link available.

### TS-001-025 — Missing required field returns 422 with field-level error (FE aspect)
- **Preconditions:** storefront-app loaded. Checkout form filled but MSW mock: `POST /checkout/guest/orders` returns 422 with `{ code: "VALIDATION_ERROR", message: "Validation failed", details: [{ field: "shipping_address.postal_code", issue: "Postal code is required" }] }`.
- **Action:** User clicks "Place Order".
- **Expected:** User stays on checkout form. Field-level error displayed on the postal code field: "Postal code is required". "Place Order" button re-enabled for retry.

### TS-001-026 — Multiple validation errors returned in single response (FE aspect)
- **Preconditions:** storefront-app loaded. MSW mock: `POST /checkout/guest/orders` returns 422 with `{ code: "VALIDATION_ERROR", message: "Validation failed", details: [{ field: "email", issue: "Email is required" }, { field: "shipping_address.line1", issue: "Address line 1 is required" }, { field: "shipping_address.postal_code", issue: "Postal code is required" }] }`.
- **Action:** User clicks "Place Order".
- **Expected:** Multiple field-level errors displayed simultaneously on the corresponding form fields. User can correct all errors and resubmit.

---

## UI Flow

```
[Checkout Step 3: Payment] → "Place Order" click
  → Stripe tokenization (stripe.createPaymentMethod)
  → POST /checkout/guest/orders
    → 201: clear cart → navigate to [Order Confirmation Page]
    → 422: display field-level errors → stay on form
```

### Step 1 — Order Submission (from Checkout Step 3)

When the user clicks "Place Order":

1. **Disable the button** and show a loading indicator (e.g., spinner, "Processing...").
2. **Tokenize payment** via Stripe: call `stripe.createPaymentMethod({ type: "card", card: cardElement })`. On Stripe error, display the error and re-enable the button.
3. **Build the request body** (`PlaceGuestOrderRequest`):
   ```json
   {
     "email": "<from checkout store>",
     "shipping_address": {
       "line1": "<from checkout store>",
       "line2": "<from checkout store>",
       "city": "<from checkout store>",
       "state": "<from checkout store>",
       "postal_code": "<from checkout store>",
       "country_code": "<from checkout store>"
     },
     "shipping_method_id": "<from checkout store>",
     "payment_method": {
       "type": "card",
       "token": "<Stripe PaymentMethod ID from step 2>"
     },
     "lines": [
       { "sku_id": "<from cart store>", "quantity": <from cart store> }
       // ... for each cart item
     ]
   }
   ```
4. **Call** `POST /checkout/guest/orders` with headers:
   - `X-Tenant-ID: <env var>`
   - `X-Guest-Session-Token: <from checkout store>`
   - `Content-Type: application/json`
5. **On 201:** navigate to `/checkout/confirmation` with order data. Clear the cart (`cartStore.clearCart()`). Reset checkout state (`checkoutStore.resetCheckout()`).
6. **On 422:** parse `details` array and display field-level errors (see Error Handling).
7. **On other errors (409, 401):** handled by WP-006-FE. This WP passes non-422 errors to a shared error handler.

### Step 2 — Order Confirmation Page (`/checkout/confirmation`)

Displays after successful order placement:

- **Order reference** — displayed prominently (e.g., heading: "Order Confirmed! Your reference: ORD-20260411-A3K9").
- **Ordered items** — list of line items, each showing `product_name` and `quantity`.
- **Order total** — `formatPrice(order.total)` (e.g., "$74.97").
- **"Continue Shopping"** link → navigates to `/products`.

The order data is passed to the confirmation page via:
- **Option A (recommended):** React Router state (`navigate("/checkout/confirmation", { state: { order } })`). Read via `useLocation().state`.
- If the user navigates directly to `/checkout/confirmation` without order state, redirect to `/products`.

---

## Error Handling

### Stripe tokenization failure

**When:** `stripe.createPaymentMethod()` returns an error (e.g., invalid card, Stripe service unavailable).
**Display:** Stripe's error message below the card element (e.g., "Your card number is invalid").
**Behavior:** "Place Order" button re-enabled. User corrects card details and retries.

### 422 Validation errors

**When:** `POST /checkout/guest/orders` returns 422 with `details` array.
**Display:** For each entry in `details`, display the `issue` message inline on the corresponding form field. Field mapping:

| API `field` value | Form field |
|---|---|
| `email` | Email input (Step 3) |
| `shipping_address.line1` | Address Line 1 (Step 1) |
| `shipping_address.city` | City (Step 1) |
| `shipping_address.postal_code` | Postal Code (Step 1) |
| `shipping_address.country_code` | Country (Step 1) |
| `shipping_method_id` | Shipping method selector (Step 2) |
| `lines` | General error above the form |

**Behavior:** Navigate back to the earliest step containing an error (e.g., if `shipping_address.postal_code` has an error, go to Step 1). "Place Order" button re-enabled. Form remains filled — user corrects and resubmits.

### Network error on order submission

**When:** `POST /checkout/guest/orders` fails with a network error or 5xx.
**Display:** "Something went wrong. Please try again."
**Behavior:** "Place Order" button re-enabled. User can retry.

### Direct navigation to confirmation without order data

**When:** User navigates to `/checkout/confirmation` directly (no order state in router).
**Display:** Redirect to `/products`.
**Behavior:** No confirmation page rendered.

---

## State Management

### Checkout Store (from WP-004-FE)

This WP reads from and writes to the checkout store created in WP-004-FE:

- **Reads:** `sessionToken`, `shippingAddress`, `selectedShippingMethodId`, `email`, `stripePaymentMethodToken`
- **Writes:** `setStripePaymentMethodToken(token)` after Stripe tokenization, `resetCheckout()` after successful order

### Cart Store (from WP-002-FE/WP-003-FE)

- **Reads:** `items` — to build `lines` array in the order request
- **Writes:** `clearCart()` — after successful order placement (201)

### Order Confirmation Data

The confirmed order response is passed to the confirmation page via React Router state. It is transient — not persisted to any store. If the user refreshes the confirmation page, they are redirected to `/products`.

### TanStack Query

- `usePlaceOrder()` — mutation: `POST /checkout/guest/orders`. Returns the `Order` response on success. Exposes `error` for 422/other handling.

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/checkout` | `CheckoutPage` | From WP-004-FE. "Place Order" wired in this WP. |
| `/checkout/confirmation` | `OrderConfirmationPage` | Displays order details. Redirects to `/products` if no order state. |

---

## Related Contracts

- `contracts/api/order-service.openapi.yaml` — `POST /checkout/guest/orders` (placeGuestOrder)

**Schemas used:**

- `PlaceGuestOrderRequest`:
  - `email` (string, email format)
  - `shipping_address` — `Address` schema: `line1`, `line2`, `city`, `state`, `postal_code`, `country_code`
  - `shipping_method_id` (UUID)
  - `payment_method` — `PaymentMethodInput`: `type` ("card"), `token` (Stripe PaymentMethod ID)
  - `lines` — array of `{ sku_id: UUID, quantity: integer >= 1 }`

- `Order` (201 response):
  - `id` (UUID), `reference` (string, e.g., "ORD-20260411-A3K9"), `status` ("confirmed")
  - `lines` — array of `OrderLine`: `sku_id`, `product_name`, `variant_label`, `quantity`, `unit_price`, `subtotal`
  - `total` (integer, minor currency units)
  - `shipping_address` — `Address`
  - `created_at` (date-time)

- `ErrorResponse` (422 response):
  - `code` (string), `message` (string)
  - `details` — array of `{ field: string, issue: string }`

**Headers:**
- `X-Tenant-ID` — tenant identifier (UUID)
- `X-Guest-Session-Token` — guest session token from checkout store

**Mock strategy:** Use MSW against the order-service OpenAPI spec during development and testing. Switch to real order-service URL once WP-002-BE is deployed.

---

## Implementation Notes

### Files to Create

1. `src/hooks/usePlaceOrder.ts` — TanStack Query mutation for `POST /checkout/guest/orders`
2. `src/pages/OrderConfirmationPage.tsx` — Confirmation page component
3. `src/components/checkout/OrderSummary.tsx` — Displays ordered items and total on confirmation page
4. MSW handler for `POST /checkout/guest/orders` (success 201, validation 422 variants)

### Files to Modify

5. `src/components/checkout/PaymentForm.tsx` (from WP-004-FE) — Wire "Place Order" button to: Stripe tokenization → `usePlaceOrder` mutation → handle 201/422
6. `src/pages/CheckoutPage.tsx` (from WP-004-FE) — Add server-side validation error state, navigate to earliest errored step on 422
7. Router config — Add `/checkout/confirmation` route

### Implementation Order

1. Create `usePlaceOrder` mutation hook
2. Wire "Place Order" button: Stripe tokenization → build request → submit
3. Handle 201 response: clear cart, reset checkout, navigate to confirmation
4. Create `OrderConfirmationPage` with order reference, items, total, "Continue Shopping" link
5. Handle 422 response: parse `details`, map to form fields, navigate to errored step
6. Handle network errors: display generic error, re-enable button
7. Add MSW handlers for `POST /checkout/guest/orders` (201, 422 scenarios)
8. Add `/checkout/confirmation` route with guard (no order state → redirect to `/products`)
9. Tests for all TS scenarios (TS-001-021, TS-001-024, TS-001-025, TS-001-026)
10. Run DoD checklist

---

## Dependency Info

**Wave 4** — depends on WP-004-FE (checkout form provides the multi-step UI and "Place Order" button).

WP-006-FE handles additional error states (409 stock conflict, payment failure, expired session) that can occur during order submission. WP-006-FE builds on the submission flow implemented here.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-021, TS-001-024, TS-001-025, TS-001-026)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
