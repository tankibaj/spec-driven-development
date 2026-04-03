# Data Schema: Product & SKU

**Service:** inventory-service
**Last updated:** 2026-04-03

---

## Overview

The product catalogue is managed by the inventory-service. A **Product** is a logical item with a name and description. Each Product has one or more **SKUs** representing specific purchasable variants (e.g. size, colour). Stock is tracked at the SKU level.

---

## Table: `products`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `tenant_id` | `UUID` | NO | Foreign key → tenants. |
| `name` | `VARCHAR(200)` | NO | Display name of the product. |
| `description` | `TEXT` | YES | Rich-text product description. |
| `is_active` | `BOOLEAN` | NO | Whether the product is visible in the storefront. Default: `true`. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC timestamp of record creation. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp of last update. |

**Constraints:**
- `UNIQUE (tenant_id, name)`

---

## Table: `skus`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `product_id` | `UUID` | NO | Foreign key → products. |
| `tenant_id` | `UUID` | NO | Foreign key → tenants (denormalised for fast tenant scoping). |
| `label` | `VARCHAR(200)` | NO | Human-readable variant description (e.g. "Blue / XL"). |
| `price_minor` | `INT` | NO | Current selling price in minor currency units. Minimum 0. |
| `is_active` | `BOOLEAN` | NO | Whether this variant is purchasable. Default: `true`. |
| `created_at` | `TIMESTAMPTZ` | NO | UTC. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC. |

**Constraints:**
- `UNIQUE (product_id, label)`
- `price_minor >= 0`

---

## Table: `stock_levels`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `sku_id` | `UUID` | NO | Primary key + foreign key → skus. One row per SKU. |
| `total` | `INT` | NO | Total physical units. Updated by inventory adjustments. |
| `reserved` | `INT` | NO | Units held by active reservations. |
| `updated_at` | `TIMESTAMPTZ` | NO | UTC timestamp of last stock movement. |

**Derived field (not stored):**
- `available = total - reserved` — units available for new orders.

**Constraints:**
- `total >= 0`
- `reserved >= 0`
- `reserved <= total`

---

## Table: `stock_reservations`

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key. |
| `order_id` | `UUID` | NO | The pending order this reservation belongs to. |
| `sku_id` | `UUID` | NO | Foreign key → skus. |
| `quantity` | `INT` | NO | Units reserved. |
| `status` | `ENUM` | NO | `active`, `converted`, `expired`, `released`. |
| `expires_at` | `TIMESTAMPTZ` | NO | UTC expiry time (15 minutes from creation). |
| `created_at` | `TIMESTAMPTZ` | NO | UTC. |

**Lifecycle:**
- `active`: reservation holds stock; counts toward `stock_levels.reserved`.
- `converted`: order confirmed; stock permanently deducted; `stock_levels.reserved` decremented, `stock_levels.total` decremented.
- `expired`: timer elapsed without order confirmation; `stock_levels.reserved` decremented; stock returned to available pool.
- `released`: order abandoned explicitly; same effect as expired.

---

## Indexes

```sql
-- Tenant catalogue browsing
CREATE INDEX idx_products_tenant_active ON products (tenant_id, is_active);

-- SKU lookups by product
CREATE INDEX idx_skus_product ON skus (product_id);

-- Active reservations for background expiry job
CREATE INDEX idx_stock_reservations_expires ON stock_reservations (expires_at)
  WHERE status = 'active';
```

---

## Related Schemas

- `contracts/data-schema/order.schema.md` — Order and order lines in order-service. Note that `sku_id`, `product_name`, and `variant_label` are **snapshotted** into order lines at placement time so that subsequent catalogue changes do not affect historical orders.
