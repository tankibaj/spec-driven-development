---
name: spec
description: Writes a Feature Spec for SDD with impact analysis and contract review baked in. Use after the PRD is approved, when ready to define acceptance criteria.
argument-hint: "[feature-id] [slug]"
allowed-tools: Read Glob
disable-model-invocation: true
---

# SDD Feature Spec

## Overview

Generate an Impact Analysis (IA) and Feature Spec (FS) in a single pass from an approved PRD. The only interactive step is resolving open questions or ambiguities in the PRD — everything else is autonomous.

The FS is the contract between the Product Owner and the implementation agent. Every AC must be testable, traceable to the concept, and aligned with existing contracts.

## When to Use

- After a PRD has been approved
- When ready to define acceptance criteria for a feature
- When the UX/UI design (if required by PRD) is complete

**Requires:** An approved PRD in the feature folder (`spec/XXX-slug/`).

**When NOT to use:** Before concept approval (use `/prd`), during implementation, for PRD writing.

## Guiding Principles

These apply throughout generation — not just one step:

- Never invent ACs beyond what the PRD describes. If you think one is missing, include it in the FS clearly labeled `[AGENT SUGGESTION]` so the human can accept or reject during review.
- Never silently assume a contract supports what you're writing. Check the auto-generated specs in workspace `docs/` dirs or flag as "to be created."
- For every happy-path AC, generate at least one error/edge-case AC where the happy path can fail.
- If the PRD is vague on a point, reframe it as testable conditions and include both the interpretation and the `Testable:` line. The human validates during review.
- Every affected service from the impact analysis must have at least one AC. If a service is affected but has no AC, either the analysis is wrong or an AC is missing.
- If breaking contract changes are needed, flag them immediately in the IA — they require an ADR and consumer approval before implementation.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The PRD covers this, I don't need to dig deeper" | The PRD captures intent, not precision. ACs require details the PRD deliberately left vague. |
| "This AC is obvious, it doesn't need a Testable: line" | If you can't write the Testable: line, the AC is too vague. Every AC, no exceptions. |
| "I'll assume the contract supports this" | Check the auto-generated specs in workspace `docs/` dirs. If the endpoint doesn't exist, flag it. Silent assumption = spec bug found in Phase 4. |
| "Edge cases can be handled during implementation" | Edge cases not in the FS won't be in the TS, won't be in the WP, won't be tested. Capture them now. |
| "This service is probably not affected" | Check `routes.yaml`. "Probably" is not analysis. |
| "The human will catch this in review" | You are the first line of defense. The verification checklist exists because review gates are not infallible. |

## Red Flags

Stop and reassess if you catch yourself doing any of these:

- Writing ACs before completing impact analysis
- Adding ACs the PRD never described without labeling them `[AGENT SUGGESTION]`
- No error/edge-case AC for a happy-path AC that can fail
- Using implementation language ("use a cron job", "add a column") instead of behavioral language
- Assuming a contract exists without checking workspace `docs/` dirs
- Leaving a service from the impact analysis without a corresponding AC

---

## The Workflow

```
LOAD PRD ──→ RESOLVE? ──→ GENERATE IA + FS ──→ WRITE FILES ──→ HUMAN REVIEWS
   │            │               │                    │               │
   ▼            ▼               ▼                    ▼               ▼
 Read the    Only if PRD     Single pass:         Save as         Human approves
 concept     has open Qs     IA + full FS         draft           or requests
             or ambiguity    autonomously                         changes
```

Two steps interactive (at most). One step autonomous. One human review gate.

---

### Step 1: Setup and PRD Assessment

If arguments provided: `$0` = feature-ID, `$1` = slug. Otherwise:
1. List existing folders in `spec/` and offer them if any match the context
2. Ask the human: "Which feature folder? e.g., 002-payment-flow"

**Output location:** `spec/{$0}-{$1}/` — all artifacts (FS, IA) are saved to this folder.

