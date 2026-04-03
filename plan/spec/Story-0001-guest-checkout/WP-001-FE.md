# WP-001-FE: Guest Checkout — Frontend

**Feature:** Story-0001-guest-checkout
**Derived from:** FS-001, TS-001
**Target workspace:** `workspaces/storefront-app`
**Status:** Approved
**Last updated:** 2026-04-03

---

## Objective

Implement the guest checkout UI flow in the `storefront-app`. The flow guides a shopper through four steps:

1. Shipping information (name, email, address)
2. Shipping method selection
3. Payment entry (embedded payment gateway widget)
4. Order confirmation

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (copied verbatim from FS-001)

**AC-001 — Guest session creation**
A shopper who has not signed in can initiate a Guest Checkout session without providing any personal details upfront. The storefront must create a guest session when the guest checkout flow begins.

**AC-002 — Collect shipping information**
The shopper must provide: full name, email address (required, validated format), shipping address (line1, city, postal code, country code required; line2 and state optional). The form MUST validate all required fields before allowing progress. Field-level errors must appear adjacent to the invalid field.

**AC-003 — Shipping method selection**
At least one shipping method and its cost must be displayed before proceeding to payment. Selecting a shipping method updates the displayed order total.

**AC-005 — Order placement — happy path**
On successful placement: the Order Reference is displayed on a confirmation screen.

**AC-006 — Out-of-stock handling**
If placement returns a stock conflict, a user-facing message identifies the unavailable items by name. The "Place Order" button is re-enabled.

**AC-008 — Order confirmation email**
(UI concern: the confirmation screen must inform the shopper that a confirmation email has been sent to their email address.)

---

## Test Scenarios (copied verbatim from TS-001)

**TS-001-004** — All required shipping fields filled in correctly → checkout advances to shipping method step.

**TS-001-005** — `postal_code` left blank → form does not submit; field-level error "Postal code is required." appears.

**TS-001-006** — Invalid email format → field-level error "Please enter a valid email address."

**TS-001-007** — `line2` and `state` blank with all required fields filled → no error; checkout advances.

**TS-001-008** — Shipping method step shows at least one shipping method with name and cost.

**TS-001-009** — Selecting a shipping method with cost €5.99 → order total updates to include €5.99.

**TS-001-011** — Declined card → error message "Your payment was declined. Please check your card details or try a different card." No order created.

**TS-001-016** — Out-of-stock response from API → user-facing message lists unavailable items by name; "Place Order" button re-enabled.

---

## Checkout Flow

```
[Cart] → [Initiate Guest Checkout] → [Step 1: Shipping Info] → [Step 2: Shipping Method]
      → [Step 3: Payment] → [Step 4: Review & Place Order] → [Confirmation Screen]
```

### Step 0 — Initiate guest session

When the shopper clicks "Guest Checkout" on the cart page:
- Call `POST /checkout/guest/sessions` (optionally passing the current `cart_id`).
- Store the returned `token` in component state (not localStorage).
- Attach the token as `X-Guest-Session-Token` header on all subsequent checkout API calls.
- If the session call fails, display a generic error and allow the shopper to retry.

### Step 1 — Shipping information form

Fields:

| Field | Label | Required | Validation |
|---|---|---|---|
| `full_name` | Full name | Yes | Min 2 chars |
| `email` | Email address | Yes | Valid email format (RFC 5322) |
| `line1` | Address line 1 | Yes | Max 100 chars |
| `line2` | Address line 2 | No | Max 100 chars |
| `city` | City | Yes | Max 100 chars |
| `state` | State / Region | No | Max 100 chars |
| `postal_code` | Postal / ZIP code | Yes | Max 20 chars |
| `country_code` | Country | Yes | Dropdown; ISO 3166-1 alpha-2 values |

Validation runs on form submit (not on every keystroke). Errors are displayed inline below each invalid field. All errors are shown at once (not one at a time).

**CTA:** "Continue to Shipping"

### Step 2 — Shipping method selection

- Fetch available shipping methods from the backend and display them as a radio list.
- Each option shows: method name, estimated delivery window, cost.
- One method is pre-selected (cheapest by default).
- The order total panel (right side or bottom) updates reactively when selection changes.
- Total breakdown: Subtotal | Shipping | Tax | **Total**

**CTA:** "Continue to Payment"

### Step 3 — Payment

- Render the payment gateway's embedded card widget (iframe/web component).
- Do NOT build a custom card input. Use the gateway SDK.
- On tokenisation success, store the `token` in component state.
- On tokenisation error, display the gateway's error message.
- Display a brief summary of the order (items, total) above the payment widget for reassurance.

**CTA:** "Review Order"

### Step 4 — Review & Place Order

- Display a full order summary: items, shipping address, shipping method, total breakdown.
- "Place Order" button calls `POST /checkout/guest/orders` with:
  - `email` from Step 1
  - `shipping_address` from Step 1
  - `shipping_method_id` from Step 2
  - `payment_method.token` from Step 3
  - `Idempotency-Key`: generate a UUID at page load and re-use on retries
- Show a loading/spinner state while the request is in flight.
- On success: navigate to the Confirmation screen.
- On 409 (stock conflict): display the out-of-stock error (see below).
- On other errors: display "Something went wrong. Please try again." and keep the button enabled.

### Confirmation screen

Display:
- "Order placed!" heading.
- Order Reference (e.g. `ORD-20260403-A3K9`) prominently.
- "A confirmation email has been sent to {email}."
- Brief summary: items, total.
- CTA: "Continue Shopping" (links to catalogue).

---

## Error Handling

### Out-of-stock error

When the API returns HTTP 409 with `code: STOCK_CONFLICT`:
- Parse the `conflicts` array.
- Resolve each `sku_id` to a product name using the cart data already in state (the cart was loaded before checkout began).
- Show an inline alert: "The following items are no longer available: [Item A], [Item B]. Please update your cart."
- Re-enable the "Place Order" button.

### Declined payment

When the payment gateway SDK reports a decline:
- Show: "Your payment was declined. Please check your card details or try a different card."
- Keep the shopper on the payment step.
- Do NOT navigate away.

---

## State Management

- All checkout state (session token, form values, selected shipping method, payment token) is held in a single checkout context/store scoped to the checkout flow.
- Do NOT persist checkout state to localStorage or sessionStorage.
- If the shopper navigates away and returns, they restart from Step 1 (a new guest session will be created).

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/checkout/guest` | `GuestCheckoutPage` | Entry point; triggers session creation |
| `/checkout/guest/shipping` | `ShippingInfoStep` | Step 1 |
| `/checkout/guest/delivery` | `ShippingMethodStep` | Step 2 |
| `/checkout/guest/payment` | `PaymentStep` | Step 3 |
| `/checkout/guest/review` | `ReviewStep` | Step 4 |
| `/checkout/guest/confirmation` | `ConfirmationPage` | Post-placement |

All routes after `/checkout/guest` MUST redirect to `/checkout/guest` if no valid session token exists in state.

---

## Related Contracts

- `contracts/api/order-service.openapi.yaml` — `createGuestSession` (Step 0), `placeGuestOrder` (Step 4).
- `contracts/api/inventory-service.openapi.yaml` — shipping methods and stock queries (if fetched client-side; otherwise proxied through order-service).
- `contracts/architecture/ADR-001.md` — explains why session token is stored in memory only.
