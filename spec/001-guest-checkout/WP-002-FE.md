# WP-002-FE: storefront-app — Cart Core

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/storefront-app`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the shopping cart core in storefront-app: add items to cart from the product detail page, display cart contents with product names, variant labels, quantities, prices, and totals, and block out-of-stock variants from being added.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-005 — Add item to cart

The guest can add a product variant (SKU) to the cart. If the variant is already in the cart, its quantity increments.

**Testable:** When the guest clicks "Add to Cart" on an in-stock variant, then the cart icon count increments and the cart contains that SKU with quantity 1 (or quantity incremented by 1 if already present).

---

### AC-006 — Cart contents display

The cart displays all added items with product name, variant label, quantity, unit price, and line subtotal. A cart total is shown summarising all items.

**Testable:** When the guest opens the cart, then each item shows product name, variant label, quantity, unit price (`price_minor` formatted as currency), and line subtotal (quantity × unit price). The footer shows the cart total across all line subtotals.

---

### AC-009 — Out-of-stock variant blocked

The guest cannot add an out-of-stock variant to the cart.

**Testable:** When a SKU has `stock_level` of `0` (from `GET /products/{productId}`), then the "Add to Cart" button for that variant is disabled and an "Out of stock" indicator is displayed.

---

## Test Scenarios (from TS-001)

### TS-001-008 — Add new SKU to cart creates cart entry with quantity 1
- **Preconditions:** storefront-app loaded. Cart is empty. Product detail page displayed with in-stock SKU-A (`stock_level > 0`).
- **Action:** User clicks "Add to Cart" button for SKU-A.
- **Expected:** Cart icon count shows 1. Cart store contains one entry: `{ sku_id: SKU-A, quantity: 1 }` with product name and variant label stored.

### TS-001-009 — Add existing SKU to cart increments quantity
- **Preconditions:** storefront-app loaded. Cart already contains SKU-A with quantity 1.
- **Action:** User clicks "Add to Cart" for SKU-A again.
- **Expected:** Cart icon count remains 1 (one line item). Cart entry for SKU-A has `quantity: 2`.

### TS-001-010 — Cart displays items with names, quantities, prices, and totals
- **Preconditions:** storefront-app loaded. Cart contains: SKU-A ("Classic T-Shirt / Small", qty 2, `price_minor: 2999`) and SKU-B ("Denim Jacket / Medium", qty 1, `price_minor: 8999`).
- **Action:** User opens the cart view.
- **Expected:** 2 line items. SKU-A line: "Classic T-Shirt", "Small", qty 2, "$29.99", subtotal "$59.98". SKU-B line: "Denim Jacket", "Medium", qty 1, "$89.99", subtotal "$89.99". Cart total: "$149.97".

### TS-001-015 — Out-of-stock SKU shows disabled Add to Cart button
- **Preconditions:** storefront-app loaded. Product detail page displayed for product with SKU-C where `stock_level = 0`.
- **Action:** User views SKU-C variant on product detail page.
- **Expected:** "Add to Cart" button for SKU-C is disabled (not clickable). "Out of stock" indicator displayed next to SKU-C. Clicking the disabled button does not add item to cart.

---

## UI Flow

```
[Product Detail Page]
  → each variant row has "Add to Cart" button
  → click "Add to Cart" (in-stock) → item added to cart store → cart icon count updates
  → out-of-stock variant → button disabled + "Out of stock" label

[Cart View: /cart]
  → list of cart items (product name, variant label, qty, unit price, line subtotal)
  → cart total at bottom
  → "Checkout" button (rendered here but checkout logic is in WP-004-FE)
