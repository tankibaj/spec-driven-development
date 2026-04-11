# WP-003-FE: storefront-app — Cart Editing + Persistence

**Feature:** story-0001-mvp-guest-shopping
**Target workspace:** `workspaces/storefront-app`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Add cart editing capabilities (change item quantity, remove items with reactive total recalculation) and localStorage persistence to the storefront cart. After this WP, the cart survives page refreshes and tab close/reopen, and the user can fully manage cart contents before proceeding to checkout.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-007 — Cart editing and empty state

The guest can change an item's quantity or remove an item entirely. Totals recalculate on every change. When all items are removed, an empty cart state is shown.

**Testable:** When the guest changes an item's quantity, then the line subtotal and cart total recalculate. When the guest removes an item, it disappears from the cart and the total updates. When the last item is removed, the cart displays "Your cart is empty" with a link to browse products.

---

### AC-008 — Cart persistence

The cart survives page refreshes and browser tab close/reopen within the same browser.

**Testable:** When the guest adds items to the cart, refreshes the page, and reopens the cart, then the same items and quantities are present.

---

## Test Scenarios (from TS-001)

### TS-001-011 — Change item quantity recalculates subtotal and total
- **Preconditions:** storefront-app loaded. Cart contains SKU-A (qty 2, `price_minor: 2999`) and SKU-B (qty 1, `price_minor: 8999`). Cart total is "$149.97".
- **Action:** User changes SKU-A quantity to 3.
- **Expected:** SKU-A line subtotal updates to "$89.97" (3 x $29.99). Cart total updates to "$179.96" ($89.97 + $89.99).

### TS-001-012 — Remove item from cart updates total
- **Preconditions:** storefront-app loaded. Cart contains SKU-A (qty 2) and SKU-B (qty 1).
- **Action:** User clicks "Remove" on SKU-B.
- **Expected:** SKU-B no longer in cart. Cart shows 1 line item (SKU-A only). Cart total updates to SKU-A subtotal only.

### TS-001-013 — Removing last item shows empty cart state
- **Preconditions:** storefront-app loaded. Cart contains only SKU-A (qty 1).
- **Action:** User clicks "Remove" on SKU-A.
- **Expected:** Cart displays "Your cart is empty". A link to "browse products" is visible, pointing to `/products`. Cart icon count shows 0.

### TS-001-014 — Cart survives page refresh
- **Preconditions:** storefront-app loaded. Cart contains SKU-A (qty 2) and SKU-B (qty 1).
- **Action:** User refreshes the browser page (F5 / `location.reload`). User opens the cart view.
- **Expected:** Cart still contains SKU-A (qty 2) and SKU-B (qty 1). Quantities and product details unchanged. Cart total unchanged.

---

## UI Flow

```
[Cart Page: /cart]
  → each item row has quantity controls and "Remove" button
  → change quantity → subtotal + total recalculate reactively
  → remove item → item disappears, total recalculates
  → remove last item → [Empty Cart State]

[Empty Cart State]
  → "Your cart is empty"
  → "Browse products" link → /products
```

### Step 1 — Quantity Controls (Cart Page)

