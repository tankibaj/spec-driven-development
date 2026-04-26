# Cross-Service Dependency Graph

> Auto-generated 2026-04-26. Re-run `/depgraph` to refresh.

## Services

| Service | Stack | Outbound HTTP Calls |
|---|---|---|
| **inventory-service** | Python / FastAPI | None (leaf node) |
| **notification-service** | Python / FastAPI | None (SMTP only, leaf node) |
| **order-service** | Python / FastAPI | inventory-service, notification-service, Stripe |
| **storefront-app** | TypeScript / React | inventory-service, order-service |
| **admin-app** | TypeScript / React | order-service |

## Dependency Edges

| From | To | Method | Path | Source |
|---|---|---|---|---|
| order-service | inventory-service | POST | `/stock/reserve` | `src/clients/inventory_client.py` |
| order-service | inventory-service | POST | `/stock/deduct` | `src/clients/inventory_client.py` |
| order-service | inventory-service | POST | `/stock/reservations/{id}/release` | `src/clients/inventory_client.py` |
| order-service | notification-service | POST | `/notifications` | `src/clients/notification_client.py` |
| order-service | notification-service | GET | `/notifications/{notificationId}` | `src/clients/notification_client.py` |
| order-service | Stripe (external) | -- | `PaymentIntent.create` | `src/clients/stripe_client.py` |
| storefront-app | inventory-service | GET | `/products` | `src/api/client.ts` |
| storefront-app | inventory-service | GET | `/products/{productId}` | `src/api/client.ts` |
| storefront-app | order-service | POST | `/checkout/guest/sessions` | `src/api/order-client.ts` |
| storefront-app | order-service | GET | `/checkout/shipping-methods` | `src/api/order-client.ts` |
| storefront-app | order-service | POST | `/checkout/guest/orders` | `src/api/order-client.ts` |
| admin-app | order-service | POST | `/auth/login` | `src/stores/auth-store.ts` |
| admin-app | order-service | POST | `/auth/mfa/verify` | `src/stores/auth-store.ts` |
| admin-app | order-service | GET | `/orders` | `src/hooks/useOrders.ts` |
| admin-app | order-service | GET | `/orders/{orderId}` | `src/hooks/useOrder.ts` |

## Graph

```
                    +---------+
                    | Stripe  |  (external)
                    +----^----+
                         |
                         | PaymentIntent.create
                         |
+----------------+   +---+---------------+   +----------------------+
| storefront-app |-->| order-service     |-->| inventory-service    |
|   (React)      |   |   (FastAPI)       |   |   (FastAPI)          |
+-------+--------+   +---+---------------+   +----------------------+
        |                 |                          ^
        |                 |  POST /notifications     |
        |                 |  GET  /notifications/{id} |
        |                 v                          |
        |            +----+-------------------+      |
        |            | notification-service   |      |
        |            |   (FastAPI)            |      |
        |            +------------------------+      |
        |                                            |
        +--- GET /products, GET /products/{id} ------+
                (direct inventory-service calls)

+----------------+
| admin-app      |---> order-service
|   (React)      |     (auth + order queries)
+----------------+
```

## Circular Dependencies

None. The graph is a clean DAG.

## Blast Radius

| If this goes down | Severity | Impact |
|---|---|---|
| **inventory-service** | CRITICAL | Product browsing + checkout both break; order saga fails on stock reserve/deduct |
| **order-service** | CRITICAL | All admin access lost (including auth), no new orders from either frontend |
| **notification-service** | LOW | Orders still complete; confirmation emails silently fail (fire-and-forget pattern) |
| **storefront-app** | HIGH | Customers cannot browse or purchase; no backend cascade |
| **admin-app** | MEDIUM | Admins cannot view/manage orders; no cascade, no customer impact |

## Key Observations

1. **inventory-service has the widest blast radius** -- called directly by both storefront-app (catalogue reads) and order-service (stock writes). Two distinct consumer concerns on one service.
2. **order-service is the central hub** -- both frontend apps depend on it, and it is the only service with outbound inter-service calls.
3. **notification-service degrades gracefully** -- fire-and-forget pattern means the order saga continues even if notifications fail.
4. **storefront-app bypasses order-service** to hit inventory-service directly for product data -- worth noting for future API gateway or BFF considerations.
5. **No service discovery** -- all base URLs are statically configured via environment variables (`INVENTORY_SERVICE_URL`, `NOTIFICATION_SERVICE_URL`, `VITE_INVENTORY_API_URL`, `VITE_ORDER_API_URL`, `VITE_API_BASE_URL`).