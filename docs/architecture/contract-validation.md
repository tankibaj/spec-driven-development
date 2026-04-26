# Contract Validation Standard

**Status:** Active
**Last updated:** 2026-04-04
**Applies to:** All backend workspaces and any frontend that consumes a service API

Contract validation is a mandatory step in Phase 4. It ensures that what the service actually
returns matches what the OpenAPI spec says it returns. No WP is `done` without passing contract
validation.

---

## 1. Backend — API Contract Tests (Schemathesis)

[Schemathesis](https://schemathesis.io/) generates and runs test cases directly from the OpenAPI
spec. It catches schema drift, missing error responses, and serialisation bugs automatically.

### 1.1 Where tests live

```
tests/contract/
├── test_openapi.py       # Schemathesis stateful + stateless tests
└── conftest.py           # Auth fixtures, base URL, shared state
```

### 1.2 Standard test file

```python
# tests/contract/test_openapi.py
import schemathesis
from schemathesis.specs.openapi.links import OpenAPILink

# Load the spec from the workspace's auto-generated docs (CI-generated, read-only)
schema = schemathesis.from_path(
    "docs/api/openapi.json",
    base_url="http://localhost:8000",
)

# Stateless — test every operation independently
@schema.parametrize()
def test_api_stateless(case):
    """Every endpoint must respond with a schema-valid response."""
    response = case.call()
    case.validate_response(response)


# Stateful — test operation sequences using OpenAPI links
@schema.parametrize()
@schemathesis.stateful.links
def test_api_stateful(case):
    """Follow operation links to test realistic workflows."""
    response = case.call()
    case.validate_response(response)
```

### 1.3 Running locally

```bash
# Against local dev server
uv run schemathesis run docs/api/openapi.json \
  --base-url http://localhost:8000 \
  --checks all \
  --hypothesis-max-examples 50

# With auth header
uv run schemathesis run docs/api/openapi.json \
  --base-url http://localhost:8000 \
  --checks all \
  --header "X-Tenant-ID: 00000000-0000-0000-0000-000000000001"
```

### 1.4 What "checks all" validates

| Check | Description |
|---|---|
| `not_a_server_error` | No 5xx responses |
| `status_code_conformance` | Response status matches what the spec declares |
| `content_type_conformance` | Content-Type header matches spec |
| `response_schema_conformance` | Response body matches the declared JSON Schema |
| `response_headers_conformance` | Required headers are present |

---

## 2. Inter-Service Contracts (Pact)

[Pact](https://pact.io/) implements consumer-driven contract testing. The **consumer** (e.g.
`order-service`) defines what it expects from the **provider** (e.g. `inventory-service`). The
provider verifies those expectations against its real implementation.

### 2.1 Consumer side (e.g. `order-service`)

```python
# tests/contract/test_inventory_consumer.py
import pytest
from pact import Consumer, Provider

pact = Consumer("order-service").has_pact_with(
    Provider("inventory-service"),
    pact_dir="tests/contract/pacts",
)

def test_reserve_stock_success(pact):
    expected_response = {
        "reservation_id": "some-uuid",
        "expires_at": "2026-04-04T00:00:00Z",
    }
    (
        pact.given("SKU-A has sufficient stock")
        .upon_receiving("a stock reservation request")
        .with_request("POST", "/stock/reserve")
        .will_respond_with(200, body=expected_response)
    )
    with pact:
        # Call your service's internal client here
        from src.clients.inventory import reserve_stock
        result = reserve_stock(order_id="uuid", lines=[{"sku_id": "uuid", "quantity": 1}])
        assert result["reservation_id"] is not None
```

### 2.2 Provider side (e.g. `inventory-service`)

```python
# tests/contract/test_inventory_provider.py
import pytest
from pact import Verifier

def test_pact_provider():
    verifier = Verifier(
        provider="inventory-service",
        provider_base_url="http://localhost:8001",
    )
    output, _ = verifier.verify_with_broker(
        broker_url="http://pact-broker:9292",  # or local pact file path
        provider_states_setup_url="http://localhost:8001/_pact/provider-states",
    )
    assert output == 0
```

### 2.3 Where pact files live

Consumer pact files (`*.json`) are generated in `tests/contract/pacts/` and MUST be committed.
They serve as the source of truth for the provider verification step.

For local development without a Pact Broker, use:
```bash
uv run pytest tests/contract/test_{service}_provider.py \
  --pact-files tests/contract/pacts/{consumer}-{provider}.json
```

---

## 3. Frontend — API Mocking (MSW)

The frontend uses [Mock Service Worker](https://mswjs.io/) to mock API responses during both
development and component tests. Handlers MUST be derived from the OpenAPI spec — do not
hand-invent response shapes.

### 3.1 Generating TypeScript types from the OpenAPI spec

```bash
npm run generate:types
# Runs: openapi-typescript docs/api/openapi.json -o src/api/types.ts
```

Run this command whenever the OpenAPI spec changes. The generated file is committed to the repo.
**Do not hand-edit `src/api/types.ts`.**

### 3.2 MSW handler pattern

```typescript
// src/mocks/handlers/orders.ts
import { http, HttpResponse } from "msw";
import type { paths } from "../api/types";

type PlaceOrderResponse =
  paths["/checkout/guest/orders"]["post"]["responses"]["201"]["content"]["application/json"];

export const orderHandlers = [
  http.post("/checkout/guest/orders", () => {
    const response: PlaceOrderResponse = {
      id: "mock-order-uuid",
      reference: "ORD-20260404-TEST",
      status: "confirmed",
      lines: [],
      subtotal: 4500,
      shipping_cost: 499,
      tax: 900,
      total: 5899,
      shipping_address: {
        line1: "12 High Street",
        city: "London",
        postal_code: "EC1A 1BB",
        country_code: "GB",
      },
      created_at: new Date().toISOString(),
    };
    return HttpResponse.json(response, { status: 201 });
  }),
];
```

### 3.3 Enabling MSW in tests

```typescript
// tests/setup.ts
import { server } from "../src/mocks/server";
import "@testing-library/jest-dom";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Using `onUnhandledRequest: "error"` ensures that any unmocked API call in tests fails loudly,
preventing silent test drift.

---

## 4. Agent Checklist — Contract Validation in Phase 4

Before marking any WP `done` in `status.yaml`, the agent MUST confirm:

- [ ] `schemathesis run` passes with zero failures against the service's OpenAPI spec
- [ ] Pact consumer tests pass (if this WP calls another service)
- [ ] Pact provider tests pass (if this WP implements endpoints consumed by another service)
- [ ] Frontend `generate:types` has been run after any contract change and types are committed
- [ ] MSW handlers cover all API calls made by new components (no unhandled request errors in tests)
