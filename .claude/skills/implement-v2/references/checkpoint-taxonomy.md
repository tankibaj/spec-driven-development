# Checkpoint Taxonomy

Standard checkpoint categories for `status.yaml` updates during Phase 4 implementation. After completing each category, update `phase_4.WP-XXX.last_checkpoint` with the category name and a brief description of what was completed.

---

## Backend Checkpoints

| Order | Category | What's completed | Example checkpoint |
|---|---|---|---|
| 1 | `scaffold` | Project structure, dependencies, config, Dockerfile, docker-compose | `"scaffold — pyproject.toml, Dockerfile, docker-compose, app factory"` |
| 2 | `models` | ORM models, Alembic migrations, seed data | `"models — Order, OrderLine models + migration + seed data"` |
| 3 | `routes` | Pydantic schemas, repositories, services, API endpoints, health endpoints | `"routes — guest checkout endpoints + health/ready/metrics"` |
| 4 | `integration` | External HTTP clients, request tracing, structured logging | `"integration — inventory-service client + X-Request-ID propagation"` |
| 5 | `tests` | All TS scenarios passing as automated tests | `"tests — 18/18 TS scenarios passing"` |
| 6 | `dod` | Full DoD checklist passed (linter, types, contracts, observability, no secrets) | `"dod — ruff, mypy, schemathesis, pact all passing"` |

---

## Frontend Checkpoints

| Order | Category | What's completed | Example checkpoint |
|---|---|---|---|
| 1 | `scaffold` | Project structure, dependencies, config, Dockerfile, OpenAPI types generated | `"scaffold — vite + react + biome + MSW + openapi types generated"` |
| 2 | `state` | Zustand stores, API client, router, custom hooks | `"state — cart store with persist, API client, route definitions"` |
| 3 | `components` | UI components, pages, layouts, forms, loading/error/empty states | `"components — catalog grid, product detail, cart drawer, checkout form"` |
| 4 | `integration` | MSW handlers, browser + node mock setup, API client integration | `"integration — MSW handlers for all inventory + order endpoints"` |
| 5 | `tests` | All TS scenarios passing as automated tests | `"tests — 12/12 TS scenarios passing"` |
| 6 | `dod` | Full DoD checklist passed (linter, types, MSW coverage, no secrets) | `"dod — biome, tsc, all MSW handlers verified"` |

---

## Rules

- **Update after each category.** Don't batch multiple categories into one checkpoint.
- **Include a count where applicable.** "tests — 18/18 passing" is better than "tests done."
- **Commit after each checkpoint.** Each category should have at least one git commit.
- **On resume:** the agent reads `last_checkpoint` and skips completed categories. A missing checkpoint means the category wasn't completed.
- **Partial completion:** if a category is partially done when a session is interrupted, the checkpoint should describe what was completed (e.g., `"routes — 3/5 endpoints implemented"`). On resume, the agent picks up from where it left off within that category.

---

## status.yaml Format

```yaml
phase_4:
  WP-001-BE:
    status: in_progress
    last_checkpoint: "routes — guest checkout endpoints + health/ready/metrics"
  WP-001-FE:
    status: not_started
```

When a WP is complete:

```yaml
phase_4:
  WP-001-BE:
    status: done
```

`last_checkpoint` is cleared when the WP reaches `done` — it's only needed during active implementation and recovery.
