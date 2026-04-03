# System Roles & Tenancy Model

> This document defines every role a principal can hold in the system, what they can and cannot do,
> and how multi-tenancy is structured. All Feature Specs that touch authorisation MUST reference this file.

---

## Tenancy Model

The platform is **multi-tenant**. Each tenant is an independent merchant operating their own storefront.

```
Platform
└── Tenant (Merchant)
    ├── Storefront (customer-facing)
    └── Admin Portal (internal)
```

- A **Tenant** owns its own catalogue, orders, customers, and settings.
- Data is strictly isolated between tenants. No cross-tenant data access is permitted at any layer.
- Platform-level roles (see below) have access across all tenants for operational purposes only.

---

## Role Definitions

### Customer-Facing Roles

| Role | Description | Scope |
|---|---|---|
| `guest` | An unauthenticated shopper. Has a short-lived Guest Session. Cannot access account features. | Tenant |
| `customer` | An authenticated shopper with a registered account. Can access order history, address book, and saved payments. | Tenant |

### Merchant / Admin Roles

| Role | Description | Scope |
|---|---|---|
| `merchant_owner` | The primary account holder for a tenant. Full access to all merchant features. Can invite and manage other admin users. | Tenant |
| `merchant_admin` | An invited admin user. Full access to order management, inventory, and reporting. Cannot manage billing or other admin users. | Tenant |
| `merchant_viewer` | Read-only access to the admin portal. Can view orders and reports but cannot modify data. | Tenant |
| `merchant_support` | Access to order detail and refund/cancel actions only. Intended for customer support agents. Cannot access inventory or reporting. | Tenant |

### Platform Roles (Internal Only)

| Role | Description | Scope |
|---|---|---|
| `platform_ops` | Internal operations team. Read access across all tenants. Can view orders, logs, and metrics for any tenant for debugging purposes. Cannot modify tenant data without an audit trail. | Platform |
| `platform_admin` | Engineering and infrastructure team. Full platform access including system configuration, feature flags, and tenant provisioning. | Platform |

---

## Permission Matrix

| Action | `guest` | `customer` | `merchant_owner` | `merchant_admin` | `merchant_viewer` | `merchant_support` | `platform_ops` | `platform_admin` |
|---|---|---|---|---|---|---|---|---|
| Browse catalogue | ✅ | ✅ | ✅ | ✅ | ✅ | — | — | — |
| Place order | ✅ | ✅ | — | — | — | — | — | — |
| View own order history | ❌ | ✅ | — | — | — | — | — | — |
| View all tenant orders | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ (read) | ✅ |
| Cancel / refund order | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ |
| Manage inventory | ❌ | ❌ | ✅ | ✅ | ✅ (read) | ❌ | ✅ (read) | ✅ |
| Manage admin users | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| View reports | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ (read) | ✅ |
| Configure tenant settings | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Authentication

- **Customers and Guests:** Storefront auth. Customers use email + password or OAuth (Google, Apple). Guests use a server-side session cookie.
- **Merchant Admins:** Separate admin auth. Email + password with MFA required for `merchant_owner` and `merchant_admin`.
- **Platform Roles:** Internal SSO only. Not exposed through any public-facing endpoint.

---

## Authorisation Conventions for Specs

When writing acceptance criteria that involve permissions:

1. Always name the role explicitly (e.g., "a `merchant_support` user").
2. Reference this file in the FS if a new or changed permission is introduced.
3. If a role boundary needs to change, propose the change here first and get it reviewed before writing the FS.
