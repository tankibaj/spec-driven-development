# WP-009-FE: Admin Order Detail UI — Frontend Work Package

**Feature:** 001-guest-checkout
**Target workspace:** `workspaces/admin-app`
**Status:** awaiting_review
**Generated:** 2026-04-11
**Depends on:** WP-007-FE (auth + shell layout must exist). Wave 5 (parallel with WP-008-FE). Switch to real BE after WP-008-BE deploys.

---

## Objective

Implement the admin order detail page in admin-app: full order view with line items table, shipping address block, order status and timestamps, notification status indicator (warning banner for failed email delivery), and a not-found state for non-existent orders.

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

### AC-028 — Order detail view

The admin views full order details including order lines, shipping address, order status, and timestamps.

**Testable:** `GET /orders/{orderId}` with valid admin JWT returns `200` with `Order` containing `lines` (each with `product_name`, `variant_label`, `quantity`, `unit_price`), `shipping_address`, `status`, `reference`, and `created_at`. UI displays all fields in a structured layout.

---

### AC-029 — Notification status indicator

When the confirmation email for an order has failed delivery, the order detail page shows a warning indicator. When the email was delivered successfully, no indicator is shown.

**Testable:** When the notification associated with an order has `status: failed`, the order detail UI shows a "Confirmation email not delivered" warning. When `status` is `sent` or `delivered`, no warning is displayed.

---

### AC-031 — Order not found

Navigating to a non-existent order shows a not-found state.

**Testable:** `GET /orders/{nonExistentId}` returns `404`. UI displays "Order not found" with a link back to the order list.

---

## Test Scenarios (from TS-001)

### TS-001-051 — Order detail renders full order with all fields
- **Preconditions:** MSW mock: `GET /orders/{orderId}` returns `200` with `Order` containing `reference: "ORD-20260411-A3K9"`, `status: "confirmed"`, 2 line items, `shipping_address: { line1: "123 Main St", city: "London", postal_code: "SW1A 1AA", country_code: "GB" }`, `total: 7497`, `created_at: "2026-04-11T10:30:00Z"`, `notification_status: "sent"`. User is fully authenticated.
- **Action:** User navigates to `/orders/{orderId}`.
- **Expected:** Page displays order reference "ORD-20260411-A3K9", status badge "confirmed", 2 line items in a table (each showing product_name, variant_label, quantity, unit_price formatted, subtotal formatted), shipping address block ("123 Main St, London, SW1A 1AA, GB"), total "$74.97", date "Apr 11, 2026". No notification warning banner.

### TS-001-052 — Failed notification shows warning banner
- **Preconditions:** MSW mock: `GET /orders/{orderId}` returns `200` with `notification_status: "failed"`. User is fully authenticated.
- **Action:** User navigates to `/orders/{orderId}`.
- **Expected:** Order detail page displays a yellow/orange warning banner: "Confirmation email not delivered". All other order fields render normally.

### TS-001-053 — Successful notification shows no warning
- **Preconditions:** MSW mock: `GET /orders/{orderId}` returns `200` with `notification_status: "sent"` (or `"delivered"`). User is fully authenticated.
- **Action:** User navigates to `/orders/{orderId}`.
- **Expected:** Order detail page does NOT display any notification warning banner. Order fields render normally.

### TS-001-055 — Non-existent order shows 404 state
- **Preconditions:** MSW mock: `GET /orders/{nonExistentId}` returns `404` with `ErrorResponse`. User is fully authenticated.
- **Action:** User navigates to `/orders/{nonExistentId}`.
- **Expected:** Page displays "Order not found" message. A link back to `/orders` ("Back to orders") is visible. No order detail content rendered.

---

## UI Flow

```
[Order List /orders]
  └── click row → /orders/:orderId → [Order Detail Page]
        ├── [Notification Banner] (if notification_status === "failed")
        ├── [Order Header] — reference + status badge + date
        ├── [Line Items Table]
        ├── [Shipping Address Block]
        ├── [Order Total]
        └── [Back Link] → /orders

[404 State]
        ├── "Order not found"
        └── [Back Link] → /orders
```

### Order Detail Page Layout

The page renders inside the shell layout's `<Outlet />` from WP-007-FE.

**Back navigation:** A "Back to orders" link at the top, navigating to `/orders`.

**Notification banner (conditional):**
- Shown ONLY when `notification_status === "failed"`.
- Banner style: yellow/orange background, warning icon.
- Text: "Confirmation email not delivered"
- Position: top of page, below back link, above order header.
- When `notification_status` is `"sent"`, `"delivered"`, `"queued"`, or `null`: no banner rendered.

