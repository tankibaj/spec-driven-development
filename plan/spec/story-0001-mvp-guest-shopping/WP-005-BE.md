# WP-005-BE: Email Delivery API — Backend Work Package

**Feature:** story-0001-mvp-guest-shopping
**Target workspace:** `workspaces/notification-service`
**Status:** awaiting_review
**Generated:** 2026-04-11

---

## Objective

Implement the notification-service API: accept notification requests (`POST /notifications`), render email templates (Jinja2), deliver via SMTP (MailPit for MVP), track delivery status, support status queries (`GET /notifications/{id}`), handle delivery failures with retry logic, and expose observability endpoints (`/health`, `/ready`, `/metrics`).

This is an internal service-to-service API — not publicly exposed, no auth required. order-service calls `POST /notifications` after successful order placement to dispatch the confirmation email.

All external service calls: SMTP server (MailPit at configurable host/port) — made via configurable SMTP client so it can be mocked in tests.

---

## Acceptance Criteria (verbatim from FS-001)

### AC-018 — Confirmation email dispatched

After successful order placement, the order-service sends an order confirmation email via the notification-service.

**Testable:** After `POST /checkout/guest/orders` returns `201`, order-service calls `POST /notifications` on notification-service with `channel: email`, `template_id: order_confirmation`, and `recipient.address` set to the guest's email address, within 30 seconds of order confirmation.

---

### AC-019 — Email contains order details

The confirmation email includes the order reference, line items, and order total so the guest can verify their order.

