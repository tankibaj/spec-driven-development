# WP-004-FE: storefront-app — Checkout Form

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/storefront-app`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the guest checkout multi-step form in storefront-app: create a guest session via API, collect shipping address with validation, select a shipping method from available options fetched from the API, and collect payment details via Stripe Elements (test mode). After this WP, the user can complete all checkout steps and reach a "Place Order" state — actual order submission is handled by WP-005-FE.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-010 — Guest session creation

The system creates a guest checkout session when the guest begins checkout. The session token is used for all subsequent checkout API calls.

**Testable:** `POST /checkout/guest/sessions` returns `201` with a `GuestSession` containing `id` (UUID), `token` (opaque string), and `expires_at` (future timestamp).

---

### AC-011 — Checkout form

The guest completes a multi-step checkout form: (1) shipping address with required fields (line1, city, postal_code, country_code), (2) shipping method selection from available options, and (3) payment details.

**Testable:** When the guest proceeds to checkout, then a multi-step form is presented with shipping address fields, a shipping method selector showing available options with costs, and a payment details form. All required fields must be completed before the order can be submitted.

---

## Test Scenarios (from TS-001)

### TS-001-016 — POST /checkout/guest/sessions creates a guest session
- **Preconditions:** MSW mock: `POST /checkout/guest/sessions` returns 201 with `{ id: "uuid", token: "session-token-abc", expires_at: "2026-04-12T..." }`.
- **Action:** User clicks "Checkout" on the cart page (cart has items).
- **Expected:** Guest session is created. Session token stored in checkout state. User navigated to checkout step 1. Subsequent checkout API calls include `X-Guest-Session-Token: session-token-abc`.

### TS-001-017 — Guest session token is usable in subsequent requests
- **Preconditions:** Guest session created with token "session-token-abc". MSW mock verifies `X-Guest-Session-Token` header presence on checkout API calls.
- **Action:** User proceeds through checkout steps (which trigger API calls).
- **Expected:** All checkout API calls include the `X-Guest-Session-Token` header with the captured token. No 401 responses (token is accepted).

### TS-001-018 — Shipping methods returned from API
- **Preconditions:** MSW mock: `GET /checkout/shipping-methods` returns 200 with `[{ id: "sm-1", name: "Standard Shipping", description: "3-5 business days", cost_minor: 599, estimated_days_min: 3, estimated_days_max: 5 }, { id: "sm-2", name: "Express Shipping", description: "1-2 business days", cost_minor: 1499, estimated_days_min: 1, estimated_days_max: 2 }]`.
- **Action:** User reaches the shipping method step in checkout.
- **Expected:** Shipping method selector displays 2 options. "Standard Shipping" shows "$5.99" and "3-5 business days". "Express Shipping" shows "$14.99" and "1-2 business days".

### TS-001-019 — Checkout form renders multi-step UI with shipping methods
- **Preconditions:** storefront-app loaded. Guest session created. Cart has at least one item. MSW mock for shipping methods returns 2 options.
- **Action:** User navigates to checkout.
- **Expected:** Step 1 shows shipping address fields: line1 (required), line2, city (required), state, postal_code (required), country_code (required). Step 2 shows shipping method selector with 2 options displaying name, estimated delivery, and cost. Step 3 shows payment details form. All required fields marked as required. "Place Order" button disabled until all required steps complete.

### TS-001-020 — Checkout form enforces required fields
- **Preconditions:** storefront-app loaded. Guest session active. Cart has items. User is on shipping address step.
- **Action:** User leaves required fields empty (line1, city, postal_code, country_code) and attempts to proceed.
- **Expected:** Form does not advance to the next step. Validation messages appear on each empty required field. User cannot submit the order.

---

## UI Flow

```
[Cart Page] → "Checkout" button
  → POST /checkout/guest/sessions → store token
  → [Step 1: Shipping Address]
    → fill fields → "Continue" → validate required fields
  → [Step 2: Shipping Method]
    → GET /checkout/shipping-methods → display options
    → select method → "Continue"
  → [Step 3: Payment Details]
    → Stripe Elements card input
    → "Place Order" button (disabled until all steps complete)
    → [Order submission handled by WP-005-FE]
