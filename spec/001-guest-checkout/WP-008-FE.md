# WP-008-FE: Admin Order List UI — Frontend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/admin-app`
**Status:** awaiting_review
**Generated:** 2026-04-11
**Depends on:** WP-007-FE (auth + shell layout must exist). Wave 5 — can mock with MSW. Switch to real BE after WP-007-BE deploys.

---

## Objective

Implement the admin order list page in admin-app: a paginated order table with free-text search (by reference or email), status filter dropdown, pagination controls, and an informative empty state. This page lives inside the shell layout scaffolded by WP-007-FE and is guarded by full authentication.

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Tech Stack

| Concern | Tool |
|---|---|
| Language | TypeScript (strict mode) |
| UI framework | React 18+ |
| Build tool | Vite |
| Routing | React Router v6+ |
| Data fetching | TanStack Query (React Query) v5+ |
| Auth state | Zustand (from WP-007-FE) |
| Testing | Vitest + React Testing Library |
| API mocking | MSW (Mock Service Worker) v2+ |
| Linting/Formatting | Biome |

---

## Acceptance Criteria (verbatim from FS-001)

### AC-025 — Paginated order list

An authenticated admin user sees a paginated list of orders scoped to their tenant.

**Testable:** `GET /orders` with valid admin JWT and `X-Tenant-ID` returns `200` with an `OrderPage` containing `data` (array of `Order` objects) and `meta` (`total`, `page`, `per_page`). UI renders an order table with columns for reference, status, customer email, date, and total.

---

### AC-026 — Order search

The admin can search orders by order reference or customer email.

**Testable:** `GET /orders?q={searchTerm}` *(search parameter to be added to contract)* returns orders where `reference` or `guest_email` contains the search term. UI: the admin types in a search box and the order list updates to show matching results.

---

### AC-027 — Order status filter

The admin can filter the order list by order status.

**Testable:** `GET /orders?status=confirmed` returns only orders with `status: confirmed`. UI: the admin selects a status from a dropdown and the list updates to show only matching orders.

---

### AC-030 — Empty order list

When no orders exist or no orders match the current search/filter criteria, an informative empty state is shown.

**Testable:** When `GET /orders` returns `200` with an empty `data` array, the UI displays "No orders found" with guidance to adjust filters or wait for incoming orders.

---

## Test Scenarios (from TS-001)

### TS-001-046 — Paginated order list returns orders with metadata
- **Preconditions:** MSW mock: `GET /orders?page=1&per_page=20` returns `200` with `OrderPage` containing 20 orders, `meta: { total: 25, page: 1, per_page: 20 }`. User is fully authenticated.
- **Action:** User navigates to `/orders`.
- **Expected:** Order table renders 20 rows. Each row shows reference, status (as badge), customer email, date (formatted), and total (formatted from minor units). Pagination shows page 1 of 2. Total count "25 orders" is displayed.

### TS-001-047 — Orders are scoped to the authenticated tenant
- **Preconditions:** MSW mock: `GET /orders` returns 5 orders (all from the authenticated tenant). Auth store has `user.tenantId = "tenant-a-id"`.
- **Action:** User navigates to `/orders`.
- **Expected:** API request includes `X-Tenant-ID: tenant-a-id` header. Table shows exactly 5 orders. `meta.total` is 5. (Tenant scoping is enforced by BE — FE verifies the header is sent.)

### TS-001-048 — Search by order reference returns matching orders
- **Preconditions:** MSW mock: `GET /orders?q=A3K9` returns `200` with 1 order (`reference: "ORD-20260411-A3K9"`).
- **Action:** User types "A3K9" into the search box.
- **Expected:** After debounce (300ms), table updates to show 1 order with reference "ORD-20260411-A3K9". Non-matching orders are not displayed.

### TS-001-049 — Search by guest email returns matching orders
- **Preconditions:** MSW mock: `GET /orders?q=grace@example.com` returns `200` with 2 orders, `meta: { total: 2 }`.
- **Action:** User types "grace@example.com" into the search box.
- **Expected:** After debounce, table updates to show 2 orders. `meta.total` is 2.

