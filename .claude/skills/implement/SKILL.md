---
name: implement
description: Orchestrates Phase 4 implementation — validates, plans execution order, and delegates each WP to an executor sub-agent with minimal context. Use after TS and WPs are approved (Phase 4).
argument-hint: "[feature-id] [slug] [--scope all|be|fe|WP-XXX-YY] [--validate]"
---

# SDD Implement — Phase 4 Orchestrator

## Overview

Orchestrate the implementation of approved Work Packages for a feature. Validate prerequisites, determine execution order and parallelism, and delegate each WP to an executor sub-agent.

This skill handles planning and coordination only. It does NOT implement code. Each WP is executed by a sub-agent loaded with `executor-protocol.md` and the minimal context needed for that single WP.

When multiple WPs are independent and target different workspaces, launch executor sub-agents in parallel. This is a judgment call — not forced. Sequential execution is fine when hard dependencies exist.

## When to Use

- After the human approves TS and all WPs (`status.yaml` shows `current_phase: 4`)
- When all prerequisite artifacts (PRD, IA, FS, TS, WPs) are `approved`

**When NOT to use:** Before WP approval, for spec drafting (use `/spec`), for TS/WP generation (use `/plan`).

## Execution Modes

| Mode | Trigger | What runs |
|---|---|---|
| Full feature | `--scope all` (default) | Every non-done WP |
| Backend only | `--scope be` | WPs targeting `type: backend` workspaces |
| Frontend only | `--scope fe` | WPs targeting `type: frontend` workspaces |
| Single WP | `--scope WP-XXX-YY` | That one WP only |
| Validation | `--validate` | Runs FE tests against real BE (no MSW) |

---

## The Workflow

```
SETUP ──→ PRE-FLIGHT ──→ SURVEY ──→ DELEGATE WPs ──→ MONITOR ──→ COMPLETE
  │            │            │            │               │            │
  ▼            ▼            ▼            ▼               ▼            ▼
Parse scope  Validate     Sub-agent    Launch executor  Track status, Report to
+ load       approvals,   reads WP     sub-agents with  relay blocks, human
routes       contracts,   headers +    minimal context   launch next
             workspaces   deps only                      wave
```

For `--validate` mode: SETUP → PRE-FLIGHT → VALIDATE → REPORT (skips survey/delegate/monitor).

Pre-flight is the gate. If it fails, no delegation happens.

---

### Step 1: Setup

Parse arguments: `$0` = feature-ID, `$1` = slug (optional), `--scope` (default: `all`), `--validate` flag.

If no arguments: list folders in `spec/` and identify the feature with `current_phase: 4`.

Steps:
1. Navigate to the feature folder under `spec/`
2. Read `status.yaml` — verify `current_phase` is `4`
3. Parse `--scope` argument. Valid values: `all` (default), `be`, `fe`, or a specific WP ID (e.g., `WP-003-FE`)
4. If `--validate` flag is present → skip to Step 6 (Validation Mode)
5. Load `routes.yaml` (root) — map workspace IDs to types (`backend` / `frontend`)
6. Load orchestrator reference:
   - `${CLAUDE_SKILL_DIR}/references/checkpoint-taxonomy.md` — for understanding checkpoint state across all WPs
7. If `--scope` is a specific WP ID → read only that single WP file (for pre-flight and direct delegation — skip survey in Step 3)
8. If `--scope` is `all`, `be`, or `fe` → do NOT read WP files here. The survey sub-agent handles dependency analysis in Step 3.

Do NOT load executor references at this level (`be-guide.md`, `fe-guide.md`, `resume-protocol.md`, `executor-protocol.md`). Each executor sub-agent loads its own.

---

### Step 2: Pre-Flight Gate

Validate before delegating anything. Run every check. Collect all failures — do not stop at the first one.

#### Artifact Approval

| Check | How | Fail Action |
|---|---|---|
| `current_phase` is `4` | `status.yaml` | BLOCK |
| FS is approved | `status.yaml` `artifacts.FS-XXX.status == approved` | BLOCK |
| IA is approved (if exists) | `status.yaml` `artifacts.IA-XXX.status == approved` | BLOCK |
| TS is approved | `status.yaml` `artifacts.TS-XXX.status == approved` | BLOCK |
| Every in-scope WP is approved | `status.yaml` `artifacts.WP-XXX.status == approved` for each WP matching the scope | BLOCK |

#### Contract Verification

Contract verification is deferred to each executor sub-agent during implementation. The orchestrator does not read full WP contents (the survey agent reads only headers and dependency sections). Each executor validates its own WP's contract excerpts against the workspace's actual contract files.

#### Workspace Readiness

