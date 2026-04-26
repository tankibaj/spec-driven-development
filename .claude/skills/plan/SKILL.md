---
name: plan
description: Generates Test Spec and Work Packages from an approved Feature Spec. Use after FS is approved, covers Phase 2 (TS) and Phase 3 (WPs) in sequence.
argument-hint: "[feature-id] [slug]"
allowed-tools: Read Glob
disable-model-invocation: true
---

# SDD Plan — Test Spec + Work Packages

## Overview

Generate a Test Spec (Phase 2) and Work Packages (Phase 3) from an approved Feature Spec. Before any generation, run a pre-flight check that validates the IA and FS for completeness, consistency, and approval status.

The TS ensures every AC is testable. The WPs ensure every test scenario is implementable by an AI agent in a single, self-contained work unit.

## When to Use

- After a Feature Spec (FS) and Impact Analysis (IA) have been approved
- When ready to derive test scenarios and split work for implementation

**Requires:** Approved FS-XXX.md and IA-XXX.md in the feature folder (`spec/XXX-slug/`).

**When NOT to use:** Before FS approval (use `/spec`), during implementation (Phase 4), for drafting the FS itself.

## Standing Instructions

These apply at every step across both phases:

- Never invent content beyond what the FS contains. If you think something is missing, flag it in the pre-flight report.
- Every AC in the FS must be covered. No AC left behind, no orphan scenarios, no unassigned WPs.

**Phase 2 — Test Spec:**
- Every scenario MUST have Preconditions, Action, Expected outcome — no exceptions.
- Each scenario traces to exactly one AC. Never combine multiple ACs into one scenario.
- After each happy-path scenario, generate negative/edge-case scenarios.
- Cross-service scenarios must validate the contract boundary — correct request shape to the mock, correct behavior on mock error.

**Phase 3 — Work Packages:**
- Copy ACs and TS scenarios VERBATIM into each WP — never paraphrase or reference by ID only.
- Maximum 3 ACs per WP. One AC per WP is ideal.
- Each WP targets exactly one workspace. Never mix backend and frontend.
- Include contract excerpts directly in the WP. The implementation agent should not need to read the workspace `docs/` specs.
- State dependency order explicitly between WPs.
- Mark parallel opportunities where WPs have no dependencies.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This AC only needs a happy path test" | If the AC can fail, it needs a failure scenario. Implementation agents test only what's in the TS. |
| "The edge case is obvious" | Undocumented = untested. The WP agent sees only the TS scenarios. |
| "I'll combine related ACs into one scenario" | Each scenario traces to exactly one AC. Combined scenarios hide which AC failed. |
| "Contract testing is separate" | Integration scenarios validating contract boundaries belong in the TS. |
| "Observability tests can wait" | Health/ready/metrics are in the DoD. Write them now or they'll be forgotten. |
| "This AC is too high-level to test" | If it's untestable, it shouldn't be in the FS. Flag it in the pre-flight report. |
| "4 ACs is fine, they're small" | Max 3. More context = more hallucination risk for the implementation agent. |
| "The implementer can look up the contract" | No. Include the relevant paths/schemas directly in the WP. |
| "I'll reference TS-001-005 instead of copying it" | Copy verbatim. The implementation agent reads only the WP. |
| "Contract excerpts will make the WP too long" | A 400-line self-contained WP beats a 200-line WP that requires reading 3 other files. |

## Red Flags

Stop and reassess if you catch yourself doing any of these:

**Phase 2:**
- Writing scenarios without Preconditions, Action, or Expected outcome
- Multiple ACs covered by a single scenario
- No negative/edge-case scenarios for a happy-path AC
- Scenarios that test implementation details rather than behavior
- Missing cross-service scenarios when the FS touches multiple services
- Skipping observability scenarios (health/ready/metrics)

**Phase 3:**
- WP with more than 3 ACs
- ACs or TS scenarios referenced by ID instead of copied verbatim
- WP targeting multiple workspaces
- Missing dependency order between BE and FE WPs
- Contract referenced by path but not excerpted
- A TS scenario that appears in no WP

---

## The Workflow