### TS-001-050 — Status filter returns only matching orders
- **Preconditions:** MSW mock: `GET /orders?status=confirmed` returns `200` with 6 orders, all with `status: "confirmed"`, `meta: { total: 6 }`.
- **Action:** User selects "Confirmed" from the status filter dropdown.
- **Expected:** Table updates to show 6 orders. All rows display "confirmed" status badge. `meta.total` is 6.

### TS-001-054 — Empty order list shows empty state
- **Preconditions:** MSW mock: `GET /orders` returns `200` with `data: []`, `meta: { total: 0 }`.
- **Action:** User navigates to `/orders`.
- **Expected:** Table is not rendered. Empty state displays "No orders found" with guidance text (e.g., "Adjust your filters or wait for incoming orders.").

---

## UI Flow

```
[Shell Layout]
  └── /orders → [Order List Page]
        ├── [Search Box] ─── debounce 300ms ─── refetch with ?q={term}
        ├── [Status Dropdown] ─── refetch with ?status={value}
        ├── [Order Table]
        │     └── click row → /orders/:orderId (handled by WP-009-FE)
        ├── [Pagination Controls] ─── refetch with ?page={n}
        └── [Empty State] (when data is empty)
```

### Order List Page Layout

The page renders inside the shell layout's `<Outlet />` from WP-007-FE.

**Top bar:** Search input (left) + Status filter dropdown (right).

**Search input:**
- Placeholder text: "Search by reference or email..."
- Debounced: 300ms delay after last keystroke before triggering API refetch.
- Sends the `q` query parameter to `GET /orders`.
- Clear button (×) to reset search.

**Status filter dropdown:**
- Options: "All statuses" (default, sends no `status` param), "Pending", "Confirmed", "Processing", "Shipped", "Delivered", "Cancelled".
- Values map to `OrderStatus` enum: `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`.
- Changing the dropdown triggers an immediate refetch.
- Selecting a status filter resets pagination to page 1.

**Order table columns:**

| Column | Source field | Format |
|---|---|---|
| Reference | `reference` | Plain text (e.g., "ORD-20260411-A3K9") |
| Status | `status` | Badge/chip with status label |
| Customer Email | `guest_email` | Plain text |
| Date | `created_at` | Formatted date (e.g., "Apr 11, 2026") |
| Total | `total` | Currency from minor units (e.g., 7497 → "$74.97") |

**Row click:** Navigate to `/orders/{orderId}` (WP-009-FE handles the detail page).

**Pagination controls:**
- Show "Page X of Y" or numbered page buttons.
- Previous / Next buttons. Disabled at boundaries.
- Total count display (e.g., "25 orders").
- Default `per_page: 20`.

**Empty state:**
- Displayed when `data` array is empty.
- Message: "No orders found"
- Guidance: "Adjust your filters or wait for incoming orders."
- No table rendered in empty state.

---

## Error Handling

### API error on order list fetch

**When:** `GET /orders` returns a non-200 response (e.g., 500, network error).
**Display:** "Failed to load orders. Please try again." with a "Retry" button.
**Behavior:** TanStack Query handles retry automatically (default 3 retries). Show error state after retries exhausted.

### 401 on order list fetch

**When:** `GET /orders` returns `401` (token expired).
**Display:** No error message — silent redirect.
**Behavior:** Auth store's `logout()` is called. User redirected to `/login`.

### 403 on order list fetch

**When:** `GET /orders` returns `403` (pre-MFA token).
**Display:** No error message — silent redirect.
**Behavior:** User redirected to `/mfa`.

---

## State Management

### Query State (TanStack Query)

Create a custom hook (e.g., `useOrders`) that wraps `useQuery`:

```typescript
interface UseOrdersParams {
  page: number;
  perPage: number;
  status?: OrderStatus;
  q?: string;
}

function useOrders(params: UseOrdersParams) {
  return useQuery({
    queryKey: ["orders", params],
    queryFn: () => fetchOrders(params),
  });
}
```