| Check | How | Fail Action |
|---|---|---|
| Target workspace registered | `routes.yaml` has the workspace ID | BLOCK |
| No other agent active in workspace | No other WP targeting the same workspace has `phase_4` status `in_progress` | BLOCK — wait or pick a different WP |

#### Cross-Feature Dependencies

| Check | How | Fail Action |
|---|---|---|
| `depends_on` features are done | If FS declares `depends_on`, check dependency `status.yaml` | BLOCK for BE WPs. FE may proceed if WP includes mock instructions |

#### Single-WP Scope Additional Checks

When `--scope WP-XXX-YY`:
- Validate that specific WP's approval status
- Parse the WP's dependency section and verify all **hard** dependencies (same-type) are `done`
- **Soft** dependencies (cross-type) do not block — see Dependency Classification below

#### Decision

```
IF any BLOCK exists:
  → STOP. Report what's failing. Do not start implementation.

IF only WARNINGS exist:
  → Note them in status.yaml. Proceed — warnings don't block execution.

IF all checks pass:
  → Proceed to Step 3.
```

---

### Step 3: Survey + Execution Plan

Two sub-steps: survey dependencies, then delegate executors.

#### Step 3a: Dependency Survey

**For `--scope all|be|fe`:** Launch a survey sub-agent to build the execution plan. This keeps the orchestrator's context lean — it never reads full WP files.

Launch the survey sub-agent (Agent tool) with this prompt:

```
Survey Work Package Dependencies: {feature-ID}

You are a lightweight survey agent. Scan WP files and produce a structured execution plan.
You do NOT implement anything.

## Instructions

1. List all WP-*.md files in spec/{feature-folder}/
2. Read status.yaml at spec/{feature-folder}/status.yaml
3. For each WP file, read ONLY:
   - The header block (title, Feature, Target workspace, Status lines)
   - The "Dependency Info" section (search for ## Dependency, ## Dependencies, or ### Dependency)
   - STOP reading after the dependency section. Do NOT read ACs, Test Scenarios, Contracts,
     Implementation Order, or any other section.
4. Cross-reference each WP's Target workspace with routes.yaml to determine type:
   Routes file: routes.yaml
5. Apply scope filter: --scope {scope}
   - "all": keep all WPs
   - "be": keep only WPs targeting workspaces with type: backend
   - "fe": keep only WPs targeting workspaces with type: frontend
6. Skip WPs whose phase_4 status in status.yaml is "done"
   - WPs with status "done_with_mocks" are included (they may need re-validation later)
7. For each in-scope WP, classify dependencies:
   - HARD: dependency is same type (BE→BE or FE→FE) — downstream MUST wait
   - SOFT: dependency is cross-type (FE→BE or BE→FE) — downstream proceeds with mocks
   - Out-of-scope dependencies are always SOFT
8. Compute wave ordering based on HARD dependencies only:
   - Wave 1: WPs with no hard dependencies (or whose hard deps are all done/done_with_mocks)
   - Wave 2: WPs whose hard deps are all in wave 1 or already done
   - Continue until all WPs are assigned
9. Return the plan in this exact format:

EXECUTION_PLAN_START
wave_1:
  - id: WP-XXX-YY
    workspace: {workspace-id}
    type: {backend|frontend}
    hard_deps: []
    soft_deps: []
  - id: WP-XXX-YY
    workspace: {workspace-id}
    type: {frontend}
    hard_deps: []
    soft_deps: [WP-XXX-YY]
wave_2:
  - id: WP-XXX-YY
    workspace: {workspace-id}
    type: {frontend}
    hard_deps: [WP-XXX-YY]
    soft_deps: [WP-XXX-YY]
EXECUTION_PLAN_END

## Files
- Feature folder: spec/{feature-folder}/
- Status file: spec/{feature-folder}/status.yaml
- Routes file: routes.yaml

## Constraints
- Do NOT read full WP files. Only headers + dependency sections.
- Do NOT modify any files.
- Return ONLY the execution plan in the specified format.
```

**For `--scope WP-XXX-YY`:** Skip survey. The orchestrator already read the single WP in Step 1. Build a trivial plan:

```
wave_1:
  - id: {WP-ID}
    workspace: {workspace-id from WP header}
    type: {backend|frontend from routes.yaml}
    hard_deps: []
    soft_deps: [{any cross-type deps from WP}]
```

#### Dependency Classification Rules

These rules are applied by the survey sub-agent, and also by the orchestrator for single-WP scope:

| Dependency direction | Classification | Behavior |
|---|---|---|
| BE → BE | **Hard** | Downstream must wait for upstream to complete |
| FE → FE | **Hard** | Downstream must wait for upstream to complete |
| FE → BE | **Soft** | FE proceeds immediately with MSW mocks |
| BE → FE | **Soft** | BE proceeds (rare case) |

