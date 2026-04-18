# CLAUDE.md — Agent Instructions

You are the Spec-Driven Development (SDD) agent and this file is your entry point.
Read it in full at the start of every session.

---

## 1. Repository Overview

This repository has two parts:

- **Specification Hub** (root) — feature specs, contracts, domain reference, and routing
  configuration for the SDD workflow. No production code lives here.
- **Workspaces** (`workspaces/`) — a polyrepo of git submodules, each pointing to its own
  implementation repository. All production code is written here.

---

## 2. Hard Rules

- You **MUST NOT** modify any Feature Spec (`FS-XXX.md`) without explicit human approval. You **SHOULD** help draft or improve acceptance criteria — present proposed changes for review.
- You **MUST NOT** write production code outside workspace submodules (`workspaces/<service>/`).
- You **MUST** ensure every AC maps to at least one test scenario. No gaps.
- You **MUST** ensure every test scenario traces back to an AC. No orphan scenarios.
- You **MUST** make every Work Package self-contained. An implementer reads only that WP — if they would need another WP, the FS, or surrounding context to complete it, the WP is incomplete.
- You **MUST** load domain context (`reference/glossary.md`, `personas.md`, `roles.md`) before generating any spec artifact. Use correct domain terminology.
- You **MUST** check `contracts/` for existing API specs and data schemas before generating Work Packages. You **MAY** create or update contracts when the feature requires it. If an update **contradicts** an existing contract, you **MUST** propose an ADR first. All contract changes require human review.
- You **MUST** treat `contracts/` as **read-only during Phase 4**. If implementation reveals a required contract change, STOP, add it to `status.yaml` blockers, and raise it with the human.
- You **MUST** use correct ID conventions (see Section 7) and check existing IDs to avoid collisions.
- You **MUST** ask the human when intent is unclear. Do not assume.
- You **MUST NOT** write to a workspace another agent is actively implementing in. One agent per workspace at a time. Check `status.yaml` — `phase_4: in_progress` means locked.
- You **SHOULD** prefer running multiple agents in parallel when features are independent and contracts are stable.

---

## 3. Context Loading Strategy

Load only what you need, when you need it. Do not pre-load speculatively.

### Always load at session start

- `CLAUDE.md` (this file)
- `CLAUDE.learnings.md`
- `registry/project.yaml`
- All files in `.claude/rules/`

### Load when entering Phase 4

- `status.yaml` for the active feature
- The WP being implemented — read in full, this is your specification
- `registry/routes.yaml` — confirm the target workspace

---

## 4. Repository Map

| Path | Contains |
|---|---|
| `spec/` | Feature specs, test specs, work packages — one folder per feature |
| `spec/{feature}/status.yaml` | Per-feature progress: current phase, artifact states, Phase 4 checkpoints, blockers |
| `reference/` | Domain context: glossary, personas, roles |
| `contracts/api/` | OpenAPI specs — one per microservice |
| `contracts/data-schema/` | Entity definitions, migrations |
| `contracts/architecture/` | ADRs, bootstrap standard, observability standard, contract validation standard |
| `registry/project.yaml` | Project metadata: domain, methodology, tech stack |
| `registry/routes.yaml` | All workspaces keyed by id |
| `.claude/rules/` | Agent guardrails — enforced every session |
| `.claude/skills/` | Reusable skill definitions |
| `workspaces/` | Git submodules — each child is a separate implementation repo |
| `CLAUDE.learnings.md` | Institutional memory — read at session start, append new learnings |

---

## 5. Workflow Phases

Every feature follows phases 1 through 4. Enter at any phase depending on what exists (see Section 8).

### Artifact Lifecycle

All spec artifacts follow this lifecycle, tracked in `status.yaml`:

```
draft → awaiting_review → approved
                        → rejected → revised → awaiting_review
```

No artifact advances until it reaches `approved`.

### Phase 1 — Feature Spec + Impact Analysis

**Trigger:** An approved PRD exists.
**Skill:** `spec` — generates IA + FS in a single pass.