```
PRE-FLIGHT ──→ PHASE 2: TEST SPEC ──→ PHASE 3: WORK PACKAGES ──→ WRITE FILES ──→ HUMAN REVIEWS
     │                │                        │                       │               │
     ▼                ▼                        ▼                       ▼               ▼
  Validate IA      Generate all            Generate all WPs        Save all         Human approves
  + FS quality     scenarios in            in one pass             as awaiting      or requests
  Report issues    one pass                                        review           changes
```

Pre-flight is the gate. If it fails, generation does not start.

---

### Step 1: Setup

If arguments provided: `$0` = feature-ID, `$1` = slug. Otherwise:
1. List existing folders in `spec/` and offer them if any match the context
2. Ask the human: "Which feature folder?"

**Output location:** `spec/{$0}-{$1}/` — all artifacts (TS, WPs) are saved to this folder.

Steps:
1. Navigate to the feature folder
2. Read `status.yaml`
3. Read the FS and IA in full
4. Load domain context: `docs/reference/glossary.md`, `personas.md`, `roles.md`
5. Load all skill reference files:
   - `${CLAUDE_SKILL_DIR}/phases/phase-2-test-spec.md` — TS generation procedure
   - `${CLAUDE_SKILL_DIR}/phases/phase-3-work-packages.md` — WP generation procedure
   - `${CLAUDE_SKILL_DIR}/templates/ts-template.md` — TS document structure
   - `${CLAUDE_SKILL_DIR}/templates/wp-be-template.md` — Backend WP structure
   - `${CLAUDE_SKILL_DIR}/templates/wp-fe-template.md` — Frontend WP structure
   - `${CLAUDE_SKILL_DIR}/references/splitting-guide.md` — AC-to-WP splitting criteria
6. Load `routes.yaml` — needed for workspace mapping
7. Read auto-generated specs from workspace `docs/` dirs (paths in `routes.yaml` `contracts` field) — needed for contract excerpts (read-only, never modify):
     - Backend: `workspaces/{service}/docs/api/openapi.json` — OpenAPI specs
     - Backend: `workspaces/{service}/docs/schema/entities.md` — entity schemas
     - Frontend: `workspaces/{app}/docs/routes.md` — route manifest
     - Frontend: `workspaces/{app}/docs/consumed-endpoints.md` — consumed backend endpoints
8. Check the feature folder for existing TS and WP IDs to avoid collisions

---

### Step 2: Pre-Flight Check

Validate the IA and FS before generating anything. Run every check below. Collect all failures — do not stop at the first one.

#### Approval Status

| Check | How | Fail Action |
|---|---|---|
| PRD is approved | `status.yaml` `artifacts.PRD-XXX.status == approved` | BLOCK |
| IA is approved | `status.yaml` `artifacts.IA-XXX.status == approved` | BLOCK |
| FS is approved | `status.yaml` `artifacts.FS-XXX.status == approved` | BLOCK |

#### FS Quality

| Check | How | Fail Action |
|---|---|---|
| No unresolved open questions | FS Open Questions section has no unchecked `[ ]` items tagged `[BLOCKS APPROVAL]` | BLOCK |
| All assumptions acknowledged | FS Assumptions section exists and is non-empty | WARN |
| Out of Scope present | FS Out of Scope section exists and is non-empty | WARN |

#### IA ↔ FS Alignment

| Check | How | Fail Action |
|---|---|---|
| Every service in IA has ≥1 AC | Cross-reference IA "Affected Services" with ACs | BLOCK — missing AC or wrong IA |
| No AC references a service not in IA | Cross-reference ACs with IA services | WARN — IA may be incomplete |
| Contract changes in IA match FS Related Contracts | Compare IA contract changes with FS Related Contracts section | WARN |

#### AC Consistency

| Check | How | Fail Action |
|---|---|---|
| Every AC has a Testable: line | Scan all AC-XXX entries | BLOCK — untestable AC |
| ACs reference existing or flagged contracts | Check each endpoint/entity reference against workspace `docs/` specs | WARN — may indicate missing contract |
| Resolved open questions reflected in ACs | If PRD open questions were resolved, verify ACs are consistent with those decisions | WARN |
| No vague ACs | ACs use behavioral language with concrete conditions | WARN |

