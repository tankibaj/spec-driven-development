# Test Spec Template

Structure for TS-XXX.md documents. The agent uses this as the reference format when assembling the Test Spec in Phase 2.

---

## Document Structure

| Section | Purpose | Required? |
|---|---|---|
| **Header block** | Feature, Based on, Generated, Status | Yes |
| **Traceability matrix** | AC → Scenario ID mapping table | Yes |
| **Scenarios** | Grouped by AC, each with Preconditions/Action/Expected | Yes |
| **Observability** | Standard health/ready/metrics scenarios per service | Yes |

---

## Scenario Format

Each scenario follows this structure:

### Full Format (in the TS document)

```
### AC-XXX — [AC title]

#### TS-{number}-{sequence} — [Scenario title]

**Preconditions:**
- [System state: what services are running]
- [Data state: what exists in DB/cache, what mocks return]
- [Authentication state: tokens, headers]

**Action:**
- [Specific HTTP call: `METHOD /path` with body/headers]
- OR [UI action: what the user does]

**Expected outcome:**
- [HTTP status code]
- [Response body shape and key fields]
- [DB/cache state changes (or no changes)]
- [Mock call assertions: called N times, NOT called, request shape]
```

### Compact Format (when copied into WPs)

```
### TS-{number}-{sequence} — [Scenario title]
- **Preconditions:** [condensed to one line or short list]
- **Action:** [single line: method, path, key payload]
- **Expected:** [single line: status, key assertions]
```

---

## Template

```markdown
# TS-XXX: {Feature Title} — Test Spec

**Feature:** Story-XXXX-{slug}
**Based on:** FS-XXX (all N ACs)
**Generated:** YYYY-MM-DD
**Status:** awaiting_review

---

## Traceability matrix

| AC | Scenarios |
|---|---|
| AC-001 — [title] | TS-XXX-001, TS-XXX-002 |
| AC-002 — [title] | TS-XXX-003, TS-XXX-004, TS-XXX-005 |
| Observability (all services) | TS-XXX-NNN, TS-XXX-NNN, TS-XXX-NNN |

---

## Scenarios

---

### AC-001 — [AC title]

#### TS-XXX-001 — [Happy path scenario title]

**Preconditions:**
- [state]

**Action:**
- [action]

**Expected outcome:**
- [result]

---

#### TS-XXX-002 — [Negative/edge case scenario title]

**Preconditions:**
- [state]

**Action:**
- [action]

**Expected outcome:**
- [result]

---

### Observability

#### TS-XXX-NNN — GET /health → 200

**Preconditions:** [service] is running.

**Action:** `GET /health`

**Expected outcome:** HTTP 200; `{"status": "ok"}`.

---

#### TS-XXX-NNN — GET /ready → 200

**Preconditions:** Dependencies (DB, Redis, etc.) are healthy.

**Action:** `GET /ready`

**Expected outcome:** HTTP 200; `body.status == "ready"`.

---

#### TS-XXX-NNN — GET /metrics → 200

**Preconditions:** [service] is running.

**Action:** `GET /metrics`

**Expected outcome:** HTTP 200; Content-Type contains `text/plain`.
```