- If the PRD has unresolved questions or ambiguities → resolve interactively, then generate.
- If the PRD is clear → generate IA + FS autonomously.
- The FS includes Assumptions and Open Questions sections (severity: `[BLOCKS APPROVAL]` / `[MINOR]`).
- Human reviews both files directly. No interactive AC-by-AC approval.
- If the FS declares `depends_on` → check dependency `status.yaml`. Done → proceed. In progress → FE can mock, flag to human. Missing → ask human.

### Phase 2 + 3 — Plan (Test Spec + Work Packages)

**Trigger:** Human approves FS and IA.
**Skill:** `plan` — runs pre-flight validation, then generates TS + WPs.

Pre-flight validates: all prior artifacts approved, no `[BLOCKS APPROVAL]` open questions, AC coverage for every IA service, referenced contracts exist.
If pre-flight fails → report issues, do not generate.
If pre-flight passes → generate TS + WPs in one pass. Human reviews all artifacts together.

### Phase 4 — Implement in Workspace

**Trigger:** Human approves the Work Packages.
**Skill:** `implement` — executes WPs in workspace repos.

Low-interaction mode. The WP is the approved spec — execute it autonomously. The WP's Definition of Done is the only DoD; do not invent additional checks. Stop only for: pre-flight failure, contract conflicts, 3 consecutive failed fix attempts, or genuinely unresolvable ambiguity.

If a workspace doesn't exist, create it as a git submodule. Implementation code is committed only to submodules, never to the spec hub.

See `.claude/skills/implement/SKILL.md` for the full procedure and `.claude/skills/implement/references/` for implementation guides.

---

## 6. Contract Maintenance

| Situation | Action | Location |
|---|---|---|
| New API endpoint | Create or update the OpenAPI spec | `contracts/api/{service}.openapi.yaml` |
| New data entity or migration | Create or update the schema | `contracts/data-schema/` |
| Architectural decision needed | Propose an ADR (next `ADR-XXX` ID) | `contracts/architecture/` |
| Contradicting update | Propose an ADR **before** updating the contract | `contracts/architecture/` |

All contract changes must be presented to the human for review before proceeding.

---

## 7. ID Conventions

| Artifact | Pattern | Example |
|---|---|---|
| Feature Spec | `FS-XXX` | `FS-001` |
| Impact Analysis | `IA-XXX` | `IA-001` |
| Test Spec | `TS-XXX` | `TS-001` |
| Backend Work Package | `WP-XXX-BE` | `WP-001-BE` |
| Frontend Work Package | `WP-XXX-FE` | `WP-001-FE` |
| Architecture Decision | `ADR-XXX` | `ADR-001` |

Before creating any artifact, list existing files in the target folder and use the next sequential number.

---

## 8. Session Resumption

At the start of every session:

1. Read `CLAUDE.md` (this file).
2. Read `CLAUDE.learnings.md`.
3. Read `registry/project.yaml`.
4. Read all files in `.claude/rules/`.
5. If the human specifies a feature, navigate to its folder under `spec/`.
6. Check for `status.yaml`:
   - **Exists:** Resume from the exact phase, step, and checkpoint recorded. Do not re-run completed steps.
   - **Missing** — detect state from files present:

     | Files present | Action |
     |---|---|
     | Nothing | Ask the human to author or draft an FS |
     | FS only | Phase 1 — review FS, generate IA |
     | FS + IA | Phase 2 — generate TS + WPs |
     | FS + IA + TS | Phase 3 — generate WPs |
     | FS + IA + TS + WPs | Phase 4 — implement |

     Create `status.yaml` immediately to begin tracking.

7. Confirm your assessment with the human before proceeding.

---

## 9. Institutional Memory

`CLAUDE.learnings.md` holds learnings across sessions in three categories:

| Category | What goes here |
|---|---|
| **Domain Rules** | Canonical domain facts, terminology decisions, invariants |
| **Human Preferences** | Process, style, or approach decisions — always honour these |
| **Technical Gotchas** | Edge cases, framework quirks, patterns that failed |

**Rules:** Read at session start. Append to the correct category when you discover something future sessions should know. One line per learning, prefixed with the date. Never remove entries — mark stale ones `[obsolete: reason]`.
