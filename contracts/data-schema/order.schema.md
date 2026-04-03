# Data Schema: Order

**Service:** order-service
**Last updated:** 2026-04-03

---

## Overview

The `orders` table is the central record of a customer's or guest's intent to purchase. It holds the full snapshot of what was ordered, at what price, and where it should be shipped — capturing values at the moment of placement so historical records remain accurate even if prices or products change later.

---

## Table: `orders`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `tenant_id` | `UUID` | NO | Foreign key → tenants. Scopes the order to a merchant. |
| `reference` | `VARCHAR(30)` | NO | Human-readable order reference (e.g. `ORD-20240415-A3K9`). Unique per tenant. |
| `status` | `ENUM` | NO | `pending`, `confirmed`, `processing`, `shipped`, `delivered`, `cancelled`. |
| `customer_id` | `UUID` | YES | Foreign key → customers. NULL for guest orders. |
| `guest_email` | `VARCHAR(254)` | YES | Email provided during guest checkout. NULL for authenticated orders. |
| `shipping_address` | `JSONB` | NO | Snapshot of the shipping address at time of order. |
| `shipping_method_id` | `UUID` | NO | Foreign key → shipping_methods. |
| `shipping_cost_minor` | `INT` | NO | Shipping cost in minor currency units at time of order. |
| `subtotal_minor` | `INT` | NO | Sum of all order line subtotals in minor currency units. |
| `tax_minor` | `INT` | NO | Tax amount in minor currency units. |
| `total_minor` | `INT` | NO | Grand total: subtotal + shipping + tax. |
| `payment_intent_id` | `VARCHAR(100)` | YES | External payment gateway intent ID. NULL until payment step. |
| `idempotency_key` | `VARCHAR(64)` | YES | Client-supplied idempotency key for safe retries. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp of order creation. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp of last status update. |

**Constraints:**
- `UNIQUE (tenant_id, reference)`
- `CHECK (customer_id IS NOT NULL OR guest_email IS NOT NULL)` — every order has either a customer or a guest email.
- `UNIQUE (tenant_id, idempotency_key)` — prevents duplicate order placement.

---

## Table: `order_lines`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `order_id` | `UUID` | NO | Foreign key → orders. |
| `sku_id` | `UUID` | NO | SKU ID snapshotted from inventory-service at order time. |
| `product_name` | `VARCHAR(200)` | NO | Product name snapshot. |
| `variant_label` | `VARCHAR(200)` | YES | Variant description snapshot (e.g. "Blue / XL"). |
| `quantity` | `INT` | NO | Units ordered. Minimum 1. |
| `unit_price_minor` | `INT` | NO | Price per unit in minor currency units at time of order. |
| `subtotal_minor` | `INT` | NO | `quantity × unit_price_minor`. |

**Constraints:**
- `quantity >= 1`
- `unit_price_minor >= 0`

---

## Indexes

```sql
-- Fast tenant-scoped order lookups
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);

-- Guest order lookup by email (e.g. post-purchase account linking)
CREATE INDEX idx_orders_guest_email ON orders (guest_email) WHERE guest_email IS NOT NULL;

-- Order lines by parent order
CREATE INDEX idx_order_lines_order_id ON order_lines (order_id);
```

---

## Status Transitions

```
pending ──► confirmed ──► processing ──► shipped ──► delivered
   │              │
   └──────────────┴──────────────────────────────────► cancelled
```

- `pending`: order record created, payment not yet captured.
- `confirmed`: payment captured, stock deducted.
- `processing`: warehouse has picked up the order.
- `shipped`: carrier has collected the parcel; tracking number available.
- `delivered`: carrier confirms delivery.
- `cancelled`: order voided; refund issued if payment was captured.

Cancelled is a terminal state. Orders in `shipped` or `delivered` status cannot be cancelled via API — a manual refund process applies.

---

## Related Schemas

- `contracts/data-schema/product.schema.md` — SKU and product definitions in inventory-service.
