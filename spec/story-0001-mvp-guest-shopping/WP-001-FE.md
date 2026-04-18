# WP-001-FE: storefront-app — Product Catalogue UI

**Feature:** story-0001-mvp-guest-shopping
**Target workspace:** `workspaces/storefront-app`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the product catalogue pages in storefront-app: paginated product list with in-stock filter, product detail with variant/SKU display, empty catalogue state, and product not found handling. This WP also scaffolds the storefront-app workspace (package.json, Vite config, React Router setup, TanStack Query provider, MSW setup).

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-001 — Product catalogue page

The guest sees a paginated list of products from the inventory-service catalogue. Each product card shows the product name, price, and thumbnail image. The guest can filter the list to show only products with available stock.

**Testable:** `GET /products` returns `200` with a `ProductPage` containing `data` (product array) and `meta` (pagination). `GET /products?in_stock_only=true` returns only products where at least one SKU has `stock_level > 0`. UI renders a product grid with name, price, and image for each product.

---

### AC-002 — Product detail page

The guest views an individual product page showing the product name, description, all available variants (SKUs) with their prices, and stock availability per variant.

**Testable:** `GET /products/{productId}` returns `200` with a `Product` containing `name`, `description`, and `skus` array where each SKU has `label`, `price_minor`, and `stock_level`. UI displays all fields and indicates which variants are in stock.

---

### AC-003 — Empty catalogue

When the catalogue contains no products, the catalogue page shows an informative empty state.

**Testable:** When `GET /products` returns `200` with an empty `data` array, then the UI displays "No products available" instead of an empty grid.

---

### AC-004 — Product not found

When the guest navigates to a URL for a product that does not exist, a not-found page is displayed.

**Testable:** `GET /products/{nonExistentId}` returns `404`. UI displays a "Product not found" message with a link back to the catalogue.

---

## Test Scenarios (from TS-001)

### TS-001-001 — Paginated product list returns products with metadata
- **Preconditions:** MSW mock: `GET /products?page=1&per_page=20` returns 200 with `ProductPage` — 20 products in `data`, `meta: { total: 25, page: 1, per_page: 20 }`. Each product has `id`, `name`, `image_url`, `skus` with `label`, `price_minor`, `stock_level`.
- **Action:** User navigates to `/products` (or `/`).
- **Expected:** Product grid renders 20 product cards. Each card shows product name, price (formatted from `price_minor`), and thumbnail image. Pagination controls are visible.

### TS-001-002 — In-stock filter returns only products with available stock
- **Preconditions:** MSW mock: `GET /products?in_stock_only=true` returns 200 with 3 products (all with at least one SKU where `stock_level > 0`).
- **Action:** User enables the "In stock only" toggle filter.
- **Expected:** Product grid updates to show exactly 3 products. All displayed products have at least one in-stock SKU.

### TS-001-003 — Pagination second page returns remaining products
- **Preconditions:** MSW mock: `GET /products?page=2&per_page=20` returns 200 with 5 products, `meta: { total: 25, page: 2, per_page: 20 }`.
- **Action:** User clicks "Next page" (or page 2) pagination control.
- **Expected:** Product grid updates to show 5 product cards. Pagination reflects page 2 of 2.

### TS-001-004 — Product detail returns product with all SKU variants
- **Preconditions:** MSW mock: `GET /products/{productId}` returns 200 with `{ name: "Classic T-Shirt", description: "A comfortable cotton tee", image_url: "https://example.com/tshirt.jpg", skus: [{ id: "sku-s", label: "Small", price_minor: 2999, stock_level: 10 }, { id: "sku-m", label: "Medium", price_minor: 2999, stock_level: 5 }, { id: "sku-l", label: "Large", price_minor: 3499, stock_level: 0 }] }`.
- **Action:** User navigates to `/products/{productId}`.
- **Expected:** Page displays "Classic T-Shirt", description "A comfortable cotton tee", product image. Variant list shows 3 SKUs with label, price (formatted), and stock status. SKU "Large" shows as out of stock.

### TS-001-005 — Product detail shows variant stock levels accurately
- **Preconditions:** MSW mock: `GET /products/{productId}` returns 200 with 2 SKUs: SKU-A `stock_level: 10`, SKU-B `stock_level: 0`.
- **Action:** User navigates to `/products/{productId}`.
- **Expected:** SKU-A displays as in stock. SKU-B displays as out of stock with "Out of stock" indicator.