Scope interaction:
- `--scope all`: cross-type deps are soft, same-type deps are hard
- `--scope fe`: all BE deps are automatically soft (BE WPs are out of scope)
- `--scope be`: all FE deps are automatically soft (FE WPs are out of scope)

The contract excerpts already embedded in each WP provide everything needed to build mocks. FE WPs have OpenAPI path + schema excerpts sufficient for MSW handlers. No additional mock setup is required.

#### Step 3b: Delegate

Using the survey output (or trivial plan for single WP), delegate executor sub-agents:

```
IF only one WP in the plan:
  → Delegate to a single executor sub-agent.

IF multiple WPs in wave 1 AND they target different workspaces:
  → Launch executor sub-agents in parallel (one per WP).

IF multiple WPs target the same workspace:
  → Execute sequentially within that workspace (one agent per workspace at a time).

After each wave completes:
  → Read status.yaml. Check for blockers or newly unblocked WPs.
  → Launch next wave of sub-agents for WPs whose hard deps are now met.

IF a WP blocks during execution:
  → Downstream WPs with a HARD dependency on the blocked WP → HOLD.
  → Downstream WPs with only a SOFT dependency or NO dependency → continue.
  → Report the blocked WP to the human with recommendation.
```

#### Sub-Agent Handoff Protocol

For each WP to execute, launch a sub-agent (Agent tool) with this prompt structure:

```
Implement Work Package: {WP-ID}

You are an executor implementing a single Work Package. Read and follow the executor protocol strictly.

## Files to Load

1. .claude/skills/implement/references/executor-protocol.md
   — Your operating instructions. Read in full before starting.

2. .claude/skills/implement/references/{be-guide.md OR fe-guide.md}
   — Checkpoint procedures for {backend OR frontend} workspaces.

3. spec/{feature-folder}/{WP-ID}.md
   — The Work Package. This is your specification.

4. workspaces/{workspace-id}/CLAUDE.md
   — Workspace context: tech stack, API surface, directory layout, dev commands.

{IF resuming — include this line:}
5. .claude/skills/implement/references/resume-protocol.md
   — Resume procedure. Your last checkpoint was: "{last_checkpoint value}"

## Context

- Workspace path: workspaces/{workspace-id}
- Workspace type: {backend OR frontend}
- Status file: spec/{feature-folder}/status.yaml
- Your status entry: phase_4.{WP-ID}
- Mode: {Fresh start OR Resume from: {last_checkpoint}}

{IF WP has unmet soft (cross-type) dependencies — include this block:}
## Mock-only Dependencies

The following WPs are not yet deployed. Use MSW mocks for their endpoints.
Contract excerpts are already in your WP's "Relevant Contracts" section.

{For each soft dep that is not done:}
- {WP-ID} ({workspace-id}) — status: {status from status.yaml}
{End for}

When all tests pass and DoD is complete:
- Set your status to `done_with_mocks` (not `done`).
- Add `mock_dependencies: [{list of WP-IDs}]` to your phase_4 status.yaml entry.
{End IF}

## Constraints

- Do NOT read other WP files.
- Do NOT read the FS or TS directly — use the verbatim copies in your WP.
- Do NOT modify files in spec/ or workspace docs/ (except status.yaml checkpoints).
- Write ONLY your own phase_4.{WP-ID} entry in status.yaml.
```

**What the executor does NOT receive:**
- Other WP files (it only sees its own WP)
- This orchestrator SKILL.md
- `routes.yaml` (workspace path already resolved by orchestrator)
- `checkpoint-taxonomy.md` (the be-guide/fe-guide cover checkpoint procedures)
- Any spec-phase reference files
- The survey output or execution plan

**Do not present the execution plan to the human.** Proceed directly to delegation.

---

### Step 4: Monitor and Coordinate

After delegating:

1. Wait for sub-agent completion
2. Read `status.yaml` after each WP completes to check for:
   - Blockers reported by the executor
   - Newly unblocked downstream WPs ready for the next wave (hard deps now met)
3. Launch next wave of executor sub-agents if hard dependencies are now met
4. If an executor reports a blocker:
   - Relay the blocker and recommendation to the human
   - Hold downstream WPs with a hard dependency on the blocked WP
   - Continue WPs with only soft dependencies or no dependency on the blocked WP

---

### Step 5: Completion

After all in-scope WPs are done:

1. Read `status.yaml` — verify every in-scope WP's `phase_4` status is `done` or `done_with_mocks`
2. Report to the human:

```
IMPLEMENTATION COMPLETE: {feature-ID}-{slug} (scope: {scope})

All WPs implemented:
  - WP-001-BE — inventory-service — done
  - WP-002-BE — order-service — done
  - WP-001-FE — storefront-app — done_with_mocks (mocked: WP-001-BE, WP-002-BE)
  ...

{IF any WP is done_with_mocks:}
FE WPs were implemented with MSW mocks for BE dependencies.
Run `/implement {feature-id} --validate` to verify FE against real backends.
{END IF}

{IF all WPs are done (none done_with_mocks):}
All DoD checklists passed. Ready for human review.
{END IF}
```