**Order header section:**
- Order reference (large/prominent text, e.g., "ORD-20260411-A3K9").
- Status badge (styled chip/badge showing the status label, e.g., "Confirmed").
- Date: `created_at` formatted as human-readable (e.g., "Apr 11, 2026, 10:30 AM").

**Line items table:**

| Column | Source field | Format |
|---|---|---|
| Product | `product_name` | Plain text |
| Variant | `variant_label` | Plain text (e.g., "Blue / XL") |
| Qty | `quantity` | Integer |
| Unit Price | `unit_price` | Currency from minor units (e.g., 2999 → "$29.99") |
| Subtotal | `subtotal` | Currency from minor units (e.g., 5998 → "$59.98") |

**Shipping address block:**
Display as a formatted address:
```
123 Main St
Apt 4B                  ← line2 (omit if null/empty)
London, SW1A 1AA
GB
```
Format: `line1`, `line2` (if present), `city, postal_code`, `country_code` (or `state, postal_code` if state is present).

**Order total:**
- Display `total` formatted from minor units (e.g., 7497 → "$74.97").
- Label: "Total" with the formatted amount.

---

## Error Handling

### Order not found (404)

**When:** `GET /orders/{orderId}` returns `404`.
**Display:** "Order not found" message with a "Back to orders" link pointing to `/orders`.
**Behavior:** No order detail content rendered. Only the not-found message and back link.

### API error on order detail fetch

**When:** `GET /orders/{orderId}` returns a non-200/non-404 response (e.g., 500, network error).
**Display:** "Failed to load order details. Please try again." with a "Retry" button.
**Behavior:** TanStack Query handles retry automatically (default 3 retries). Show error state after retries exhausted.

### 401 on order detail fetch

**When:** `GET /orders/{orderId}` returns `401` (token expired).
**Display:** No error message — silent redirect.
**Behavior:** Auth store's `logout()` is called. User redirected to `/login`.

### 403 on order detail fetch

**When:** `GET /orders/{orderId}` returns `403` (pre-MFA token).
**Display:** No error message — silent redirect.
**Behavior:** User redirected to `/mfa`.

---

## State Management

### Query State (TanStack Query)

Create a custom hook (e.g., `useOrder`) that wraps `useQuery`:

```typescript
function useOrder(orderId: string) {
  return useQuery({
    queryKey: ["orders", orderId],
    queryFn: () => fetchOrder(orderId),
    retry: (failureCount, error) => {
      // Don't retry on 404 — it's a definitive "not found"
      if (error.status === 404) return false;
      return failureCount < 3;
    },
  });
}
```

- Query key includes the `orderId` for per-order caching.
- On 404: do not retry, immediately show not-found state.
- `staleTime`: 60 seconds (single order detail is viewed, not refreshed frequently).

### No persistent state needed

Order detail is read-only. No form state, no editable fields. All state is managed by TanStack Query. Navigating away discards the query cache after `gcTime` (default 5 minutes).

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `/orders/:orderId` | `OrderDetailPage` | Requires full auth (guarded by `RequireAuth` from WP-007-FE). `orderId` extracted from URL params. |

The route is already defined in WP-007-FE's router setup with a placeholder component. This WP replaces the placeholder with the full implementation.

---

## Price Formatting

Reuse the `formatPrice` utility created in WP-008-FE (`src/lib/format.ts`):

```typescript
function formatPrice(minorUnits: number): string {
  return `$${(minorUnits / 100).toFixed(2)}`;
}
```

Apply to: `unit_price` and `subtotal` in line items table, and `total` in the order summary.

If WP-008-FE has not been implemented yet (parallel work), create the utility in this WP. It is a single function with no dependencies.

---

## Date Formatting

Format `created_at` (ISO 8601 string) to a human-readable format:

```typescript
function formatDate(isoString: string): string {
  return new Intl.DateTimeFormat("en-US", {
    year: "numeric",
    month: "short",
    day: "numeric",
    hour: "numeric",
    minute: "2-digit",
  }).format(new Date(isoString));
}
```

Example: `"2026-04-11T10:30:00Z"` → `"Apr 11, 2026, 10:30 AM"`.

