# Skill: Run the Definition of Done Checklist

Run these commands before marking any WP `done` in `status.yaml`.
Every command must exit with code 0. Fix failures before proceeding.

---

## Backend

```bash
# 1. Linter
uv run ruff check .
uv run ruff format --check .

# 2. Type checker
uv run mypy src/

# 3. Unit + integration tests
uv run pytest tests/unit tests/integration -v --tb=short

# 4. Start service for contract validation
uv run uvicorn src.main:app --host 0.0.0.0 --port 8000 &
sleep 3

# 5. Contract validation (Schemathesis)
uv run schemathesis run ../../contracts/api/{service}.openapi.yaml \
  --base-url http://localhost:8000 \
  --checks all \
  --hypothesis-max-examples 25

# 6. Health + readiness endpoints
curl -sf http://localhost:8000/health | python -m json.tool
curl -sf http://localhost:8000/ready  | python -m json.tool

# 7. Metrics endpoint (spot check)
curl -sf http://localhost:8000/metrics | grep http_request_duration_seconds

# 8. Secrets scan (no tokens or credentials in source)
uv run ruff check . --select S

# Shut down background service
kill %1
```

Substitute `{service}` with the correct filename from `contracts/api/`.

---

## Frontend

```bash
# 1. Linter + formatter
npx biome check .

# 2. Type checker
npm run typecheck

# 3. Tests (MSW active, onUnhandledRequest: "error")
npm test -- --reporter=verbose

# 4. Production build (catches bundler and tree-shaking errors)
npm run build
```

---

## Final Gate

Before updating `status.yaml`:

- [ ] All commands above exit 0
- [ ] No `TODO`, `FIXME`, or `HACK` comments introduced in this WP
- [ ] No hardcoded secrets, API keys, or credentials
- [ ] `status.yaml` `phase_4.WP-XXX.status` → `done`