Steps:
1. Navigate to the feature folder (create it if it doesn't exist)
2. Verify PRD exists and status is approved in `status.yaml`
3. Load domain context:
   - `docs/reference/glossary.md`, `personas.md`, `roles.md`
4. Load all skill reference files:
   - `${CLAUDE_SKILL_DIR}/template.md` — FS structure and Testable: line formats
   - `${CLAUDE_SKILL_DIR}/assets/impact-analysis-template.md` — IA document format
   - `${CLAUDE_SKILL_DIR}/references/contract-review-checklist.md` — contract review rules
5. Load contracts and registry:
   - `routes.yaml` — all workspaces (includes `contracts` field with paths to each workspace's auto-generated specs)
   - For each workspace in `routes.yaml`, read the auto-generated specs listed in `contracts` (read-only, never modify):
     - Backend: `workspaces/{service}/docs/api/openapi.json` — OpenAPI spec
     - Backend: `workspaces/{service}/docs/schema/entities.md` — entity schemas
     - Frontend: `workspaces/{app}/docs/routes.md` — route manifest
     - Frontend: `workspaces/{app}/docs/consumed-endpoints.md` — consumed backend endpoints
6. Check the feature folder for existing FS and IA IDs to avoid collisions
7. Read the PRD in full. Assess:
   - Are there unresolved open questions (unchecked `[ ]` items)?
   - Are any scope items ambiguous or vague?
   - Are there assumptions that need human confirmation?

If no approved PRD exists:

```
BLOCKED: No approved PRD found in this feature folder.
→ Use /prd to create one first.
```

**Decision point:**

```
IF PRD has unresolved open questions OR critical ambiguities:
  → Go to Step 2 (Resolve)

IF PRD is clear and all questions are resolved:
  → Skip Step 2, go directly to Step 3 (Generate)
```

---

### Step 2: Resolve Open Questions (interactive — only if needed)

This step runs ONLY when the PRD has unresolved open questions or ambiguities that would fundamentally change the ACs.

For each open question or ambiguity:
1. Present options (minimum 2) with your recommendation and reasoning
2. Wait for the human to choose
3. Record the answer in the PRD (mark question `[x]`, add answer line below)

After all questions are resolved, proceed to Step 3. Do not ask additional questions beyond what the PRD flagged — save any agent-identified concerns for the FS Open Questions section.

---

### Step 3: Generate IA + FS (autonomous — single pass)

Generate both artifacts in one pass. Do not stop to ask the human questions. If you encounter uncertainty, document it in the FS Assumptions or Open Questions section with your recommended resolution.

#### 3a: Impact Analysis

Answer these five questions:

1. **Which services are affected?** — Read `routes.yaml`, check each workspace
2. **Which contracts need to change?** — Read auto-generated specs from workspace `docs/` dirs (paths in `routes.yaml`)
3. **Which consumers of those contracts exist?** — Identify downstream dependencies
4. **What is the blast radius if this breaks?** — Assess scope of impact
5. **Can this be done without changing contracts?** — Prefer additive changes over breaking ones

When anything is unclear, document it as an explicit assumption in the IA — do not stop to ask.

If breaking changes are identified:
- Flag for ADR proposal in `docs/architecture/`
- Note in FS "Related Contracts" section

#### 3b: Feature Spec — All Sections

Generate the complete FS using the template structure (loaded in Step 1):

**Goal** — Pull from PRD's WHAT and WHO. Name the primary persona from `docs/reference/personas.md`.

**Background** — Pull from PRD's WHY. Add business context.

**Impact Analysis Summary** — Summarize IA findings: services affected, contracts to change, blast radius, breaking changes.

**Assumptions** — Seed from:
- PRD assumptions (carried forward)
- Impact analysis findings (services assumed available, data assumed to exist)
- Resolved open question decisions
- Any agent assumptions made during generation — clearly labeled

**User Flow** — For multi-step features: map the end-to-end sequence. Number each step and note which AC(s) it covers. For single-action features: skip this section.

**Acceptance Criteria** — For each capability in the PRD scope:

1. Write each AC with:
   - `AC-XXX` ID and descriptive title
   - Behavioral description (what happens, not how to implement)
   - `Testable:` line using standardized formats from the FS template

2. For every happy-path AC, generate error/edge-case ACs where the happy path can fail.

3. Cross-check each AC against:
   - Impact analysis — every affected service should have ≥1 AC
   - workspace `docs/api/openapi.json` — do referenced endpoints exist?
   - workspace `docs/schema/entities.md` — do referenced entities exist?
   - workspace `docs/routes.md` — do referenced routes exist? (frontend only)
   - workspace `docs/consumed-endpoints.md` — do consumed endpoints match? (frontend only)
   - `docs/reference/glossary.md` — is domain terminology correct?

4. Flag missing contracts as "(to be created — see Contract Review)".

5. If any ACs go beyond what the PRD explicitly described, label them `[AGENT SUGGESTION]`.

**AC quality criteria — every AC must be:**
- **TESTABLE:** has a `Testable:` line with concrete verification
- **TRACEABLE:** maps to a capability in the PRD
- **CONTRACT-ALIGNED:** references existing or flagged-for-creation contracts
- **BEHAVIORAL:** describes what happens, not implementation details

**Contract Review** — Summarize contract needs using the table format and preference rules from the contract review checklist (loaded in Step 1). Review each changed contract against the checklist criteria.

**Open Questions** — Capture items the agent could not resolve with confidence. For each:
- State the question
- Provide 2-3 options with the agent's recommendation
- Tag with severity: `[BLOCKS APPROVAL]` if it changes ACs, or `[MINOR]` if it's a refinement
- FS stays at `draft` while `[BLOCKS APPROVAL]` questions remain

**Non-Functional Requirements** — Include if the PRD implies performance, security, or scalability requirements. Otherwise omit.

**Out of Scope** — Pull from PRD's "Explicitly Out of Scope" section. Must be non-empty.

**Related Contracts** — Link existing contracts + flag ones to be created. Include contract change summary from the contract review.

#### 3c: Self-Verification

Run the verification checklist (below) before proceeding. Fix any failures silently — do not present a known-incomplete artifact.

---

### Step 4: Write Files and Present

1. Determine the next available `FS-XXX` and `IA-XXX` IDs from the feature folder scan
2. Write `IA-XXX.md` to the feature folder using the IA document template
3. Write `FS-XXX.md` to the feature folder using the FS template structure
4. Update `status.yaml`:

```yaml
current_phase: 1
artifacts:
  PRD-XXX: { status: approved }
  IA-XXX: { status: draft, date: YYYY-MM-DD }
  FS-XXX: { status: draft, date: YYYY-MM-DD }
```

5. Present a summary to the human:
   - Impact analysis highlights (services affected, contract changes, blast radius)
   - AC count and coverage summary
   - Open questions (if any) with recommended answers
   - Assumptions that need validation
   - "IA and FS written as drafts. Review both files and update `status.yaml` when approved."

6. **STOP.** The human reviews the files directly, resolves open questions, and updates `status.yaml` to `approved` when satisfied.

---

## Verification Checklist

Run before writing files. Every item must pass.

- [ ] Approved PRD was loaded and referenced
- [ ] Impact analysis completed — all affected services identified
- [ ] Header block complete (folder, status, author, date, depends on, blocks)
- [ ] Goal names a persona from `docs/reference/personas.md`
- [ ] Every AC has a unique `AC-XXX` ID and descriptive title
- [ ] Every AC has a `Testable:` line with concrete verification
- [ ] `Testable:` lines use standardized format (API / UI / System)
- [ ] Error/edge-case ACs exist for every happy-path AC that can fail
- [ ] Every affected service (from impact analysis) has ≥1 AC
- [ ] ACs cross-checked against workspace contract docs (`openapi.json`, `entities.md`, `routes.md`, `consumed-endpoints.md`)
- [ ] Missing contracts flagged as "to be created"
- [ ] Breaking changes flagged with ADR recommendation
- [ ] Domain terminology matches `docs/reference/glossary.md`
- [ ] Out of Scope section is present and non-empty
- [ ] Open questions captured with options and recommendations (blocks approval if `[BLOCKS APPROVAL]`)
- [ ] Contract review summary included in Related Contracts
- [ ] ACs beyond PRD scope labeled `[AGENT SUGGESTION]`