```

### Step 1 — Add to Cart (on Product Detail Page)

Each SKU variant row on the product detail page (built in WP-001-FE) gets a functional "Add to Cart" button:

- **In-stock variant** (`stock_level > 0`): Button is enabled. On click, calls `addItem()` on the cart store with `{ sku_id, product_name, variant_label, price_minor, image_url }`.
- **Out-of-stock variant** (`stock_level === 0`): Button is disabled. "Out of stock" indicator shown. Clicking does nothing.

After adding, a brief visual confirmation (e.g., button text changes to "Added!" for 1 second, or a toast).

### Step 2 — Cart Icon (Header)

A cart icon in the app header shows the total item count (sum of all quantities). Reads from `cartStore.totalItems()`. Links to `/cart`.

### Step 3 — Cart Page (`/cart`)

Displays all items in the cart:

| Column | Source |
|---|---|
| Product name | `item.product_name` |
| Variant | `item.variant_label` |
| Quantity | `item.quantity` |
| Unit price | `formatPrice(item.price_minor)` |
| Subtotal | `formatPrice(item.price_minor * item.quantity)` |

Cart total at the bottom: `formatPrice(cartStore.totalPrice())`.

A "Checkout" button is rendered below the cart total (navigation to `/checkout` is wired, but checkout logic is in WP-004-FE).

**CTA:** "Checkout" → `/checkout`

---

## Error Handling

### Attempting to add out-of-stock item

**When:** SKU has `stock_level === 0`.
**Display:** "Add to Cart" button is disabled. "Out of stock" label visible.
**Behavior:** Click is a no-op. No item added to cart.

### Cart icon with zero items

**When:** Cart is empty.
**Display:** Cart icon shows no badge or shows "0". Navigating to `/cart` shows empty cart state (handled by WP-003-FE).

---

## State Management

### Cart Store (Zustand)

Create `src/stores/cart-store.ts` using Zustand:

```typescript
interface CartItem {
  sku_id: string;
  product_name: string;
  variant_label: string;
  price_minor: number;
  quantity: number;
  image_url: string | null;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: Omit<CartItem, "quantity">) => void;
  removeItem: (skuId: string) => void;
  updateQuantity: (skuId: string, quantity: number) => void;
  clearCart: () => void;
  totalItems: () => number;
  totalPrice: () => number;
}
```

**`addItem` behavior:**
- If a `CartItem` with the same `sku_id` already exists, increment its `quantity` by 1.
- Otherwise, add a new entry with `quantity: 1`.

**`totalItems`:** Sum of all `item.quantity` values.

**`totalPrice`:** Sum of all `item.price_minor * item.quantity` values.

**Note:** The Zustand store is created in this WP **without** persistence middleware. Persistence (`localStorage` via Zustand `persist`) is added in WP-003-FE. The store shape defined here is the canonical shape — WP-003-FE wraps it with `persist`, it does not change the interface.

### Price Formatting

Reuse `formatPrice` from `src/utils/format.ts` (created in WP-001-FE):
- `formatPrice(2999)` → `"$29.99"`

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/cart` | `CartPage` | Displays cart contents, total, checkout button |

No route guards — cart page is public.

The `/products/:productId` route (from WP-001-FE) is enhanced with functional "Add to Cart" buttons.

---

## Related Contracts

- `workspaces/inventory-service/docs/api/openapi.json` — `GET /products/{productId}` (for `stock_level` data used to enable/disable Add to Cart)

**Schemas used:**
- `SKU` — `id`, `label`, `price_minor`, `stock_level` (from product detail response)

**Mock strategy:** Use MSW against the inventory-service OpenAPI spec during development and testing. Same MSW setup from WP-001-FE is reused.

---

## Implementation Notes

### Files to Create

1. `src/stores/cart-store.ts` — Zustand cart store (shape defined above)
2. `src/components/CartIcon.tsx` — Header cart icon with item count badge
3. `src/pages/CartPage.tsx` — Cart contents display
4. `src/components/CartItemRow.tsx` — Single cart line item (name, variant, qty, price, subtotal)
5. `src/components/CartTotal.tsx` — Cart total display

### Files to Modify (from WP-001-FE)

6. `src/components/VariantList.tsx` (or equivalent) — Wire "Add to Cart" button to `cartStore.addItem()`. Disable button when `stock_level === 0`.
7. `src/App.tsx` or layout — Add `CartIcon` to the app header.
8. Router config — Add `/cart` route.

### Implementation Order

1. Create Zustand cart store (`cart-store.ts`)
2. Create `CartIcon` component (reads `totalItems()` from store)
3. Add `CartIcon` to app header/layout
4. Wire "Add to Cart" buttons on product detail page to `cartStore.addItem()`
5. Disable "Add to Cart" for out-of-stock variants (`stock_level === 0`)
6. Create `CartPage` component with item list and total
7. Create `CartItemRow` component
8. Create `CartTotal` component
9. Add `/cart` route
10. Tests for all TS scenarios (TS-001-008, TS-001-009, TS-001-010, TS-001-015)
11. Run DoD checklist

---

## Dependency Info

**Wave 2** — depends on WP-001-FE (product pages provide the product detail page and variant rendering).

WP-003-FE depends on this WP (adds cart editing and persistence to the store created here).

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-008, TS-001-009, TS-001-010, TS-001-015)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
