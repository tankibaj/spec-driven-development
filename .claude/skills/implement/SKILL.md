---
name: implement
description: Implements approved Work Packages in workspace repos. Executes WPs for a feature — runs parallel agents when independent WPs target different workspaces. Use after TS and WPs are approved (Phase 4).
argument-hint: "[story-id] [slug]"
---

# SDD Implement — Phase 4 Execution

## Overview

Execute approved Work Packages for a feature. Read each WP, implement it in the target workspace, run tests, and verify the Definition of Done.

The WP is the single source of truth. The agent does not deviate from it, does not add scope, and does not modify contracts. Implementation is autonomous — the agent only stops when genuinely blocked.

When multiple WPs are independent and target different workspaces, run them in parallel using sub-agents. This is a judgment call — not forced. Sequential execution is fine when dependencies exist or when the situation is simpler.

## When to Use

- After the human approves TS and all WPs (`status.yaml` shows `current_phase: 4`)
- When all prerequisite artifacts (PRD, IA, FS, TS, WPs) are `approved`

**When NOT to use:** Before WP approval, for spec drafting (use `/spec`), for TS/WP generation (use `/plan`).

## Interaction Model

This skill is **low-interaction**. The WP is an approved spec — execute it.

**Do NOT ask the human for:**
- Confirmation before starting a WP
- Permission to create branches (follow `.claude/rules/branching-strategy.md`)
- Guidance on implementation choices already covered by the WP
- Whether to run WPs in parallel (decide based on dependencies)

**DO stop for the human ONLY when:**
- Pre-flight gate fails (missing approvals, missing contracts)
- Contract conflict discovered during implementation (see Guardrails)
- 3 consecutive failed fix attempts (see Failure Escalation)
- Genuinely ambiguous requirement that cannot be resolved from the WP content

## Standing Instructions

These apply at every step:

- The WP is your specification. Implement exactly what it says — nothing more, nothing less.
- Contracts are read-only. If the code doesn't match the contract, the code is wrong.
- Write tests that map 1:1 to the TS scenarios in the WP. No extra tests, no missing tests.
- The WP's Definition of Done is the only DoD. Do not invent additional checks beyond what the WP specifies.
- Update `status.yaml` checkpoints after every taxonomy category. This is your crash recovery trail.
- If running parallel sub-agents, each agent writes ONLY its own `phase_4.WP-XXX` entry in `status.yaml`. Never modify another WP's entry.
- Commit early and often. Each checkpoint category should have at least one commit.
- Follow `.claude/rules/branching-strategy.md` for branch creation — automatic, no asking.
- Implementation source code is committed only to workspace submodule repos, never to the spec-hub.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This endpoint needs one more field not in the WP" | If it's not in the WP, it's not in scope. Flag it as a suggestion for a future WP. |
| "I'll add this helper utility — it's not in the WP but it's useful" | Useful doesn't mean in-scope. Implement only what the WP requires. |
| "The contract seems wrong, I'll adjust my code to work around it" | Contracts are frozen. STOP, block, recommend. Do not patch around it. |
| "This test is redundant, I'll skip it" | Every TS scenario in the WP gets a test. No exceptions. |
| "I'll fix this later" | There is no later. Checkpoint what works, block what doesn't. |
| "The WP doesn't mention observability, so I'll skip it" | Check the WP's DoD. If it lists observability, implement it. If it doesn't, don't invent it. |
| "I need to read the FS to understand this AC" | The AC is copied verbatim in the WP. If it's still unclear, block — don't go hunting. |
| "This dependency WP isn't done but I can probably work around it" | Check the dependency graph. If the WP depends on it, wait or mock per the WP's instructions. |

## Red Flags

Stop and reassess if you catch yourself doing any of these:

- Implementing features not described in the WP
- Modifying files in `contracts/`, `spec/`, or `reference/` (except `status.yaml`)
- Reading the FS or TS directly instead of the verbatim copies in the WP
- Skipping TS scenarios because they seem redundant
- Continuing past 3 failed fix attempts on the same issue
- Writing to another WP's `phase_4` entry in `status.yaml`
- Starting a WP whose dependencies aren't `done`
- Asking the human for implementation guidance that's already in the WP
- Adding DoD checks that aren't in the WP's Definition of Done
- Committing implementation code to the spec-hub repo

---

## The Workflow

```
SETUP ──→ PRE-FLIGHT ──→ PLAN ──→ IMPLEMENT WPs ──→ FEATURE COMPLETE
  │            │            │           │                   │
  ▼            ▼            ▼           ▼                   ▼
Load WPs    Validate     Determine   Per-WP:             All WPs done,
+ context   approvals,   order +     branch →            report to human
            contracts,   parallelism implement →
            workspaces               test → DoD
```