```

### Step 1 — Shipping Address

The user fills in their shipping address.

| Field | Label | Required | Validation |
|---|---|---|---|
| `line1` | Address Line 1 | Yes | Non-empty, max 100 characters |
| `line2` | Address Line 2 | No | Max 100 characters |
| `city` | City | Yes | Non-empty, max 100 characters |
| `state` | State / Province | No | Max 100 characters |
| `postal_code` | Postal Code | Yes | Non-empty, max 20 characters |
| `country_code` | Country | Yes | 2-letter ISO 3166-1 alpha-2 code. Render as dropdown of country options. |

**CTA:** "Continue to Shipping" — validates all required fields. If any empty, show inline validation message: "{Field Label} is required". Does not advance until all required fields pass.

### Step 2 — Shipping Method

The user selects a shipping method from options fetched via `GET /checkout/shipping-methods`.

Each option displays:
- **Name** — e.g., "Standard Shipping"
- **Description** — e.g., "Delivery in 3-5 business days"
- **Cost** — `formatPrice(cost_minor)` e.g., "$5.99"
- **Estimated delivery** — derived from `estimated_days_min` / `estimated_days_max` (e.g., "3-5 business days")

Render as a radio button group or selectable card list. One option must be selected to proceed.

**CTA:** "Continue to Payment" — disabled until a shipping method is selected.

### Step 3 — Payment Details

The user enters payment card details using Stripe Elements (test mode).

- **Email** — email address for order confirmation (required, valid email format). Prefill if previously entered.
- **Card input** — Stripe `CardElement` (handles card number, expiry, CVC in a single secure iframe).
- Stripe publishable key sourced from environment variable: `VITE_STRIPE_PUBLISHABLE_KEY`.
- For testing, use Stripe test publishable key `pk_test_...` (value configured in `.env`, NOT hardcoded).

**CTA:** "Place Order" — disabled until: (a) all address fields valid, (b) shipping method selected, (c) Stripe card element reports complete, (d) email is valid. Order submission logic is in WP-005-FE.

---

## Error Handling

### Guest session creation failure

**When:** `POST /checkout/guest/sessions` returns non-201 (network error, 400, 5xx).
**Display:** "Unable to start checkout. Please try again."
**Behavior:** User stays on the cart page. "Checkout" button re-enabled. Retry on next click.

### Shipping methods fetch failure

**When:** `GET /checkout/shipping-methods` returns non-200 (network error, 5xx).
**Display:** "Unable to load shipping options. Please try again." with a "Retry" button.
**Behavior:** User stays on step 2. Retry button triggers TanStack Query refetch.

### Client-side validation failures

**When:** User attempts to proceed past step 1 with empty required fields.
**Display:** Inline validation message below each empty required field: "{Field Label} is required".
**Behavior:** Form does not advance. Focus moves to the first invalid field.

### Stripe Elements error

**When:** Stripe CardElement reports a validation error (e.g., invalid card number).
**Display:** Stripe's built-in error message displayed below the card input.
**Behavior:** "Place Order" button remains disabled until Stripe reports the card element is complete.

---

## State Management

### Checkout State (Zustand Store)

Create `src/stores/checkout-store.ts` using Zustand:

```typescript
interface CheckoutState {
  // Guest session
  sessionToken: string | null;
  sessionExpiresAt: string | null;

  // Step tracking
  currentStep: 1 | 2 | 3;

  // Step 1: Shipping address
  shippingAddress: {
    line1: string;
    line2: string;
    city: string;
    state: string;
    postal_code: string;
    country_code: string;
  };

  // Step 2: Shipping method
  selectedShippingMethodId: string | null;

  // Step 3: Payment
  email: string;
  stripePaymentMethodToken: string | null; // set after Stripe tokenization

  // Actions
  setSessionToken: (token: string, expiresAt: string) => void;
  setShippingAddress: (address: Partial<CheckoutState["shippingAddress"]>) => void;
  setShippingMethodId: (id: string) => void;
  setEmail: (email: string) => void;
  setStripePaymentMethodToken: (token: string) => void;
  setCurrentStep: (step: 1 | 2 | 3) => void;
  resetCheckout: () => void;
}
```

**This store is transient (NOT persisted to localStorage).** If the user navigates away from checkout and returns, they restart from the beginning (new guest session).

### Cart Store (read-only in this WP)

Cart contents are read from `useCartStore()` (from WP-002-FE/WP-003-FE). The checkout form does not modify the cart — it reads `items` to build the order payload (in WP-005-FE).

### TanStack Query Hooks

- `useCreateGuestSession()` — mutation: `POST /checkout/guest/sessions`. On success, stores token via `setSessionToken()`.
- `useShippingMethods()` — query: `GET /checkout/shipping-methods`. Fetched when user reaches step 2.

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/checkout` | `CheckoutPage` | Multi-step form (steps 1-3). Redirects to `/cart` if cart is empty. |
| `/checkout/confirmation` | Reserved | Used by WP-005-FE for order confirmation. Not implemented here. |

**Route guard:** If cart is empty (`cartStore.items.length === 0`), redirect from `/checkout` to `/cart`.

