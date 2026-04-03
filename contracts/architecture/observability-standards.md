# Observability Standards

**Status:** Active
**Last updated:** 2026-04-04
**Applies to:** All backend workspaces

Every backend service MUST implement these standards before a Work Package is considered `done`.
They are the minimum required for production operation, on-call debugging, and SLO monitoring.

---

## 1. Health Endpoints

Every service MUST expose two health endpoints. These are checked by Docker, Kubernetes, and
the CI pipeline.

### `GET /health` — Liveness

Returns `200 OK` if the process is running and able to accept requests. Does **not** check
downstream dependencies. Used by the container orchestrator to decide whether to restart the
pod.

```json
HTTP 200
{
  "status": "ok",
  "service": "order-service",
  "version": "1.2.3"
}
```

### `GET /ready` — Readiness

Returns `200 OK` only if the service is ready to serve traffic (database reachable, Redis
reachable, migrations applied). Returns `503` if any dependency is unhealthy. Used by the load
balancer to decide whether to route traffic to this instance.

```json
HTTP 200
{
  "status": "ready",
  "checks": {
    "database": "ok",
    "redis":    "ok"
  }
}
```

```json
HTTP 503
{
  "status": "not_ready",
  "checks": {
    "database": "ok",
    "redis":    "error: connection refused"
  }
}
```

### FastAPI implementation pattern

```python
# src/api/health.py
from fastapi import APIRouter, status
from fastapi.responses import JSONResponse
from src.config import settings
from src.dependencies import get_db, get_redis

router = APIRouter(tags=["Health"])

@router.get("/health")
async def health():
    return {"status": "ok", "service": settings.service_name, "version": settings.version}

@router.get("/ready")
async def ready(db=Depends(get_db), redis=Depends(get_redis)):
    checks = {}
    http_status = status.HTTP_200_OK

    try:
        await db.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"
        http_status = status.HTTP_503_SERVICE_UNAVAILABLE

    try:
        await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"
        http_status = status.HTTP_503_SERVICE_UNAVAILABLE

    return JSONResponse(
        content={"status": "ready" if http_status == 200 else "not_ready", "checks": checks},
        status_code=http_status,
    )
```

---

## 2. Prometheus Metrics (`GET /metrics`)