Each cart item row (from WP-002-FE's `CartItemRow` component) is enhanced with:

- **Quantity input/stepper:** Allows changing the quantity (minimum 1). On change, calls `cartStore.updateQuantity(skuId, newQty)`.
- **"Remove" button:** Removes the item entirely. Calls `cartStore.removeItem(skuId)`.

Line subtotal (`price_minor * quantity`) and cart total recalculate reactively via Zustand selectors.

### Step 2 — Empty Cart State

When `cartStore.items.length === 0`:

- Display "Your cart is empty" message.
- Display a "Browse products" link that navigates to `/products`.
- Cart icon count in header shows 0 (or no badge).
- "Checkout" button is hidden or disabled.

---

## Error Handling

### Quantity set to zero or less

**When:** User attempts to set quantity to 0 or a negative number.
**Display:** No visual error — quantity input enforces minimum value of 1.
**Behavior:** If the user wants to remove the item, they use the "Remove" button. Quantity input clamps to 1 as minimum.

### Corrupted localStorage data

**When:** `localStorage` cart data is present but malformed (e.g., user manually edited it, or schema changed between versions).
**Display:** No user-visible error.
**Behavior:** Zustand `persist` middleware's `onRehydrateStorage` callback catches parse errors. On failure, reset to empty cart. Log warning to console.

---

## State Management

### Cart Store Enhancement

The Zustand cart store created in WP-002-FE (`src/stores/cart-store.ts`) is enhanced in this WP:

**Existing methods (from WP-002-FE):**
- `addItem(item)` — add or increment quantity
- `removeItem(skuId)` — remove item by SKU ID
- `updateQuantity(skuId, quantity)` — set quantity for a SKU
- `clearCart()` — remove all items
- `totalItems()` — sum of quantities
- `totalPrice()` — sum of `price_minor * quantity`

**Changes in this WP:**

1. **Wrap store with Zustand `persist` middleware** using `localStorage` adapter:
   ```typescript
   import { persist } from "zustand/middleware";

   export const useCartStore = create<CartStore>()(
     persist(
       (set, get) => ({
         // ... existing store implementation
       }),
       {
         name: "storefront-cart", // localStorage key
         version: 1,
         // onRehydrateStorage: handle parse errors gracefully
       }
     )
   );
   ```

2. **`updateQuantity` validation:** If `quantity < 1`, clamp to 1 (do not remove — user must explicitly click "Remove").

3. **`removeItem` reactivity:** After removing, if `items.length === 0`, the cart page automatically shows the empty state (no explicit action needed — Zustand subscription triggers re-render).

**Persistence details:**
- `localStorage` key: `"storefront-cart"`
- Serialization: JSON (default Zustand persist behavior)
- Cart data persists across: page refresh, tab close/reopen, browser restart (same browser, same origin)
- Cart does NOT persist across: different browsers, incognito ↔ normal mode, `localStorage.clear()`

---

## Routing

No new routes. This WP enhances the existing `/cart` route (from WP-002-FE).

| Route | Component | Notes |
|---|---|---|
| `/cart` | `CartPage` | Enhanced with quantity editing, remove, empty state |

---

## Related Contracts

None. This WP is entirely client-side. No API calls are made for cart editing or persistence.

---

## Implementation Notes

### Files to Modify

1. `src/stores/cart-store.ts` — Wrap with Zustand `persist` middleware. Add `onRehydrateStorage` error handling.
2. `src/components/CartItemRow.tsx` — Add quantity input/stepper and "Remove" button.
3. `src/pages/CartPage.tsx` — Add empty cart state rendering when `items.length === 0`.

### Files to Create

4. `src/components/EmptyCart.tsx` — "Your cart is empty" + "Browse products" link.
5. `src/components/QuantityControl.tsx` — Quantity input/stepper component (increment/decrement buttons, numeric input, min value 1).

### Implementation Order

1. Add Zustand `persist` middleware to the cart store
2. Add `onRehydrateStorage` error handling for corrupted data
3. Create `QuantityControl` component
4. Enhance `CartItemRow` with quantity control and "Remove" button
5. Create `EmptyCart` component ("Your cart is empty" + link)
6. Enhance `CartPage` to show empty state when cart is empty
7. Tests for all TS scenarios (TS-001-011, TS-001-012, TS-001-013, TS-001-014)
8. Run DoD checklist

### Testing Notes for TS-001-014 (Persistence)

To test cart persistence across page refresh in Vitest + RTL:
- Render the app, add items to cart.
- Verify `localStorage.getItem("storefront-cart")` contains expected data.
- Unmount the component.
- Re-render the app (simulates page reload — Zustand `persist` rehydrates from `localStorage`).
- Verify cart items are restored with correct quantities.

---

## Dependency Info

**Wave 2** — parallel with WP-002-FE in practice (both modify the cart store). If developing sequentially, implement WP-002-FE first (creates the store), then WP-003-FE (wraps it with persistence and adds editing).

WP-004-FE depends on this WP (checkout reads from the persisted cart).

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-011, TS-001-012, TS-001-013, TS-001-014)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