---

## Related Contracts

- `workspaces/order-service/docs/api/openapi.json` — `POST /checkout/guest/sessions` (createGuestSession), `GET /checkout/shipping-methods` (listShippingMethods)

**Schemas used:**
- `GuestSession` — `id` (UUID), `token` (string), `expires_at` (date-time)
- `ShippingMethod` — `id` (UUID), `name`, `description`, `cost_minor` (integer), `estimated_days_min`, `estimated_days_max`
- `Address` — `line1`, `line2`, `city`, `state`, `postal_code`, `country_code`

**Headers:**
- `X-Tenant-ID` — required on all order-service calls (UUID, from env var `VITE_TENANT_ID`).
- `X-Guest-Session-Token` — required on all checkout calls after session creation (from checkout store).

**Mock strategy:** Use MSW against the order-service OpenAPI spec above during development and testing. Switch to the real order-service backend URL once WP-002-BE is deployed.

---

## Implementation Notes

### Stripe Elements Setup

1. Install `@stripe/stripe-js` and `@stripe/react-stripe-js`.
2. Create Stripe provider in `src/providers/StripeProvider.tsx`:
   - Load Stripe with `loadStripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY)`.
   - Wrap checkout page with `<Elements stripe={stripePromise}>`.
3. Use `<CardElement>` for card input.
4. Use `useStripe()` and `useElements()` hooks for tokenization (tokenization itself called in WP-005-FE when "Place Order" is clicked).
5. For tests, mock `@stripe/react-stripe-js` — render a test input in place of `CardElement` and mock `stripe.createPaymentMethod()`.

### Environment Variables

| Variable | Purpose | Example Value |
|---|---|---|
| `VITE_TENANT_ID` | Tenant ID for API headers | `550e8400-e29b-41d4-a716-446655440000` |
| `VITE_ORDER_API_URL` | Order-service base URL | `http://localhost:3002/v1` (or MSW in test) |
| `VITE_STRIPE_PUBLISHABLE_KEY` | Stripe test publishable key | `pk_test_...` |

### Files to Create

1. `src/stores/checkout-store.ts` — Checkout state (Zustand, transient)
2. `src/pages/CheckoutPage.tsx` — Multi-step form container
3. `src/components/checkout/ShippingAddressForm.tsx` — Step 1 form
4. `src/components/checkout/ShippingMethodSelector.tsx` — Step 2 selector
5. `src/components/checkout/PaymentForm.tsx` — Step 3 with Stripe Elements
6. `src/components/checkout/CheckoutStepper.tsx` — Step indicator (Step 1 of 3, etc.)
7. `src/providers/StripeProvider.tsx` — Stripe Elements provider
8. `src/api/order-client.ts` — Order-service API client (fetch wrapper with `X-Tenant-ID` + `X-Guest-Session-Token`)
9. `src/hooks/useCreateGuestSession.ts` — TanStack Query mutation
10. `src/hooks/useShippingMethods.ts` — TanStack Query query
11. MSW handlers for `POST /checkout/guest/sessions` and `GET /checkout/shipping-methods`

### Files to Modify

12. `src/pages/CartPage.tsx` — Wire "Checkout" button to create guest session then navigate to `/checkout`.
13. Router config — Add `/checkout` route with cart-empty guard.

### Implementation Order

1. Create order-service API client (`order-client.ts`)
2. Create checkout Zustand store
3. Create `useCreateGuestSession` mutation hook
4. Wire "Checkout" button on cart page → create session → navigate to `/checkout`
5. Create `CheckoutPage` container with step management
6. Create `CheckoutStepper` (step indicator)
7. Create `ShippingAddressForm` (step 1) with client-side validation
8. Create `useShippingMethods` query hook + MSW handler
9. Create `ShippingMethodSelector` (step 2)
10. Set up Stripe Elements provider
11. Create `PaymentForm` (step 3) with Stripe `CardElement` + email field
12. Wire "Place Order" button disabled state (all steps must be complete)
13. Add `/checkout` route with cart-empty guard
14. MSW handlers for order-service checkout endpoints
15. Tests for all TS scenarios (TS-001-016, TS-001-017, TS-001-018, TS-001-019, TS-001-020)
16. Run DoD checklist

---

## Dependency Info

**Wave 3** — depends on WP-002-FE and WP-003-FE (cart provides items for checkout and the "Checkout" entry point).

WP-005-FE depends on this WP (order submission builds on the completed checkout form).
WP-006-FE depends on this WP (error handling builds on the checkout flow).

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-016, TS-001-017, TS-001-018, TS-001-019, TS-001-020)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