### TS-001-006 — Empty catalogue returns empty data array
- **Preconditions:** MSW mock: `GET /products` returns 200 with `{ data: [], meta: { total: 0, page: 1, per_page: 20 } }`.
- **Action:** User navigates to `/products`.
- **Expected:** UI displays "No products available" message instead of an empty grid. No product cards rendered.

### TS-001-007 — Non-existent product returns 404
- **Preconditions:** MSW mock: `GET /products/{nonExistentId}` returns 404 with `{ code: "NOT_FOUND", message: "Product not found" }`.
- **Action:** User navigates to `/products/{nonExistentId}`.
- **Expected:** UI displays "Product not found" message. A link back to the catalogue (`/products`) is visible and functional.

---

## UI Flow

```
[Product List Page: /products]
  → grid of product cards (name, price, thumbnail)
  → pagination controls (prev/next, page numbers)
  → "In stock only" toggle filter
  → click product card → [Product Detail Page: /products/:productId]
  → empty state: "No products available" (when data is empty)

[Product Detail Page: /products/:productId]
  → product name, description, image
  → variant/SKU list (label, price, stock indicator)
  → "Add to Cart" button per variant (handled by WP-002-FE)
  → 404 state: "Product not found" + link back to /products
```

### Step 1 — Product List Page (`/products`)

The user lands on the product catalogue. A grid of product cards is displayed with pagination controls at the bottom. Each card shows:

- **Product name** — from `Product.name`
- **Price** — from the first SKU's `price_minor`, converted to display format (e.g., 2999 → "$29.99")
- **Thumbnail image** — from `Product.image_url`. If `null`, show a placeholder image.

A toggle filter labeled "In stock only" is visible above or beside the grid. When enabled, the API call includes `?in_stock_only=true`.

Pagination controls show current page, total pages (calculated from `meta.total / meta.per_page`), and prev/next buttons.

**CTA:** Click a product card → navigates to `/products/:productId`

### Step 2 — Product Detail Page (`/products/:productId`)

The user sees the full product details:

