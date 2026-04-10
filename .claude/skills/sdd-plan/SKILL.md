---
name: sdd-plan
description: Generates Test Spec and Work Packages from an approved Feature Spec. Use after FS is approved, covers Phase 2 (TS) and Phase 3 (WPs) in sequence.
argument-hint: "[story-id] [slug]"
allowed-tools: Read Glob
disable-model-invocation: true
---

# SDD Plan — Test Spec + Work Packages

## Overview

Generate a Test Spec (Phase 2) and Work Packages (Phase 3) from an approved Feature Spec. Both phases run in sequence within one session. The human reviews both artifacts together at the end.

The TS ensures every AC is testable. The WPs ensure every test scenario is implementable by an AI agent in a single, self-contained work unit.

## When to Use

- After a Feature Spec (FS) has been approved
- When ready to derive test scenarios and split work for implementation

**Requires:** An approved FS-XXX.md in the feature folder (`plan/spec/Story-XXXX/`).

**When NOT to use:** Before FS approval (use `/sdd-feature-spec`), during implementation (Phase 4), for drafting the FS itself.

## Standing Instructions

These apply at every step across both phases:

- After completing each step, state: "Step N done. Moving to Step N+1: [name]."
- If you cannot resolve a question or blocker, add it as an Open Question with a `[BLOCKED: Step N]` tag and continue. Do not stop the entire workflow for one unresolved item.
- Never invent content beyond what the FS contains. If you think something is missing, flag it to the human.
- Every AC in the FS must be covered. No AC left behind, no orphan scenarios, no unassigned WPs.

**Phase 2 — Test Spec:**
- Every scenario MUST have Preconditions, Action, Expected outcome — no exceptions.
- Each scenario traces to exactly one AC. Never combine multiple ACs into one scenario.
- After each happy-path scenario, ask: "What goes wrong here?" Write negative/edge-case scenarios.
- Cross-service scenarios must validate the contract boundary — correct request shape to the mock, correct behavior on mock error.

**Phase 3 — Work Packages:**
- Copy ACs and TS scenarios VERBATIM into each WP — never paraphrase or reference by ID only.
- Maximum 3 ACs per WP. One AC per WP is ideal.
- Each WP targets exactly one workspace. Never mix backend and frontend.
- Include contract excerpts directly in the WP. The implementation agent should not need to read `contracts/`.
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
| "This AC is too high-level to test" | If it's untestable, it shouldn't be in the FS. Flag it back to the human. |
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

## The Workflow

```
LOAD FS ──→ PHASE 2: TEST SPEC ──→ PHASE 3: WORK PACKAGES ──→ HUMAN REVIEW (both TS + WPs) ──→ DONE
```

### Step 1: Setup

If arguments provided: `$0` = Story-ID, `$1` = slug. Otherwise:
1. List existing folders in `plan/spec/` and offer them if any match the context
2. Ask the human: "Which feature folder?"

**Output location:** `plan/spec/Story-{$0}-{$1}/` — all artifacts (TS, WPs) are saved to this folder.

Steps:
1. Navigate to the feature folder
2. Verify FS exists and is approved in `status.yaml`
3. If the FS has a `depends_on` field, check `status.yaml` for each dependency:
   - All dependencies at `phase_4: done` → proceed normally
   - Any dependency `in_progress` → flag to human; FE WPs can proceed with contract mocks, BE WPs may be blocked
   - Any dependency pre-phase-4 or missing → ask the human whether to block or proceed with mocks
4. Read the FS in full — this is your primary input for both phases
4. Load domain context: `plan/reference/glossary.md`, `personas.md`, `roles.md`
5. Load all skill reference files:
   - `${CLAUDE_SKILL_DIR}/phases/phase-2-test-spec.md` — TS generation procedure
   - `${CLAUDE_SKILL_DIR}/phases/phase-3-work-packages.md` — WP generation procedure
   - `${CLAUDE_SKILL_DIR}/templates/ts-template.md` — TS document structure
   - `${CLAUDE_SKILL_DIR}/templates/wp-be-template.md` — Backend WP structure
   - `${CLAUDE_SKILL_DIR}/templates/wp-fe-template.md` — Frontend WP structure
   - `${CLAUDE_SKILL_DIR}/references/splitting-guide.md` — AC-to-WP splitting criteria
6. Load `registry/routes.yaml` — needed for workspace mapping in Phase 3
7. Scan `contracts/api/` and `contracts/data-schema/` — needed for contract excerpts in Phase 3
8. Check the feature folder for existing TS and WP IDs to avoid collisions
9. Ask the human: "Should I create a feature branch `spec/{Story-ID}-{slug}` for this work, or will you manage branching?" Ask once — Phase 3 continues on the same branch.

If no approved FS exists:

```
BLOCKED: No approved Feature Spec found in this feature folder.
→ Use /sdd-feature-spec to create one first.
```

### Phase 2: Test Spec Generation

Follow the procedure in `phases/phase-2-test-spec.md` (loaded in Step 1).

After completing Phase 2:

1. Determine the next available `TS-XXX` ID
2. Write `TS-XXX.md` to the feature folder immediately
3. Proceed directly to Phase 3

### Phase 3: Work Package Generation

Follow the procedure in `phases/phase-3-work-packages.md` (loaded in Step 1). Use the TS from Phase 2 as input.

After completing Phase 3:

1. Determine the next available `WP-XXX` IDs (one per WP, using `-BE` or `-FE` suffix)
2. Write `WP-XXX-BE.md` and/or `WP-XXX-FE.md` to the feature folder
3. State the recommended implementation order:

```
IF contracts for all required endpoints exist and are stable:
  → PARALLEL: "WP-XXX-BE and WP-XXX-FE can run simultaneously.
    FE uses Prism or MSW against contracts/api/{service}.openapi.yaml for mocking."

IF FE depends on new endpoints being built in this feature:
  → SEQUENTIAL: "WP-XXX-BE first. WP-XXX-FE second.
    FE mocks against the OpenAPI spec until BE is deployed."
```

4. Update `status.yaml` — all new artifacts as `awaiting_review`:

```yaml
current_phase: 3
artifacts:
  FS-XXX: { status: approved }
  TS-XXX: { status: awaiting_review, date: YYYY-MM-DD }
  WP-XXX-BE: { status: awaiting_review, date: YYYY-MM-DD }
  WP-XXX-FE: { status: awaiting_review, date: YYYY-MM-DD }
```

5. Present ALL artifacts to the human for review: the TS and every WP.
6. **STOP. Wait for human to review and approve.**

### Finalize

After the human approves, update `status.yaml`:

```yaml
current_phase: 4
artifacts:
  FS-XXX: { status: approved }
  TS-XXX: { status: approved, date: YYYY-MM-DD }
  WP-XXX-BE: { status: approved, date: YYYY-MM-DD }
  WP-XXX-FE: { status: approved, date: YYYY-MM-DD }
phase_4:
  WP-XXX-BE: { status: not_started }
  WP-XXX-FE: { status: not_started }
```

Tell the human: "All artifacts approved. Phase 4 (implementation) is ready."

### Implementation Agent Gate

The Phase 4 implementation agent MUST verify before starting any WP:

1. `current_phase` is `4`
2. The WP's status under `artifacts` is `approved`
3. All prerequisite artifacts are `approved`: PDR, FS, TS
4. If the WP has dependencies on other WPs (e.g., FE depends on BE), the dependency WP's `phase_4` status is `done`

If any check fails, the agent reports what is missing and does not proceed.
