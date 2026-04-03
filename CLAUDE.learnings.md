# CLAUDE.learnings.md — Institutional Memory

> Append-only. One learning per line, prefixed with date. Do not remove existing entries.
> Read at the start of every session. Append when you discover something future sessions should know.

---

## Learnings

- 2026-04-03: Initial repo scaffolded. Domain: e-commerce / order management. Workspaces: order-service, inventory-service, notification-service (backend); storefront-app, admin-app (frontend).
- 2026-04-03: "Basket" is a deprecated synonym for "Cart" — use "Cart" in all specs (see glossary.md).
- 2026-04-03: Guest session token must be stored in component state only, never in localStorage (ADR-001). Flag this if any WP suggests persistent storage.
- 2026-04-03: Notification dispatch after order placement is fire-and-forget (async). A notification failure must NOT roll back the order. This is an intentional design choice.
- 2026-04-03: Stock reservation expires after 15 minutes. This TTL is enforced by the inventory-service background job, not by the order-service.
- 2026-04-03: Order Reference format: `ORD-YYYYMMDD-{4 uppercase alphanumeric}`. Always verify uniqueness per tenant before persisting; retry up to 3× on collision.