#### Dependency Check (if FS has `depends_on`)

| Check | How | Fail Action |
|---|---|---|
| Dependencies approved | Check dependency `status.yaml` | BLOCK for BE WPs; FE may proceed with mocks |

#### Report

Present findings as:

```
PRE-FLIGHT CHECK: [Feature Name]

✅ PASSED:
  - [check]: [detail]

❌ BLOCKED:
  - [check]: [detail] — [what needs to happen]

⚠️ WARNINGS:
  - [check]: [detail] — [recommendation]
```

**Decision:**

```
IF any BLOCK exists:
  → STOP. Tell the human what needs fixing. Do not generate.

IF only WARNINGS exist:
  → Present warnings. Ask: "Proceed with generation despite warnings?"
  → If human confirms, continue. If not, stop.

IF all checks pass:
  → Proceed to Phase 2.
```

---

### Phase 2: Test Spec Generation

Follow the procedure in `phases/phase-2-test-spec.md` (loaded in Step 1). Generate all scenarios autonomously in one pass — do not stop to ask the human.

After completing Phase 2:

1. Determine the next available `TS-XXX` ID
2. Write `TS-XXX.md` to the feature folder immediately
3. Proceed directly to Phase 3

---

### Phase 3: Work Package Generation

Follow the procedure in `phases/phase-3-work-packages.md` (loaded in Step 1). Use the TS from Phase 2 as input. Generate all WPs autonomously in one pass.

After completing Phase 3:

1. Determine the next available `WP-XXX` IDs (one per WP, using `-BE` or `-FE` suffix)
2. Write all `WP-XXX-BE.md` and/or `WP-XXX-FE.md` to the feature folder
3. State the recommended implementation order:

```
IF contracts for all required endpoints exist and are stable:
  → PARALLEL: "WP-XXX-BE and WP-XXX-FE can run simultaneously.
    FE uses Prism or MSW against workspaces/{service}/docs/api/openapi.json for mocking."

IF FE depends on new endpoints being built in this feature:
  → SEQUENTIAL: "WP-XXX-BE first. WP-XXX-FE second.
    FE mocks against the OpenAPI spec until BE is deployed."
```

---

### Step 5: Finalize and Present

1. Update `status.yaml` — all new artifacts as `awaiting_review`:

```yaml
current_phase: 3
artifacts:
  PRD-XXX: { status: approved }
  IA-XXX: { status: approved }
  FS-XXX: { status: approved }
  TS-XXX: { status: awaiting_review, date: YYYY-MM-DD }
  WP-XXX-BE: { status: awaiting_review, date: YYYY-MM-DD }
  WP-XXX-FE: { status: awaiting_review, date: YYYY-MM-DD }
```

2. Present a summary to the human:
   - Pre-flight check result (all passed / warnings acknowledged)
   - TS coverage: number of scenarios, traceability matrix
   - WP split: number of WPs, workspace assignments, dependency order
   - Any warnings or concerns from generation
   - "TS and WPs written as awaiting_review. Review all files and update `status.yaml` when approved."

3. **STOP. Wait for human to review and approve.**

After the human approves, update `status.yaml`:

```yaml
current_phase: 4
artifacts:
  TS-XXX: { status: approved, date: YYYY-MM-DD }
  WP-XXX-BE: { status: approved, date: YYYY-MM-DD }
  WP-XXX-FE: { status: approved, date: YYYY-MM-DD }
phase_4:
  WP-XXX-BE: { status: not_started }
  WP-XXX-FE: { status: not_started }
```

Tell the human: "All artifacts approved. Phase 4 (implementation) is ready."

---

### Implementation Agent Gate

The Phase 4 implementation agent MUST verify before starting any WP:

1. `current_phase` is `4`
2. The WP's status under `artifacts` is `approved`
3. All prerequisite artifacts are `approved`: PRD, IA, FS, TS
4. If the WP has dependencies on other WPs (e.g., FE depends on BE), the dependency WP's `phase_4` status is `done`

If any check fails, the agent reports what is missing and does not proceed.