Pre-flight is the gate. If it fails, implementation does not start.

---

### Step 1: Setup

If arguments provided: `$0` = story-ID, `$1` = slug. Otherwise, list folders in `spec/` and identify the feature with `current_phase: 4`.

Steps:
1. Navigate to the feature folder under `spec/`
2. Read `status.yaml` — verify `current_phase` is `4`
3. Read ALL WP files in the feature folder
4. Load `registry/routes.yaml` — map each WP to its target workspace
5. Load skill reference files:
   - `${CLAUDE_SKILL_DIR}/references/checkpoint-taxonomy.md`
   - `${CLAUDE_SKILL_DIR}/references/resume-protocol.md`
   - `${CLAUDE_SKILL_DIR}/references/be-guide.md` and/or `${CLAUDE_SKILL_DIR}/references/fe-guide.md` based on workspace types involved
6. If resuming an interrupted session: follow the resume protocol

---

### Step 2: Pre-Flight Gate

Validate before implementing anything. Run every check. Collect all failures — do not stop at the first one.

#### Artifact Approval

| Check | How | Fail Action |
|---|---|---|
| `current_phase` is `4` | `status.yaml` | BLOCK |
| FS is approved | `status.yaml` `artifacts.FS-XXX.status == approved` | BLOCK |
| IA is approved (if exists) | `status.yaml` `artifacts.IA-XXX.status == approved` | BLOCK |
| TS is approved | `status.yaml` `artifacts.TS-XXX.status == approved` | BLOCK |
| Every WP is approved | `status.yaml` `artifacts.WP-XXX.status == approved` for each | BLOCK |

#### Contract Verification

| Check | How | Fail Action |
|---|---|---|
| Every contract referenced in WPs exists | Read each WP's "Relevant Contracts" section, verify file exists in `contracts/` | BLOCK |
| Contract content matches WP excerpts | Spot-check key endpoints/schemas — the WP may reference a contract version that has since changed | WARN |

#### Workspace Readiness

| Check | How | Fail Action |
|---|---|---|
| Target workspace registered | `registry/routes.yaml` has the workspace ID | BLOCK |
| No other agent active in workspace | No other WP targeting the same workspace has `phase_4` status `in_progress` | BLOCK — wait or pick a different WP |

#### Cross-Feature Dependencies

| Check | How | Fail Action |
|---|---|---|
| `depends_on` features are done | If FS declares `depends_on`, check dependency `status.yaml` | BLOCK for BE WPs. FE may proceed if WP includes mock instructions |

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

### Step 3: Execution Plan

Determine the order and parallelism for all WPs.

1. **Read dependency info** from each WP (plan puts dependency order in every WP)
2. **Skip completed WPs**: if `phase_4.WP-XXX.status == done`, skip it
3. **Determine execution approach**:

```
IF only one WP remains:
  → Execute it directly (Step 4).

IF multiple WPs remain AND they target different workspaces AND have no dependencies between them:
  → Run them in parallel using sub-agents (one Task per WP).
  → Each sub-agent receives the WP path and runs Step 4 independently.

IF multiple WPs remain with dependencies:
  → Execute in dependency order. Start with WPs that have no unmet dependencies.
  → After each WP completes, check if newly unblocked WPs can run in parallel.

IF a WP blocks during execution:
  → Downstream WPs that DEPEND on the blocked WP → HOLD automatically.
  → Downstream WPs with NO dependency on the blocked WP → continue.
  → Report the blocked WP to the human with recommendation.
```

Proceed directly to execution. Do not present the plan to the human.

---

### Step 4: Implement a Work Package

This is the core execution loop. Runs for each WP, either inline or as a sub-agent.

#### 4.1 Initialize Workspace

1. Check if the workspace submodule exists at the path specified in `registry/routes.yaml`

```
IF workspace directory does not exist or is not a git submodule:
  → ASK the human for the git repository URL for this workspace.
  → Add it as a git submodule: git submodule add <repo-url> workspaces/{workspace-id}
  → Commit the submodule reference in the spec-hub (this is the ONLY spec-hub commit allowed)

IF workspace exists but is empty (no pyproject.toml / package.json):
  → The WP should include scaffold instructions. Begin with the scaffold checkpoint.
  → Reference contracts/architecture/workspace-bootstrap.md for the standard toolchain and structure.
```

2. Navigate to the workspace submodule
3. Read the WP in full — this is your specification
4. Create a feature branch per `.claude/rules/branching-strategy.md`
5. Update `status.yaml`: set `phase_4.WP-XXX.status` to `in_progress`