Every service MUST expose a `/metrics` endpoint in the
[Prometheus exposition format](https://prometheus.io/docs/instrumenting/exposition_formats/).

### 2.1 Setup (FastAPI)

```python
# src/main.py
from prometheus_fastapi_instrumentator import Instrumentator

def create_app() -> FastAPI:
    app = FastAPI(...)
    Instrumentator().instrument(app).expose(app, endpoint="/metrics")
    return app
```

### 2.2 Required metrics (auto-instrumented)

`prometheus-fastapi-instrumentator` provides these automatically:

| Metric | Type | Description |
|---|---|---|
| `http_request_duration_seconds` | Histogram | Request latency by method, path, status code |
| `http_requests_total` | Counter | Total requests by method, path, status code |
| `http_requests_in_progress` | Gauge | Concurrent requests currently being processed |

### 2.3 Required labels

Every metric MUST include these labels:

| Label | Value |
|---|---|
| `service` | The workspace ID (e.g. `order-service`) |
| `method` | HTTP method (`GET`, `POST`, etc.) |
| `path` | Route path (e.g. `/checkout/guest/orders`) — **not** the full URL with IDs |
| `status_code` | HTTP status code |

### 2.4 Custom business metrics

Add custom metrics for domain events that Prometheus cannot derive from HTTP traffic alone.
Use the `prometheus_client` library directly.

```python
# src/metrics.py
from prometheus_client import Counter, Histogram

orders_placed_total = Counter(
    "orders_placed_total",
    "Total number of orders placed successfully",
    ["tenant_id"],
)

stock_conflicts_total = Counter(
    "stock_conflicts_total",
    "Total number of order placements rejected due to stock conflict",
    ["tenant_id"],
)

saga_duration_seconds = Histogram(
    "saga_duration_seconds",
    "Duration of the order placement saga in seconds",
    ["outcome"],  # "success" | "stock_conflict" | "payment_failed"
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)
```

Emit these in the relevant service methods, not in the API layer.

---

## 3. Structured Logging (Loki-compatible)

Every service MUST emit logs as **single-line JSON objects** to stdout. Loki scrapes container
stdout and indexes by labels. Do not write log files.

### 3.1 Required fields per log line

| Field | Type | Description |
|---|---|---|
| `timestamp` | ISO 8601 string | UTC time of the log event |
| `level` | string | `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL` |
| `service` | string | Workspace ID (e.g. `order-service`) |
| `trace_id` | string | `X-Request-ID` value (see Section 4) |
| `message` | string | Human-readable description |
| `...` | any | Additional structured fields relevant to the event |

### 3.2 Example log lines

```json
{"timestamp":"2026-04-04T10:22:01.123Z","level":"INFO","service":"order-service","trace_id":"abc-123","message":"Guest order placement started","tenant_id":"uuid","session_id":"uuid"}
{"timestamp":"2026-04-04T10:22:01.250Z","level":"INFO","service":"order-service","trace_id":"abc-123","message":"Stock reserved","reservation_id":"uuid","expires_at":"2026-04-04T10:37:01Z"}
{"timestamp":"2026-04-04T10:22:01.890Z","level":"INFO","service":"order-service","trace_id":"abc-123","message":"Order placed","order_id":"uuid","reference":"ORD-20260404-A3K9","total_minor":5899}
```

### 3.3 Setup (FastAPI + python-json-logger)

```python
# src/logging_config.py
import logging
import sys
from pythonjsonlogger import jsonlogger

def configure_logging(service_name: str, log_level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    formatter = jsonlogger.JsonFormatter(
        fmt="%(timestamp)s %(level)s %(service)s %(trace_id)s %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S.%fZ",
        rename_fields={"levelname": "level", "asctime": "timestamp"},
    )
    handler.setFormatter(formatter)

    root = logging.getLogger()
    root.addHandler(handler)
    root.setLevel(getattr(logging, log_level.upper()))

# src/middleware/logging.py
import uuid
import logging
from starlette.middleware.base import BaseHTTPMiddleware

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        trace_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        # Bind trace_id to all log records in this request context
        with logging_context(trace_id=trace_id, service=settings.service_name):
            response = await call_next(request)
            response.headers["X-Request-ID"] = trace_id
            logger.info(
                "Request completed",
                extra={
                    "method": request.method,
                    "path": request.url.path,
                    "status_code": response.status_code,
                }
            )
            return response
```

### 3.4 What MUST NOT appear in logs

- Card numbers, CVVs, or any payment instrument data
- Passwords or authentication tokens
- Full request/response bodies for payment endpoints
- Personally identifiable information (PII) unless explicitly required and masked

Use Ruff rule `S320` and structured log field allowlists to enforce this.

---

## 4. Request Tracing (`X-Request-ID`)

Every service MUST propagate a `X-Request-ID` header across all inter-service HTTP calls.

### Rules

1. **Inbound:** If the incoming request contains `X-Request-ID`, use it. If not, generate a new UUID v4.
2. **Outbound:** Attach the same `X-Request-ID` to every downstream HTTP request made during that operation.
3. **Logs:** Include `trace_id` (= `X-Request-ID`) in every log line emitted during that request.
4. **Response:** Echo the `X-Request-ID` back in the response headers.

This creates a traceable chain across `storefront-app → order-service → inventory-service →
notification-service` without requiring a full distributed tracing system.

### httpx client pattern

```python
# src/clients/base.py
import httpx
from contextvars import ContextVar

current_trace_id: ContextVar[str] = ContextVar("trace_id", default="")

class TracingTransport(httpx.AsyncBaseTransport):
    async def handle_async_request(self, request):
        trace_id = current_trace_id.get()
        if trace_id:
            request.headers["X-Request-ID"] = trace_id
        return await super().handle_async_request(request)
```

---

## 5. Agent Checklist — Observability in Phase 4

Before marking any WP `done` in `status.yaml`, confirm:

- [ ] `GET /health` returns `200 {"status": "ok"}` with no downstream checks
- [ ] `GET /ready` returns `200` when all dependencies are healthy, `503` when any are down
- [ ] `GET /metrics` returns Prometheus exposition text with `http_request_duration_seconds` and `http_requests_total`
- [ ] All log lines are single-line JSON with `timestamp`, `level`, `service`, `trace_id`, `message`
- [ ] `X-Request-ID` is read from inbound requests, generated if absent, propagated to all outbound calls, and echoed in responses
- [ ] No PII, card data, or secrets appear in any log line
- [ ] Custom business metrics are defined for domain events relevant to this WP
