# Phase 2: Test Spec Generation

This procedure is part of the `plan` skill. Follow these steps after Step 1 (Setup) is complete and the FS is loaded.

> **Remember:** Every scenario has Preconditions/Action/Expected — no exceptions. Each traces to exactly one AC. After each happy path, write failure scenarios. Never combine ACs into one scenario. Cross-service scenarios validate the contract boundary.

---

## Step 2.1: Coverage Analysis

List every AC from the FS. For each AC, classify:

| AC | Type | Cross-Service? | Notes |
|---|---|---|---|
| AC-001 | happy path | no | |
| AC-002 | validation | no | Multiple fields to validate |
| AC-003 | happy path | yes | Depends on inventory service |

Identify:
- Which ACs need integration/cross-service scenarios (external service calls)
- Which ACs imply error paths (stock conflicts, payment failures, validation errors)
- Which services need observability scenarios (health/ready/metrics)

Present the coverage plan to the human: "Here are the ACs and my plan for scenario coverage. Confirm before I write scenarios."

---

## Step 2.2: Generate Scenarios per AC

For each AC in the FS, write test scenarios in order:

1. **Happy path first** — the AC working as intended
2. **Negative scenarios** — what happens when inputs are invalid, services fail, data is missing
3. **Edge cases** — boundary values, concurrency, empty collections, max lengths

Each scenario MUST include:
- **Scenario ID:** `TS-{ts-number}-{sequence}` (e.g., TS-001-003)
- **Preconditions:** System state before the test — what services are running, what data exists, what mocks return
- **Action:** What the user or system does — specific HTTP call with method and path, or UI action
- **Expected outcome:** Observable result that proves the AC is met — HTTP status, response shape, DB state, mock call count

Group scenarios under their parent AC heading.

**After each AC's happy path, ask yourself:** "What goes wrong here?" Write scenarios for:
- Invalid input → validation error
- Missing prerequisite → appropriate error
- External service failure → correct fallback behavior
- External service NOT called when it shouldn't be (e.g., payment not called on stock conflict)

---

## Step 2.3: Cross-Service Integration Scenarios

For each external service call identified in the feature:

1. **Mock returns success** — verify the request shape sent to the mock matches the contract
2. **Mock returns error** — verify the system handles the error correctly (rolls back, returns appropriate error to caller)
3. **Mock NOT called** — verify the service is NOT called when preconditions aren't met (e.g., payment not called when stock reservation fails)

These validate the **contract boundary**, not internal logic. They catch API shape mismatches before Phase 4.

Reference `contracts/api/` (loaded in Step 1) for the exact request/response shapes the mocks should expect and return.

---

## Step 2.4: Observability Scenarios

For each service affected by this feature, add standard scenarios:

- `GET /health` → 200, `{"status": "ok"}`
- `GET /ready` → 200 when dependencies (DB, Redis, etc.) are available
- `GET /metrics` → 200, Prometheus format (`Content-Type` contains `text/plain`)

These are always present per `contracts/architecture/observability-standards.md`. Do not skip them.

---

## Step 2.5: Build Traceability Matrix

Create a table mapping ACs to scenario IDs:

```
| AC | Scenarios |
|---|---|
| AC-001 — [title] | TS-XXX-001, TS-XXX-002 |
| AC-002 — [title] | TS-XXX-003, TS-XXX-004, TS-XXX-005 |
| Observability (all services) | TS-XXX-NNN, TS-XXX-NNN, TS-XXX-NNN |
```

**Self-check before proceeding:**
- Every AC has at least one scenario
- Every scenario traces to exactly one AC (or Observability)
- No orphan scenarios (scenarios without a parent AC)
- Scenario IDs are sequential with no gaps

---

## Step 2.6: Assemble

1. Assemble the full TS using the template structure (loaded in Step 1)
2. Place the traceability matrix at the top, before the scenarios
3. Group scenarios under their parent AC heading
4. Run the verification checklist below
5. Write TS to file immediately (the main SKILL.md handles ID and path)
6. Proceed to Phase 3

---

## Verification Checklist — Test Spec

Run before presenting. Every item must pass.

- [ ] Every AC in the FS has at least one test scenario
- [ ] Every test scenario traces back to exactly one AC (cited in heading)
- [ ] No orphan scenarios (scenarios without a parent AC)
- [ ] Preconditions, Action, and Expected outcome are explicit in every scenario
- [ ] Negative and edge-case scenarios included where ACs imply them
- [ ] Cross-service scenarios validate contract boundaries (request shape, error handling, not-called assertions)
- [ ] Observability scenarios present for each affected service
- [ ] Domain terminology matches `plan/reference/glossary.md`
- [ ] Traceability matrix is complete and consistent with scenario list
- [ ] Scenario IDs follow the pattern `TS-{number}-{sequence}` with no gaps or collisions
- [ ] No scenarios test implementation details (internal function calls, DB queries) — only observable behavior