Place in `src/lib/format.ts` (shared with WP-008-FE's `formatPrice`).

---

## Related Contracts

- `workspaces/order-service/docs/api/openapi.json` — `GET /orders/{orderId}`, `Order`, `OrderLine`, `Address` schemas

### Key Schema Reference

**GET /orders/{orderId}:**
- Path param: `orderId` (UUID)
- Headers: `Authorization: Bearer {jwt}`, `X-Tenant-ID: {tenantId}`
- `200`: `Order` object
- `401`: missing/invalid auth
- `403`: insufficient permissions (pre-MFA)
- `404`: order not found

**Order (full detail):**
```json
{
  "id": "uuid",
  "reference": "ORD-20260411-A3K9",
  "status": "confirmed",
  "guest_email": "grace@example.com",
  "lines": [
    {
      "sku_id": "uuid",
      "product_name": "Classic T-Shirt",
      "variant_label": "Small",
      "quantity": 2,
      "unit_price": 2999,
      "subtotal": 5998
    }
  ],
  "subtotal": 5998,
  "shipping_cost": 599,
  "tax": 900,
  "total": 7497,
  "shipping_address": {
    "line1": "123 Main St",
    "line2": null,
    "city": "London",
    "state": null,
    "postal_code": "SW1A 1AA",
    "country_code": "GB"
  },
  "notification_id": "uuid | null",
  "notification_status": "sent | delivered | failed | queued | null",
  "created_at": "2026-04-11T10:30:00Z"
}
```

**OrderLine:**
```json
{
  "sku_id": "uuid",
  "product_name": "string",
  "variant_label": "string",
  "quantity": 2,
  "unit_price": 2999,
  "subtotal": 5998
}
```

**Address:**
```json
{
  "line1": "string (required)",
  "line2": "string | null",
  "city": "string (required)",
  "state": "string | null",
  "postal_code": "string (required)",
  "country_code": "string 2-char ISO (required)"
}
```

**notification_status enum:** `queued`, `sent`, `delivered`, `failed`, or `null`.

**ErrorResponse (404):**
```json
{ "code": "NOT_FOUND", "message": "Order not found" }
```

**Mock strategy:** Use MSW against the OpenAPI spec above during development. Switch to the real order-service URL once WP-008-BE is deployed.

---

## MSW Handlers

Add the following MSW handler to the existing handlers file (`src/test/mocks/handlers.ts`):

### GET /orders/:orderId

- If `orderId` matches a known mock order ID: return `200` with a full `Order` object including line items, shipping address, and `notification_status`.
- If `orderId` does not match: return `404` with `{ "code": "NOT_FOUND", "message": "Order not found" }`.

**Mock order fixtures** — create at least 3 fixtures for testing:

1. **Order with successful notification:** `notification_status: "sent"`, 2 line items, full shipping address.
2. **Order with failed notification:** `notification_status: "failed"`, 1 line item, full shipping address.
3. **Non-existent order ID:** returns 404.

Example mock order:
```json
{
  "id": "order-001",
  "reference": "ORD-20260411-A3K9",
  "status": "confirmed",
  "guest_email": "grace@example.com",
  "lines": [
    {
      "sku_id": "sku-001",
      "product_name": "Classic T-Shirt",
      "variant_label": "Small",
      "quantity": 2,
      "unit_price": 2999,
      "subtotal": 5998
    },
    {
      "sku_id": "sku-002",
      "product_name": "Denim Jacket",
      "variant_label": "Medium",
      "quantity": 1,
      "unit_price": 8999,
      "subtotal": 8999
    }
  ],
  "subtotal": 14997,
  "shipping_cost": 599,
  "tax": 0,
  "total": 15596,
  "shipping_address": {
    "line1": "123 Main St",
    "line2": null,
    "city": "London",
    "state": null,
    "postal_code": "SW1A 1AA",
    "country_code": "GB"
  },
  "notification_id": "notif-001",
  "notification_status": "sent",
  "created_at": "2026-04-11T10:30:00Z"
}
```

---

## Implementation Order

1. Date formatting utility (`src/lib/format.ts` — add `formatDate` alongside `formatPrice` from WP-008-FE)
2. `useOrder` TanStack Query hook (`src/hooks/useOrder.ts`)
3. Line items table component (`src/components/orders/LineItemsTable.tsx`)
4. Shipping address display component (`src/components/orders/ShippingAddress.tsx`)
5. Notification status banner component (`src/components/orders/NotificationBanner.tsx`)
6. Order not-found component (`src/components/orders/OrderNotFound.tsx`)
7. Order detail page (`src/pages/OrderDetailPage.tsx` — compose all components)
8. MSW handlers for `GET /orders/:orderId` (update `src/test/mocks/handlers.ts`)
9. Tests for all TS scenarios (`src/test/pages/OrderDetailPage.test.tsx`)
10. Run Definition of Done checklist

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-051, TS-001-052, TS-001-053, TS-001-055)
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] `status.yaml` `phase_4.WP-009-FE.status` set to `done`
