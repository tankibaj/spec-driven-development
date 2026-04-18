---
name: implement-v2
description: Orchestrates Phase 4 implementation — validates, plans execution order, and delegates each WP to an executor sub-agent with minimal context. Use after TS and WPs are approved (Phase 4).
argument-hint: "[story-id] [slug]"
---

# SDD Implement v2 — Phase 4 Orchestrator

## Overview

Orchestrate the implementation of approved Work Packages for a feature. Validate prerequisites, determine execution order and parallelism, and delegate each WP to an executor sub-agent.

This skill handles planning and coordination only. It does NOT implement code. Each WP is executed by a sub-agent loaded with `executor-protocol.md` and the minimal context needed for that single WP.

When multiple WPs are independent and target different workspaces, launch executor sub-agents in parallel. This is a judgment call — not forced. Sequential execution is fine when dependencies exist.

## When to Use

- After the human approves TS and all WPs (`status.yaml` shows `current_phase: 4`)
- When all prerequisite artifacts (PRD, IA, FS, TS, WPs) are `approved`

**When NOT to use:** Before WP approval, for spec drafting (use `/spec`), for TS/WP generation (use `/plan`).

---

## The Workflow

```
SETUP ──→ PRE-FLIGHT ──→ PLAN ──→ DELEGATE WPs ──→ MONITOR ──→ FEATURE COMPLETE
  │            │            │           │               │              │
  ▼            ▼            ▼           ▼               ▼              ▼
Load WPs    Validate     Determine   Launch executor  Track status,  All WPs done,
+ routes    approvals,   order +     sub-agents with  relay blocks,  report to human
            contracts,   parallelism minimal context   launch next
            workspaces                                 wave
```

Pre-flight is the gate. If it fails, no delegation happens.

---

### Step 1: Setup

If arguments provided: `$0` = story-ID, `$1` = slug. Otherwise, list folders in `spec/` and identify the feature with `current_phase: 4`.

Steps:
1. Navigate to the feature folder under `spec/`
2. Read `status.yaml` — verify `current_phase` is `4`
3. Read ALL WP files in the feature folder (for planning and pre-flight — not for implementation)
4. Load `registry/routes.yaml` — map each WP to its target workspace
5. Load orchestrator reference:
   - `${CLAUDE_SKILL_DIR}/references/checkpoint-taxonomy.md` — for understanding checkpoint state across all WPs

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

### Step 3: Execution Plan + Delegation

Determine the order and parallelism for all WPs, then delegate to executor sub-agents.

1. **Read dependency info** from each WP (plan puts dependency order in every WP)
2. **Skip completed WPs**: if `phase_4.WP-XXX.status == done`, skip it
3. **Determine execution approach**:

```
IF only one WP remains:
  → Delegate to a single executor sub-agent.

IF multiple WPs remain AND they target different workspaces AND have no dependencies between them:
  → Launch executor sub-agents in parallel (one per WP).

IF multiple WPs remain with dependencies:
  → Execute in dependency order. Start with WPs that have no unmet dependencies.
  → After each WP completes, check if newly unblocked WPs can run in parallel.

IF a WP blocks during execution:
  → Downstream WPs that DEPEND on the blocked WP → HOLD automatically.
  → Downstream WPs with NO dependency on the blocked WP → continue.
  → Report the blocked WP to the human with recommendation.
```

#### Sub-Agent Handoff Protocol

For each WP to execute, launch a sub-agent (Task tool) with this prompt structure:

```
Implement Work Package: {WP-ID}

You are an executor implementing a single Work Package. Read and follow the executor protocol strictly.

## Files to Load

1. .claude/skills/implement-v2/references/executor-protocol.md
   — Your operating instructions. Read in full before starting.

2. .claude/skills/implement-v2/references/{be-guide.md OR fe-guide.md}
   — Checkpoint procedures for {backend OR frontend} workspaces.

3. spec/{feature-folder}/{WP-ID}.md
   — The Work Package. This is your specification.

{IF resuming — include this line:}
4. .claude/skills/implement-v2/references/resume-protocol.md
   — Resume procedure. Your last checkpoint was: "{last_checkpoint value}"

## Context

- Workspace path: workspaces/{workspace-id}
- Workspace type: {backend OR frontend}
- Status file: spec/{feature-folder}/status.yaml
- Your status entry: phase_4.{WP-ID}
- Mode: {Fresh start OR Resume from: {last_checkpoint}}

## Constraints

- Do NOT read other WP files.
- Do NOT read the FS or TS directly — use the verbatim copies in your WP.
- Do NOT modify files in contracts/, spec/, or reference/ (except status.yaml checkpoints).
- Write ONLY your own phase_4.{WP-ID} entry in status.yaml.
```

**What the executor does NOT receive:**
- Other WP files (it only sees its own WP)
- This orchestrator SKILL.md
- `registry/routes.yaml` (workspace path already resolved by orchestrator)
- `checkpoint-taxonomy.md` (the be-guide/fe-guide cover checkpoint procedures)
- Any spec-phase reference files

**Do not present the execution plan to the human.** Proceed directly to delegation.

---

### Step 4: Monitor and Coordinate

After delegating:

1. Wait for sub-agent completion
2. Read `status.yaml` after each WP completes to check for:
   - Blockers reported by the executor
   - Newly unblocked downstream WPs ready for the next wave
3. Launch next wave of executor sub-agents if dependencies are now met
4. If an executor reports a blocker:
   - Relay the blocker and recommendation to the human
   - Hold dependent downstream WPs
   - Continue unaffected WPs

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