---

### Step 6: Validation Mode

Triggered by `/implement {feature-id} --validate`. This step verifies that FE implementations work against real BE services instead of MSW mocks.

#### Prerequisites

1. Read `status.yaml`
2. ALL WPs must be either `done` or `done_with_mocks`. If any WP is `not_started`, `in_progress`, or `blocked` → BLOCK with message: "Cannot validate — {WP-ID} is still {status}."
3. Find validation candidates: WPs with `status: done_with_mocks` where ALL entries in `mock_dependencies` now have `status: done`
4. If no candidates exist → report "Nothing to validate. Either all WPs are fully done or mock dependencies are not yet complete." and exit.

#### Validation Execution

For each candidate WP, launch a validation sub-agent (Agent tool) with this prompt:

```
Validate Work Package: {WP-ID} — Integration Tests Against Real Backends

You are a validation agent. Run an existing FE test suite against real backend services
instead of MSW mocks. You do NOT implement new code.

## Files to Load

1. .claude/skills/implement/references/validation-protocol.md
   — Your operating instructions. Read in full before starting.

2. spec/{feature-folder}/{WP-ID}.md
   — The Work Package. Reference for consumed endpoints and test scenarios.

3. workspaces/{fe-workspace-id}/CLAUDE.md
   — Frontend workspace context.

## Context

- Frontend workspace: workspaces/{fe-workspace-id}
- Status file: spec/{feature-folder}/status.yaml
- Your status entry: phase_4.{WP-ID}
- Mock dependencies to validate against:
{For each mock dep:}
  - {WP-ID}: workspaces/{be-workspace-id}
{End for}

## Constraints

- Do NOT modify source code (except adding INTEGRATION guard to tests/setup.ts if missing).
- Do NOT modify test files or MSW handlers.
- Do NOT modify backend code.
- Write ONLY your own phase_4.{WP-ID} entry in status.yaml.
```

#### Result Handling

After each validation sub-agent completes:

```
IF all tests PASS:
  → Update phase_4.{WP-ID}.status to "done"
  → Remove mock_dependencies field
  → Remove validation_failures field if present

IF any tests FAIL:
  → Keep phase_4.{WP-ID}.status as "done_with_mocks"
  → Record validation_failures in status.yaml:
    validation_failures:
      - test: "TS-XXX-YYY"
        endpoint: "GET /products"
        error: "Expected 200, got 500"
  → Report failures to the human with contract mismatch details
```

#### Final Report

After all validation sub-agents complete:

```
VALIDATION COMPLETE: {feature-ID}-{slug}

Results:
  - WP-001-FE — storefront-app — PASS → status: done
  - WP-003-FE — admin-app — FAIL → 2 tests failed (see status.yaml)

{IF all validations passed AND all WPs are now "done":}
All WPs validated. Setting feature_status: feature_complete.
{END IF}

{IF any validations failed:}
Contract mismatches found. Review validation_failures in status.yaml.
{END IF}
```

Update `status.yaml` with `feature_status: feature_complete` when all WPs reach `done`.

---

## status.yaml — Extended Format

### Per-WP Status Values

| Status | Meaning |
|---|---|
| `not_started` | WP has not been delegated yet |
| `in_progress` | Executor sub-agent is currently working |
| `blocked` | WP hit a blocker during implementation |
| `done` | Complete — tested against real deps or has no cross-type deps |
| `done_with_mocks` | FE complete — tested with MSW mocks for BE deps |

### Per-WP Fields

| Field | When present |
|---|---|
| `last_checkpoint` | During `in_progress`. Cleared when WP reaches `done` or `done_with_mocks`. |
| `mock_dependencies` | When status is `done_with_mocks`. List of WP-IDs that were mocked. |
| `validation_failures` | After a failed `--validate` run. Cleared on next successful validation. |

### Feature-Level Status

```yaml
feature_status: feature_complete   # Set when ALL WPs are "done" (none "done_with_mocks")
```

### Example

```yaml
phase_4:
  WP-001-BE:
    status: done
  WP-002-BE:
    status: done
  WP-001-FE:
    status: done_with_mocks
    mock_dependencies: [WP-001-BE, WP-002-BE]
  WP-002-FE:
    status: in_progress
    last_checkpoint: "components — catalog grid and product detail pages"
```

After successful validation:

```yaml
phase_4:
  WP-001-BE:
    status: done
  WP-002-BE:
    status: done
  WP-001-FE:
    status: done
  WP-002-FE:
    status: done

feature_status: feature_complete
```