**Testable:** The `POST /notifications` request `payload` object includes `order_reference` (string matching the order's `reference`), line item details (product name, quantity, unit price per line), and `total` (formatted currency string).

---

### AC-020 — Email delivery failure

If the confirmation email fails all delivery retries, the notification is marked as failed. The order itself remains confirmed — notification failure does not roll back the order.

**Testable:** When notification-service marks a notification as `status: failed` after exhausting retries, the corresponding order's `status` remains `confirmed`. `GET /notifications/{notificationId}` returns `status: failed`.

---

## Test Scenarios (from TS-001)

### TS-001-034 — Order-service calls notification-service after successful order
- **Preconditions:** notification-service running; a valid `POST /notifications` request arrives with `channel: "email"`, `template_id: "order_confirmation"`, `recipient.address: "grace@example.com"`, and payload with order details
- **Action:** `POST /notifications` with header `X-Tenant-ID: {tenantId}` and body containing `channel`, `template_id`, `recipient`, `payload`
- **Expected:** HTTP 202; response matches `NotificationReceipt` schema with `id` (UUID), `status: "queued"`, `created_at`; notification record persisted in DB

> **Note (notification-service perspective):** This scenario validates that notification-service correctly accepts and queues a notification request. The "order-service calls" part is order-service's responsibility — from notification-service's view, a valid POST arrives and is accepted.

### TS-001-035 — Notification not sent when order placement fails
- **Preconditions:** notification-service running and configured
- **Action:** No `POST /notifications` request arrives (order-service does not call notification-service because the order failed)
- **Expected:** No notification record created in notification-service

> **Note (notification-service perspective):** This scenario validates the contract boundary — notification-service only acts when called. If no request arrives, no notification is created. This scenario is tested at the order-service level (TS verifies order-service does NOT call notification-service on failure). From notification-service's perspective, there is nothing to test — it is a passive service that only responds to inbound requests.

### TS-001-036 — Notification payload includes order reference, items, and total
- **Preconditions:** notification-service running; `order_confirmation` email template exists and expects `order_reference`, `lines` (array of items), and `total` in payload
- **Action:** `POST /notifications` with payload `{ "order_reference": "ORD-20260411-A3K9", "lines": [{"product_name": "Classic T-Shirt", "variant_label": "Small", "quantity": 2, "unit_price": "$29.99"}], "total": "$74.97" }`
- **Expected:** HTTP 202; notification queued; rendered email HTML contains "ORD-20260411-A3K9", "Classic T-Shirt", quantity "2", and "$74.97"

### TS-001-037 — Failed notification does not affect order status
- **Preconditions:** notification-service running; a notification was accepted (`POST /notifications` returned 202); SMTP delivery fails after all retries (mock SMTP rejects delivery)
- **Action:** notification-service processes the notification and exhausts 3 delivery retries
- **Expected:** `GET /notifications/{notificationId}` returns `status: "failed"`; `delivered_at` is null; the notification record exists with `status = failed` in DB

> **Note:** The "order remains confirmed" part is order-service's responsibility — notification-service has no knowledge of order status. This scenario validates that notification-service correctly marks failed notifications and they are queryable.

### TS-001-038 — Failed notification is queryable via GET
- **Preconditions:** notification-service running; a notification exists with `status: "failed"`
- **Action:** `GET /notifications/{notificationId}` with header `X-Tenant-ID: {tenantId}`
- **Expected:** HTTP 200; response matches `NotificationReceipt`: `id`, `status: "failed"`, `channel: "email"`, `template_id: "order_confirmation"`, `created_at`; `delivered_at` is null

### TS-001-062 — notification-service GET /health returns 200
- **Preconditions:** notification-service running
- **Action:** `GET /health`
- **Expected:** HTTP 200; body `{"status": "ok"}`

### TS-001-063 — notification-service GET /ready returns 200
- **Preconditions:** notification-service running; SMTP connection (MailPit) available
- **Action:** `GET /ready`
- **Expected:** HTTP 200; body contains `"status": "ready"`

### TS-001-064 — notification-service GET /metrics returns Prometheus format
- **Preconditions:** notification-service running
- **Action:** `GET /metrics`
- **Expected:** HTTP 200; `Content-Type` contains `text/plain`; body contains Prometheus-formatted metrics

---

## Relevant Contracts

### notification-service.openapi.yaml (full excerpt)

**Base URL:** `https://example.com/notification-service/api/v1/` (internal network only)

#### Parameters (shared)

```yaml
X-Tenant-ID:
  in: header
  required: true
  schema:
    type: string
    format: uuid

Idempotency-Key:
  in: header
  required: false
  schema:
    type: string
    maxLength: 64
```

#### `POST /notifications` — Send a transactional notification

```
Description: Dispatches a notification to the specified recipient using
the chosen channel and template. Idempotent when the same Idempotency-Key
is provided.

Parameters:
  - X-Tenant-ID (header, required, UUID)
  - Idempotency-Key (header, optional, string max 64)

Request body → SendNotificationRequest (required)

Response 202 → NotificationReceipt (notification accepted for delivery)
Response 400 → ErrorResponse (malformed request)
Response 422 → ErrorResponse (semantically invalid)
```

#### `GET /notifications/{notificationId}` — Get notification delivery status

```
Parameters:
  - X-Tenant-ID (header, required, UUID)
  - notificationId (path, required, UUID)

Response 200 → NotificationReceipt
Response 404 → ErrorResponse
```

### Schemas

#### SendNotificationRequest

```yaml
type: object
required: [channel, template_id, recipient, payload]
properties:
  channel:
    type: string
    enum: [email, sms]
  template_id:
    type: string
    description: Identifier of the notification template (e.g. "order_confirmation")
  recipient:
    type: object
    required: [address]
    properties:
      address:
        type: string
        description: Email address for email channel; E.164 phone for sms
      name:
        type: string
        description: Optional display name (used in email salutation)
  payload:
    type: object
    additionalProperties: true
    description: >
      Template variable values. Keys/types depend on template.
      Example for order_confirmation:
        { "order_reference": "ORD-20240415-A3K9", "total": "€49.99" }
```

#### NotificationReceipt

```yaml
type: object
required: [id, status, created_at]
properties:
  id:           { type: string, format: uuid }
  status:       { type: string, enum: [queued, sent, delivered, failed] }
  channel:      { type: string }
  template_id:  { type: string }
  created_at:   { type: string, format: date-time }
  delivered_at: { type: string, format: date-time, nullable: true }
```

#### ErrorResponse

```yaml
type: object
required: [code, message]
properties:
  code:    { type: string }
  message: { type: string }
```

### notifications table (data schema — designed for this WP)

No pre-existing data schema file exists for notification-service. The following table design satisfies the contract and AC requirements:

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `UUID` | NO | Primary key |
| `tenant_id` | `UUID` | NO | Tenant scope |
| `channel` | `VARCHAR(10)` | NO | `email` or `sms` |
| `template_id` | `VARCHAR(100)` | NO | Template identifier (e.g. `order_confirmation`) |
| `recipient_address` | `VARCHAR(254)` | NO | Email address or phone number |
| `recipient_name` | `VARCHAR(200)` | YES | Optional display name |
| `payload` | `JSONB` | NO | Template variable values |
| `status` | `VARCHAR(20)` | NO | `queued`, `sent`, `delivered`, `failed` |
| `retry_count` | `INT` | NO | Number of delivery attempts made. Default: 0 |
| `created_at` | `TIMESTAMPTZ` | NO | Record creation time (UTC) |
| `delivered_at` | `TIMESTAMPTZ` | YES | Time of successful delivery. NULL if not yet delivered or failed |
| `idempotency_key` | `VARCHAR(64)` | YES | Client-provided idempotency key |

**Constraints:**
- `UNIQUE (tenant_id, idempotency_key)` — partial unique index where `idempotency_key IS NOT NULL`
- `status` values: `queued`, `sent`, `delivered`, `failed`

**Indexes:**

```sql
CREATE INDEX idx_notifications_tenant ON notifications (tenant_id);
CREATE INDEX idx_notifications_status ON notifications (status) WHERE status = 'queued';
CREATE UNIQUE INDEX idx_notifications_idempotency
  ON notifications (tenant_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;
```

### Caller context: how order-service calls notification-service

For reference, order-service calls `POST /notifications` after a successful order placement (saga step 4: notify). The request looks like:

```json
{
  "channel": "email",
  "template_id": "order_confirmation",
  "recipient": {
    "address": "grace@example.com",
    "name": "Grace"
  },
  "payload": {
    "order_reference": "ORD-20260411-A3K9",
    "lines": [
      {
        "product_name": "Classic T-Shirt",
        "variant_label": "Small",
        "quantity": 2,
        "unit_price": "$29.99"
      }
    ],
    "total": "$74.97"
  }
}
```

This context helps when designing the `order_confirmation` email template.

---

## Implementation Notes

### Project structure

```
workspaces/notification-service/
  src/
    __init__.py
    config.py                   # Settings: SMTP host/port, DB URL, etc.
    main.py                     # FastAPI app factory, lifespan, middleware
    dependencies.py             # DB session, SMTP client injection
    models/
      __init__.py
      notification.py           # Notification ORM model
    schemas/
      __init__.py
      notification.py           # SendNotificationRequest, NotificationReceipt, etc.
    repositories/
      __init__.py
      notification_repository.py # CRUD for notifications table
    services/
      __init__.py
      notification_service.py   # Queue, render, send, retry, status update
      template_engine.py        # Jinja2 template loading and rendering
      smtp_client.py            # Async SMTP delivery (configurable)
    templates/
      order_confirmation.html   # Jinja2 HTML email template
    api/
      __init__.py
      health.py                 # GET /health, /ready, /metrics
      router.py                 # Mounts all route groups
      v1/
        __init__.py
        notifications.py        # POST /notifications, GET /notifications/{id}
  tests/
    __init__.py
    conftest.py                 # Shared fixtures: app, async DB, test client, mock SMTP
    integration/
      test_notifications.py     # TS-001-034, TS-001-036, TS-001-037, TS-001-038
      test_observability.py     # TS-001-062, TS-001-063, TS-001-064
  alembic/
    env.py
    versions/
  Dockerfile
  docker-compose.yml            # Service + PostgreSQL + MailPit
  pyproject.toml
  .env.example
```

### Tech stack

- **Language:** Python 3.13
- **Framework:** FastAPI `^0.135`
- **ASGI:** Uvicorn `^0.43`
- **ORM:** SQLAlchemy async `^2.0`
- **DB driver:** asyncpg `^0.31`
- **Migrations:** Alembic `^1.18`
- **Validation:** Pydantic v2 `^2.12`
- **Settings:** pydantic-settings `^2.13`
- **SMTP client:** `aiosmtplib` (async SMTP) — add to `pyproject.toml`
- **Template engine:** Jinja2 (included with FastAPI/Starlette)
- **HTTP client:** httpx `^0.28` (not needed outbound in this WP, but included per bootstrap standard)
- **Package manager:** uv (latest)
- **Linter:** Ruff `^0.15`
- **Type checker:** mypy `^1.20` (strict mode)
- **Test runner:** pytest `^9.0` + pytest-asyncio `^1.3`
- **Test HTTP mocking:** respx (for any HTTP mocks, though this service does SMTP not HTTP outbound)
- **Metrics:** prometheus-fastapi-instrumentator `^7.1`
- **Logging:** python-json-logger `^4.1`
- **Docker base:** `python:3.13-slim`

### Key patterns

#### Notification processing flow

1. **Accept** (`POST /notifications`): Validate request, check idempotency key (return existing receipt if duplicate), persist notification with `status = queued`, return 202 with `NotificationReceipt`.

2. **Process** (synchronous within the request for MVP, or via background task): Render the email template with Jinja2 using the `payload` data, then attempt SMTP delivery.

3. **Deliver via SMTP**: Connect to the configured SMTP server (MailPit at `localhost:1025` or configurable via `SMTP_HOST`/`SMTP_PORT` env vars). Send the rendered HTML email. On success, update `status = sent`, set `delivered_at`. On SMTP error, increment `retry_count` and retry.

4. **Retry logic**: 3 attempts total with exponential backoff (1s, 2s, 4s). After all retries exhausted, mark `status = failed`. Do NOT raise an error to the caller — the 202 was already returned.

> **MVP simplification:** For the initial implementation, process the notification synchronously in the POST handler (render + send + retry). The 202 response is returned after processing completes. A future iteration can move to async background processing. Alternatively, use `asyncio.create_task()` to fire-and-forget the delivery after returning 202 — this is the preferred approach as it matches the 202 "accepted" semantics.

#### Recommended approach: fire-and-forget delivery

```python
@router.post("/notifications", status_code=202)
async def send_notification(request: SendNotificationRequest, ...):
    notification = await service.create_notification(request)  # persist as queued
    asyncio.create_task(service.process_notification(notification.id))  # fire-and-forget
    return NotificationReceipt.from_model(notification)  # return 202 immediately
```

The `process_notification` task handles rendering, SMTP delivery, and retry logic. The GET endpoint reflects the current status.

#### Jinja2 email templates

Store templates in `src/templates/`. The `order_confirmation.html` template receives:

```
Variables:
  - order_reference: str (e.g. "ORD-20260411-A3K9")
  - lines: list of { product_name, variant_label, quantity, unit_price }
  - total: str (e.g. "$74.97")
  - recipient_name: str | None
```

Example template structure:

```html
<!DOCTYPE html>
<html>
<head><title>Order Confirmation</title></head>
<body>
  <h1>Order Confirmed!</h1>
  {% if recipient_name %}<p>Hi {{ recipient_name }},</p>{% endif %}
  <p>Your order <strong>{{ order_reference }}</strong> has been confirmed.</p>
  <table>
    <tr><th>Product</th><th>Variant</th><th>Qty</th><th>Price</th></tr>
    {% for line in lines %}
    <tr>
      <td>{{ line.product_name }}</td>
      <td>{{ line.variant_label }}</td>
      <td>{{ line.quantity }}</td>
      <td>{{ line.unit_price }}</td>
    </tr>
    {% endfor %}
  </table>
  <p><strong>Total: {{ total }}</strong></p>
</body>
</html>
```

#### SMTP client (configurable)

```python
# src/services/smtp_client.py
import aiosmtplib

class SMTPClient:
    def __init__(self, host: str, port: int, sender: str):
        self.host = host
        self.port = port
        self.sender = sender

    async def send_email(self, to: str, subject: str, html_body: str) -> None:
        await aiosmtplib.send(
            message=build_mime_message(self.sender, to, subject, html_body),
            hostname=self.host,
            port=self.port,
            use_tls=False,  # MailPit does not use TLS
        )
```

**Environment variables:**
- `SMTP_HOST` — default `mailpit` (Docker service name)
- `SMTP_PORT` — default `1025`
- `SMTP_SENDER` — default `noreply@example.com`

#### Idempotency

If `Idempotency-Key` header is present, check for an existing notification with the same `(tenant_id, idempotency_key)`. If found, return the existing `NotificationReceipt` without creating a duplicate. If not found, create normally and store the key.

#### Readiness check

`GET /ready` should check:
- Database connection (execute `SELECT 1`)
- SMTP server reachable (attempt connection to SMTP host:port, or skip if acceptable for MVP)

#### Multi-tenant scoping

All queries MUST filter by `tenant_id` from the `X-Tenant-ID` header. Same dependency pattern as inventory-service:

```python
async def get_tenant_id(x_tenant_id: str = Header(...)) -> uuid.UUID:
    return uuid.UUID(x_tenant_id)
```

#### Observability

Follow `contracts/architecture/observability-standards.md`:
- `GET /health` → `200 {"status": "ok", "service": "notification-service", "version": "0.1.0"}`
- `GET /ready` → checks DB + SMTP, returns `200` if healthy, `503` if not
- `GET /metrics` → Prometheus exposition format via `prometheus-fastapi-instrumentator`
- Structured JSON logging to stdout
- Propagate `X-Request-ID` header
- No PII in logs (mask recipient email in logs — log only domain or hash)

#### Custom business metrics

```python
from prometheus_client import Counter, Histogram

notifications_sent_total = Counter(
    "notifications_sent_total",
    "Total notifications sent successfully",
    ["tenant_id", "channel", "template_id"],
)

notifications_failed_total = Counter(
    "notifications_failed_total",
    "Total notifications that failed all retries",
    ["tenant_id", "channel", "template_id"],
)

notification_delivery_duration_seconds = Histogram(
    "notification_delivery_duration_seconds",
    "Time to deliver a notification (render + SMTP send)",
    ["channel"],
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)
```

#### SMS channel

The contract allows `channel: sms`. For MVP, SMS delivery is **stubbed** — accept the request, persist it as `queued`, but do not attempt delivery. It stays `queued`. This is explicitly out of scope per FS-001.

### Key constraints

- notification-service is internal-only: no public exposure, no auth required
- Retry logic: exactly 3 attempts, exponential backoff (1s, 2s, 4s)
- After all retries fail → `status: failed`, never retry again
- `delivered_at` is NULL for `queued` and `failed` notifications
- Idempotency: same `(tenant_id, idempotency_key)` → same response
- MailPit captures all email in dev — no real email delivery
- Templates are file-based (bundled in the Docker image), not database-managed

### docker-compose.yml (includes MailPit)

```yaml
services:
  app:
    build: .
    ports:
      - "8002:8000"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
      mailpit:
        condition: service_started

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports:
      - "5434:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI for viewing captured emails
```

### Test strategy

- **Integration tests** use the real DB (PostgreSQL in Docker) and a **mock SMTP server**.
- For SMTP mocking: either use `aiosmtplib` mock/patch, or run MailPit in tests and verify captured emails via MailPit's API (`GET http://mailpit:8025/api/v1/messages`).
- **Recommended for unit/integration tests:** Mock the `SMTPClient` dependency in FastAPI to control success/failure without needing a real SMTP server. Use `unittest.mock.AsyncMock` or a test double.
- For TS-001-037 (retry exhaustion): mock SMTP to always raise an error, verify notification ends up `status: failed` after 3 attempts.
- For TS-001-036 (template rendering): verify the rendered HTML contains the expected payload values.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests (TS-001-034, TS-001-036, TS-001-037, TS-001-038, TS-001-062, TS-001-063, TS-001-064)
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes (Schemathesis against `notification-service.openapi.yaml`)

---

## Implementation Order

Implement in this order:

1. **Scaffold workspace** — `pyproject.toml` (add `aiosmtplib` and `Jinja2` to dependencies), `Dockerfile`, `docker-compose.yml` (with PostgreSQL + MailPit), `.env.example`, project directory structure per workspace-bootstrap standard
2. **DB model + Alembic migration** — `notifications` table with all columns, constraints, and indexes
3. **Template engine** — Jinja2 loader + `order_confirmation.html` template in `src/templates/`
4. **SMTP client** — async SMTP delivery wrapper with configurable host/port/sender
5. **Repository** — `NotificationRepository` (create, get by ID, update status, idempotency lookup)
6. **Service layer** — `NotificationService` (create/queue notification, render template, send via SMTP, retry with exponential backoff, mark failed after 3 attempts)
7. **API endpoints** — `POST /notifications` (accept + fire-and-forget delivery), `GET /notifications/{id}` (status query)
8. **Health/ready/metrics endpoints** — `GET /health`, `GET /ready` (DB + SMTP check), `GET /metrics`
9. **Tests** — all TS scenarios (TS-001-034, TS-001-036, TS-001-037, TS-001-038, TS-001-062 through TS-001-064), using mock SMTP for delivery tests
10. **Run DoD checklist** — ruff, mypy, Schemathesis, secrets check

### Dependency info

No dependencies on other services. This WP can start immediately (Wave 1). order-service depends on this service for post-order notification delivery.