#### 4.2 Resume Check

```
IF status.yaml phase_4.WP-XXX.last_checkpoint exists:
  → Follow the resume protocol (references/resume-protocol.md)
  → Skip completed checkpoint categories
  → Re-run tests to verify current state before continuing

IF no checkpoint exists:
  → Fresh start. Begin from the first checkpoint category.
```

#### 4.3 Implement

Follow the WP's Implementation Order section. The WP drives implementation.

**Precedence rule:**
1. **WP Implementation Order** — always primary. If the WP specifies an order, follow it exactly.
2. **Reference guide** (`be-guide.md` or `fe-guide.md`) — fills gaps. If the WP doesn't specify how to approach a checkpoint, follow the reference guide's per-checkpoint procedure.
3. **Agent judgment** — last resort. If both the WP and the reference guide are silent, implement the simplest solution that passes the TS scenarios.

If the WP and reference guide contradict each other, the WP wins.

**After each checkpoint category** (see `references/checkpoint-taxonomy.md`), update `status.yaml`:

```yaml
phase_4:
  WP-XXX-BE:
    status: in_progress
    last_checkpoint: "models — ORM models and Alembic migration created"
```

Commit after each checkpoint category. Use the commit message convention from `.claude/rules/branching-strategy.md`.

#### 4.4 Test

Write tests that map 1:1 to the TS scenarios listed in the WP's "Test Scenarios" section.

- Each TS scenario becomes one test function (or one parametrized case)
- Test names reference the scenario ID: `test_ts_001_003_stock_conflict_rejects_order`
- Use the testing patterns from the reference guide (`be-guide.md` or `fe-guide.md`)

Run all tests. All must pass before proceeding to DoD.

#### 4.5 Definition of Done

Run the Definition of Done checklist **from the WP**. The WP's DoD is the only authoritative checklist — do not add, remove, or modify checks.

If every item passes, proceed to completion. If any item fails, fix it. If you can't fix it after 3 attempts, follow Failure Escalation.

#### 4.6 Complete

1. Update `status.yaml`: set `phase_4.WP-XXX.status` to `done`
2. Clear `last_checkpoint` (WP is complete)
3. Return to Step 3 to check if more WPs are ready to execute

---

### Step 5: Feature Completion

After all WPs are done:

1. Verify every WP's `phase_4` status is `done` in `status.yaml`
2. Report to the human:

```
FEATURE COMPLETE: story-XXXX-{slug}

All WPs implemented:
  - WP-001-BE — inventory-service — done
  - WP-002-BE — order-service — done
  - WP-001-FE — storefront-app — done
  ...

All DoD checklists passed. Ready for human review.
```

---

## Guardrails

### Contract Freeze

Contracts are read-only during Phase 4 (per `.claude/rules/contracts-readonly-phase4.md`).

If implementation reveals a contract issue:

1. **STOP** implementation of this WP immediately.
2. Add a blocker to `status.yaml` with a clear description.
3. Set `phase_4.WP-XXX.status` to `blocked`.
4. **Provide a recommendation** to the human:
   - What the contract says vs. what the implementation needs
   - Whether the fix is additive (safe) or breaking (needs ADR)
   - Which consumers would be affected
   - Your suggested resolution
5. Wait for the human to decide. Do not patch around it.
6. Other WPs not affected by this contract issue may continue.

### Failure Escalation

If you attempt to fix a failing test, type error, or linter error and fail **3 consecutive times** on the same issue:

1. Stop attempting. Do not continue cycling through fixes.
2. Revert the affected file to its last working state.
3. Add the issue to `status.yaml` `blockers`:
   - What you tried (all 3 attempts)
   - The exact error message
   - Which TS scenario is affected
4. Set `phase_4.WP-XXX.status` to `blocked`.
5. Report to the human. Three failed attempts means you are missing context not in the WP.

### Scope Enforcement

- Do NOT add features, endpoints, or behaviors not in the WP.
- Do NOT modify files in `spec/`, `reference/`, or `contracts/` during implementation (except `status.yaml`).
- Do NOT read the FS or TS directly — use the verbatim copies in the WP.
- If you think something is missing from the WP, block and report. Do not fill the gap.

### Unblocking a Blocked WP

When the human resolves a blocker:

1. Re-read the WP in full — treat it as a fresh spec (it may have been updated).
2. Resume from the `last_checkpoint` in `status.yaml` — do not re-run completed steps.
3. Clear the resolved entry from `status.yaml` `blockers`.
4. Set `phase_4.WP-XXX.status` back to `in_progress`.
5. Re-run existing tests to verify state before continuing.