- **Product name** — from `Product.name`
- **Description** — from `Product.description`
- **Product image** — from `Product.image_url`. If `null`, show a placeholder.
- **Variant list** — each SKU displayed as a row/card with:
  - `label` (e.g., "Small", "Blue / XL")
  - `price_minor` formatted as currency (e.g., 2999 → "$29.99")
  - Stock indicator: "In stock" if `stock_level > 0`, "Out of stock" if `stock_level === 0`
  - "Add to Cart" button — **disabled** when `stock_level === 0` (this button's behavior is implemented in WP-002-FE; this WP renders the button and disables it for out-of-stock variants)

---

## Error Handling

### Empty catalogue

**When:** `GET /products` returns 200 with empty `data` array.
**Display:** "No products available" message centered in the grid area.
**Behavior:** No product cards rendered. Pagination hidden. Filter still functional.

### Product not found

**When:** `GET /products/{productId}` returns 404.
**Display:** "Product not found" message with a "Back to catalogue" link.
**Behavior:** Link navigates to `/products`.

### Network error on product list

**When:** `GET /products` fails (network error, 5xx).
**Display:** "Unable to load products. Please try again." with a "Retry" button.
**Behavior:** Retry button triggers a TanStack Query refetch.

### Network error on product detail

**When:** `GET /products/{productId}` fails (network error, 5xx).
**Display:** "Unable to load product. Please try again." with a "Retry" button.
**Behavior:** Retry button triggers a TanStack Query refetch.

---

## State Management

- **Server state:** TanStack Query manages all API data. No local duplication.
  - `useProducts(page, perPage, inStockOnly)` — fetches `GET /products` with query params.
  - `useProduct(productId)` — fetches `GET /products/{productId}`.
- **UI state:** Component-local state only for this WP.
  - `currentPage` (number) — current pagination page.
  - `inStockOnly` (boolean) — whether the in-stock filter is active.
- **No persistent state** in this WP. Product data is fetched fresh on each page load. TanStack Query handles caching.

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/` | Redirect to `/products` | App entry point |
| `/products` | `ProductListPage` | Paginated product grid with filter |
| `/products/:productId` | `ProductDetailPage` | Product details with variant list |

No route guards for catalogue pages — these are public.

---

## Related Contracts

- `contracts/api/inventory-service.openapi.yaml` — `GET /products` (listProducts), `GET /products/{productId}` (getProduct)

**Schemas used:**
- `Product` — `id`, `name`, `description`, `image_url` (nullable), `skus`, `created_at`
- `SKU` — `id`, `label`, `price_minor`, `stock_level`
- `ProductPage` — `data` (array of `Product`), `meta` (`total`, `page`, `per_page`)
- `ErrorResponse` — `code`, `message`

**Headers:** All requests must include `X-Tenant-ID` header (UUID). For development/testing, use a hardcoded test tenant ID in the API client.

**Mock strategy:** Use MSW against the inventory-service OpenAPI spec above during development and testing. Switch to the real inventory-service backend URL once WP-001-BE is deployed.

---

## Implementation Notes

### Workspace Scaffold

This WP scaffolds the `storefront-app` workspace from scratch:

1. **package.json** — dependencies:
   - `react`, `react-dom` (^18)
   - `react-router-dom` (^6) — client-side routing
   - `@tanstack/react-query` (^5) — data fetching and caching
   - `zustand` (^4) — state management (used by later WPs, install now)
   - Dev dependencies: `typescript`, `vite`, `@vitejs/plugin-react`, `vitest`, `@testing-library/react`, `@testing-library/jest-dom`, `@testing-library/user-event`, `msw` (^2), `jsdom`
   - Linting: `@biomejs/biome`

2. **Vite config** (`vite.config.ts`) — React plugin, test config for Vitest with `jsdom` environment.

3. **TypeScript config** (`tsconfig.json`) — strict mode, `jsx: "react-jsx"`, path aliases optional.

4. **Biome config** (`biome.json`) — default recommended rules.

5. **React Router setup** (`src/main.tsx`) — `BrowserRouter` wrapping `App`, `QueryClientProvider` wrapping routes.

6. **MSW setup** (`src/mocks/`) — `handlers.ts` for inventory-service endpoints, `browser.ts` for dev, `server.ts` for tests.

### API Client

Create `src/api/client.ts`:
- Base fetch wrapper that adds `X-Tenant-ID` header to every request.
- Tenant ID sourced from environment variable (`VITE_TENANT_ID`) or hardcoded for tests.
- Base URL sourced from environment variable (`VITE_INVENTORY_API_URL`) with fallback to MSW in dev/test.

### Price Formatting

Create a utility `src/utils/format.ts`:
- `formatPrice(priceMinor: number): string` — converts cents to display (e.g., `2999` → `"$29.99"`).
- Use `Intl.NumberFormat` with `style: "currency"`, `currency: "USD"`.

### Image Placeholder

- When `Product.image_url` is `null`, render a placeholder (a grey box with a package icon, or a CSS-based placeholder).
- Do not reference external placeholder services.

### Implementation Order

1. Scaffold workspace (package.json, Vite, React Router, TanStack Query, Biome, MSW)
2. API client module (fetch wrapper with `X-Tenant-ID` header)
3. Price formatting utility (`formatPrice`)
4. TypeScript types for `Product`, `SKU`, `ProductPage` (from OpenAPI schema)
5. MSW handlers for `GET /products` and `GET /products/{productId}`
6. `useProducts` TanStack Query hook
7. `ProductListPage` component with product grid
8. `ProductCard` component (name, price, image/placeholder)
9. `Pagination` component (prev/next, page numbers)
10. In-stock filter toggle
11. `useProduct` TanStack Query hook
12. `ProductDetailPage` component
13. `VariantList` component (SKU rows with label, price, stock indicator, disabled Add to Cart button)
14. `EmptyState` component ("No products available")
15. `NotFound` component ("Product not found" + link back)
16. Tests for all TS scenarios (TS-001-001 through TS-001-007)
17. Run DoD checklist

---

## Dependency Info

**Wave 1** — can start immediately with MSW mocks. No backend dependency.

Switch `VITE_INVENTORY_API_URL` to the real inventory-service URL after WP-001-BE deploys.

WP-002-FE depends on this WP (product pages provide the "Add to Cart" entry point).

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-001 through TS-001-007)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
