# Backend Work Package Template

Structure for WP-XXX-BE.md documents. The implementation agent reads ONLY this file during Phase 4. Every section must be self-contained.

---

## Template

```markdown
# WP-XXX-BE: {Feature Title} — Backend Work Package

**Feature:** Story-XXXX-{slug}
**Target workspace:** `workspaces/{service-id}`
**Status:** awaiting_review
**Generated:** YYYY-MM-DD

---

## Objective

{One paragraph: what this WP delivers. List the key capabilities being implemented.}

All external service calls ({list services}) are made via configurable HTTP clients so they can be mocked in tests.

---

## Acceptance Criteria (verbatim from FS-XXX)

### AC-XXX — {title}

{Full behavioral description copied verbatim from the FS.}

**Testable:** {Testable line copied verbatim from the FS.}

<!-- Repeat for each AC assigned to this WP. -->

---

## Test Scenarios (verbatim from TS-XXX)

### TS-XXX-NNN — {scenario title}
- **Preconditions:** {condensed: what services/mocks are set up, what data exists}
- **Action:** {method, path, key payload or headers}
- **Expected:** {HTTP status, key response fields, DB/mock assertions}

<!-- Repeat for each scenario assigned to this WP. -->

---

## Relevant Contracts

<!-- Excerpt enough detail that the implementer does NOT need to read contracts/.
     Include endpoint signatures, request/response schemas, key constraints. -->

### {service}.openapi.yaml (excerpts)

```
{Endpoint signatures with method, path, security, headers, body schema, response codes}
{Key schema definitions: required fields, types, enums, constraints}
```

### {entity}.schema.md (key constraints)

{Column types, unique constraints, foreign keys, nullable rules, index requirements}

<!-- Include contracts for external services this WP calls (inventory, payment, notification, etc.) -->

---

## Implementation Notes

### Project structure
```
workspaces/{service-id}/
  src/
    config.py                   # Settings and env vars
    main.py                     # FastAPI app setup
    dependencies.py             # Dependency injection
    models/                     # ORM models
    schemas/                    # Pydantic request/response schemas
    repositories/               # Data access
    services/                   # Business logic / orchestration
    clients/                    # External HTTP client wrappers
    api/
      health.py                 # GET /health, /ready, /metrics
      v1/                       # Versioned API routes
  tests/
    conftest.py                 # Shared fixtures
    integration/                # Integration tests (all TS scenarios)
  alembic/
    versions/                   # DB migrations
  Dockerfile
  docker-compose.yml
  pyproject.toml
```

### Tech stack
{Framework, language version, key dependencies — pull from workspace-bootstrap.md}

### Key patterns

{Saga/orchestration flow with numbered steps if applicable}
{External HTTP client setup with configurable base URLs}
{Test mocking pattern (e.g., respx for Python HTTP mocks)}
{Any domain-specific patterns from contracts/architecture/}

### Key constraints
{Learnings from CLAUDE.learnings.md relevant to this workspace}

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests
- [ ] `ruff check src/` passes with zero errors
- [ ] `mypy src/` passes with zero errors
- [ ] `/health`, `/ready`, `/metrics` endpoints return expected responses
- [ ] No secrets, tokens, or credentials in code or test fixtures
- [ ] Contract validation passes

---

## Implementation Order

Implement in this order:
1. Scaffold workspace (pyproject.toml, Dockerfile, docker-compose.yml)
2. DB models and Alembic migration
3. {data store setup: Redis, cache, etc. if needed}
4. External HTTP clients (configurable base URLs)
5. Repositories (data access layer)
6. Service layer (business logic / saga)
7. API endpoints
8. Health/ready/metrics endpoints
9. Tests (all TS scenarios)
10. Run DoD checklist
```