- Query key includes all filter/search/pagination params so TanStack Query caches and refetches correctly.
- `staleTime`: 30 seconds (orders don't change rapidly in admin view).
- On filter/search change: TanStack Query automatically refetches because the query key changes.

### Local Component State

- `searchTerm` (string): raw input value, updated on every keystroke.
- `debouncedSearch` (string): debounced value (300ms), used in the query key.
- `statusFilter` (OrderStatus | undefined): selected status, `undefined` = "All".
- `currentPage` (number): starts at 1, resets to 1 on filter/search change.

No persistent state needed — order list state is transient. Navigating away and returning reloads with default filters.

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/orders` | `OrderListPage` | Requires full auth (guarded by `RequireAuth` from WP-007-FE) |

The route is already defined in WP-007-FE's router setup with a placeholder component. This WP replaces the placeholder with the full implementation.

---

## Price Formatting

All prices in the API are in **minor currency units** (cents). Convert for display:

```typescript
function formatPrice(minorUnits: number): string {
  return `$${(minorUnits / 100).toFixed(2)}`;
}
```

Examples: `7497` → `"$74.97"`, `2999` → `"$29.99"`, `0` → `"$0.00"`.

Use this utility consistently across the order table's Total column. Extract to a shared utility (e.g., `src/lib/format.ts`) since WP-009-FE will also need it.

---

## Related Contracts

- `workspaces/order-service/docs/api/openapi.json` — `GET /orders` (query params: `status`, `q`, `page`, `per_page`), `OrderPage`, `Order`, `OrderStatus` schemas

### Key Schema Reference

**GET /orders query parameters:**
| Param | Type | Default | Description |
|---|---|---|---|
| `status` | `OrderStatus` enum | — | Filter by order status |
| `q` | string (max 200) | — | Free-text search across reference and guest_email |
| `page` | integer | 1 | Page number |
| `per_page` | integer | 20 (max 100) | Items per page |

**OrderPage:**
```json
{
  "data": [Order, ...],
  "meta": { "total": 25, "page": 1, "per_page": 20 }
}
```

**Order (list-relevant fields):**
```json
{
  "id": "uuid",
  "reference": "ORD-20260411-A3K9",
  "status": "confirmed",
  "guest_email": "grace@example.com",
  "total": 7497,
  "created_at": "2026-04-11T10:30:00Z"
}
```

**OrderStatus enum:** `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`

**Mock strategy:** Use MSW against the OpenAPI spec above during development. Switch to the real order-service URL once WP-007-BE is deployed.

---

## MSW Handlers

Add the following MSW handler to the existing handlers file from WP-007-FE (`src/test/mocks/handlers.ts`):

### GET /orders

Generate a mock dataset of orders. Support the following query param handling:

- `status`: if provided, filter mock orders to only those matching the status.
- `q`: if provided, filter mock orders where `reference` or `guest_email` contains the search term (case-insensitive).
- `page` and `per_page`: slice the filtered dataset accordingly.
- Return `OrderPage` response with correct `meta.total` (count of filtered results, not page size).

**Mock order factory** — generate orders with realistic data:
```json
{
  "id": "uuid",
  "reference": "ORD-20260411-XXXX",
  "status": "confirmed",
  "guest_email": "grace@example.com",
  "total": 7497,
  "created_at": "2026-04-11T10:30:00Z",
  "lines": [],
  "shipping_address": { "line1": "123 Main St", "city": "London", "postal_code": "SW1A 1AA", "country_code": "GB" }
}
```

For the empty state test, configure an MSW handler that returns `data: []` with `meta: { total: 0, page: 1, per_page: 20 }`.

---

## Implementation Order

1. Price formatting utility (`src/lib/format.ts` — `formatPrice` function)
2. `useOrders` TanStack Query hook (`src/hooks/useOrders.ts`)
3. Order table component (`src/components/orders/OrderTable.tsx`)
4. Search input with debounce (`src/components/orders/OrderSearch.tsx`)
5. Status filter dropdown (`src/components/orders/StatusFilter.tsx`)
6. Pagination component (`src/components/orders/Pagination.tsx`)
7. Empty state component (`src/components/orders/EmptyOrderList.tsx`)
8. Order list page (`src/pages/OrderListPage.tsx` — compose all components, manage local state)
9. MSW handlers for `GET /orders` (update `src/test/mocks/handlers.ts`)
10. Tests for all TS scenarios (`src/test/pages/OrderListPage.test.tsx`)
11. Run Definition of Done checklist

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-046, TS-001-047, TS-001-048, TS-001-049, TS-001-050, TS-001-054)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] `status.yaml` `phase_4.WP-008-FE.status` set to `done`
