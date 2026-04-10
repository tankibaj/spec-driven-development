---
name: sdd-feature-spec
description: Writes a Feature Spec for SDD with impact analysis and contract review baked in. Use after the Feature Concept (PDR) is approved, when ready to define acceptance criteria.
argument-hint: "[story-id] [slug]"
allowed-tools: Read Glob
disable-model-invocation: true
---

# SDD Feature Spec

## Overview

Write a Feature Spec (FS) informed by an approved Feature Concept. Impact analysis and contract review are part of this workflow — not separate steps.

The FS is the contract between the Product Owner and the implementation agent. Every AC must be testable, traceable to the concept, and aligned with existing contracts.

## When to Use

- After a Feature Concept (PDR) has been approved
- When ready to define acceptance criteria for a feature
- When the UX/UI design (if required by PDR) is complete

**Requires:** An approved PDR in the feature folder (`plan/spec/Story-XXXX/`).

**When NOT to use:** Before concept approval (use `/sdd-feature-concept`), during implementation, for PDR writing.

## Standing Instructions

These apply at every step of the workflow — not just one:

- Never invent ACs beyond what the human described. If you think one is missing, propose it separately and label it as your suggestion.
- Never silently assume a contract supports what you're writing. Check `contracts/` or flag as "to be created."
- After every happy-path AC, ask: "Can this fail? What should happen when it does?"
- If the human gives vague input, reframe it as testable conditions and confirm before writing ACs.
- If anything is unclear, ask the human. If they can't answer, document it as an explicit assumption — never silently fill the gap.
- Every affected service from the impact analysis must have at least one AC. If a service is affected but has no AC, either the analysis is wrong or an AC is missing.
- If breaking contract changes are needed, flag them immediately — they require an ADR and consumer approval before implementation.
- After completing each step, state: "Step N done. Moving to Step N+1: [name]." This keeps progress visible across long sessions.
- If you cannot resolve a question or blocker at any step, add it as an Open Question with a `[BLOCKED: Step N]` tag and continue to the next step. The FS stays at Draft status while open questions remain. Do not stop the entire workflow for one unresolved item.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The PDR covers this, I don't need to ask" | The PDR captures intent, not precision. ACs require details the PDR deliberately left vague. |
| "This AC is obvious, it doesn't need a Testable: line" | If you can't write the Testable: line, the AC is too vague. Every AC, no exceptions. |
| "I'll assume the contract supports this" | Check `contracts/`. If the endpoint doesn't exist, flag it. Silent assumption = spec bug found in Phase 4. |
| "Edge cases can be handled during implementation" | Edge cases not in the FS won't be in the TS, won't be in the WP, won't be tested. Capture them now. |
| "This service is probably not affected" | Check `registry/routes.yaml`. "Probably" is not analysis. |
| "The human will catch this in review" | You are the first line of defense. The verification checklist exists because review gates are not infallible. |

## Red Flags

Stop and reassess if you catch yourself doing any of these:

- Writing ACs before completing impact analysis
- Adding ACs the human never described without labeling them as suggestions
- Skipping the "Can this fail?" question after a happy-path AC
- Using implementation language ("use a cron job", "add a column") instead of behavioral language
- Proceeding past a review gate without explicit human confirmation
- Assuming a contract exists without checking `contracts/`
- Leaving a service from the impact analysis without a corresponding AC

## The Workflow

```
LOAD PDR ──→ IMPACT ──→ DRAFT ACs ──→ CONTRACT ──→ ASSEMBLE ──→ PRESENT
   │            │           │            │            │            │
   ▼            ▼           ▼            ▼            ▼            ▼
 Read the    Blast       Write each   Review what   Build the   Human
 concept     radius      AC with      contracts     full FS     reviews
             analysis    Testable:    need changing
```

Each phase has a human review gate. Do not advance until confirmed.

### Step 1: Setup

If arguments provided: `$0` = Story-ID, `$1` = slug. Otherwise:
1. List existing folders in `plan/spec/` and offer them if any match the context
2. Ask the human: "Which feature folder? e.g., Story-0002-payment-flow"

**Output location:** `plan/spec/Story-{$0}-{$1}/` — all artifacts (FS, IA) are saved to this folder.

Steps:
1. Navigate to the feature folder (create it if it doesn't exist)
2. Verify PDR exists and status is approved in `status.yaml`
3. Load domain context:
   - `plan/reference/glossary.md`, `personas.md`, `roles.md`
4. Load all skill reference files (used in later steps):
   - `${CLAUDE_SKILL_DIR}/template.md` — FS structure and Testable: line formats (Steps 6, 9)
   - `${CLAUDE_SKILL_DIR}/assets/impact-analysis-template.md` — IA presentation + document format (Step 2)
   - `${CLAUDE_SKILL_DIR}/references/contract-review-checklist.md` — contract review table + rules (Step 7)
5. Check the feature folder for existing FS and IA IDs to avoid collisions

If no approved PDR exists:

```
BLOCKED: No approved Feature Concept found in this feature folder.
→ Use /sdd-feature-concept to create one first.
```

### Step 2: Impact Analysis

This is a critical step for existing systems. Before writing any ACs, assess what this feature touches.

Answer these five questions:

1. **Which services are affected?** — Read `registry/routes.yaml`, list all workspaces
2. **Which contracts need to change?** — Scan `contracts/api/` and `contracts/data-schema/`
3. **Which consumers of those contracts exist?** — Identify downstream dependencies
4. **What is the blast radius if this breaks?** — Assess scope of impact
5. **Can this be done without changing contracts?** — Prefer additive changes over breaking ones. If the feature can be delivered without modifying existing contracts, that is always the safer path.

**When anything is unclear, ask the human.** Do not assume service boundaries, contract ownership, or blast radius. If the human cannot answer, document the item as an explicit assumption in the impact analysis — never silently fill in the gap.

Present findings to the human using the IA presentation format (loaded in Step 1). Must include: services affected, contracts to change, consumers affected, blast radius, recommended approach, and any assumptions made.

If breaking changes are identified:
- Flag for ADR proposal in `contracts/architecture/`
- Note in FS "Related Contracts" section

**Present impact analysis to human. Confirm before writing ACs.**

### Step 3: Goal and Background

Draft the **Goal** section — pull from PDR's WHAT and WHO. Name the primary persona from `plan/reference/personas.md`.

Draft the **Background** section — pull from PDR's WHY. Add data or business context.

Ask the human: "What is the business context? Why now?" (The human may have more context than the PDR captured.)

### Step 4: Assumptions

Ask the human: "What are we taking for granted? What, if wrong, would change this spec?"

Seed from:
- PDR assumptions
- Impact analysis findings (services assumed available, data assumed to exist)

### Step 5: User Flow

For multi-step features: map the end-to-end sequence. Number each step and note which future AC(s) it covers.

For single-action features: skip this section.

### Step 6: Acceptance Criteria

This is the core of the spec. For each capability in the PDR scope:

**If the PDR or human gives vague input** (e.g., "make checkout faster"), reframe it as testable conditions before writing ACs:

```
VAGUE: "Make checkout faster"

REFRAMED:
- AC-XXX: Checkout API responds within 2s (p95)
- AC-XXX: No layout shift during checkout load (CLS < 0.1)
→ Are these the right targets?
```

Confirm the reframing with the human before proceeding.

1. **Propose ACs one at a time.** Each AC must have:
   - `AC-XXX` ID and descriptive title
   - Behavioral description (what happens, not how to implement)
   - `Testable:` line using standardized formats from the FS template (loaded in Step 1)

2. **After each happy-path AC, propose error/edge-case ACs** where the happy path can fail. Ask the human: "Can this fail? What should happen when it does?"

3. **Cross-check each AC against:**
   - Impact analysis — every affected service should have ≥1 AC
   - `contracts/api/` — do referenced endpoints exist?
   - `contracts/data-schema/` — do referenced entities exist?
   - `plan/reference/glossary.md` — is domain terminology correct?

4. **Flag missing contracts.** If an AC references an API or entity that doesn't exist yet, note it as "(to be created — see Contract Review below)".

5. **Flag new domain terms.** If the feature introduces terms not in `glossary.md`, note them for glossary addition.

**AC quality criteria — every AC must be:**
- **TESTABLE:** has a `Testable:` line with concrete verification
- **TRACEABLE:** maps to a capability in the PDR
- **CONTRACT-ALIGNED:** references existing or flagged-for-creation contracts
- **BEHAVIORAL:** describes what happens, not implementation details

### Step 7: Contract Review

Based on impact analysis + ACs written, summarize contract needs using the table format and preference rules from the contract review checklist (loaded in Step 1). Review each changed contract against the checklist criteria.

**Present to human.** Contract changes will be executed in Phase 3 (WP generation) per CLAUDE.md.

### Step 8: Open Questions and Remaining Sections

- **Open Questions** — capture unresolved items as checkboxes. FS stays at Draft while questions remain.
- **Non-Functional Requirements** — ask: "Any performance, security, or scalability requirements?" Optional.
- **Out of Scope** — ask: "What should we explicitly exclude?" Must be non-empty.
- **Related Contracts** — link existing contracts + flag ones to be created. Include contract change summary from Step 7.

### Step 9: Assemble and Present

1. Determine the next available `FS-XXX` ID by listing existing files in the feature folder
2. Assemble the full draft FS using the template structure (loaded in Step 1)
3. Include impact analysis summary in the Related Contracts section
4. Run the verification checklist (below)
5. Present the complete draft to the human
6. **STOP.** Wait for human review and sign-off.

### Step 10: Finalize

After the human signs off on the draft content:

1. Determine the next available `FS-XXX` ID (from Step 1 scan)
2. Determine the next available `IA-XXX` ID by listing existing files in the feature folder
3. Write `FS-XXX.md` to the feature folder
4. Write `IA-XXX.md` to the same feature folder
5. Update `status.yaml`:

```yaml
artifacts:
  PDR: { status: approved }
  IA-XXX: { status: approved, date: YYYY-MM-DD }
  FS-XXX: { status: draft, date: YYYY-MM-DD }
```

6. Tell the user: "FS saved. Ready to proceed to Phase 2 (Test Spec generation) when you approve."

## Verification Checklist

Run before presenting the draft. Every item must pass.

- [ ] Approved PDR was loaded and referenced
- [ ] Impact analysis completed — all affected services identified
- [ ] Header block complete (folder, status, author, date, depends on, blocks)
- [ ] Goal names a persona from `plan/reference/personas.md`
- [ ] Every AC has a unique `AC-XXX` ID and descriptive title
- [ ] Every AC has a `Testable:` line with concrete verification
- [ ] `Testable:` lines use standardized format (API / UI / System)
- [ ] Error/edge-case ACs exist for every happy-path AC that can fail
- [ ] Every affected service (from impact analysis) has ≥1 AC
- [ ] ACs cross-checked against `contracts/api/` and `contracts/data-schema/`
- [ ] Missing contracts flagged as "to be created"
- [ ] Breaking changes flagged with ADR recommendation
- [ ] Domain terminology matches `plan/reference/glossary.md`
- [ ] Out of Scope section is present and non-empty
- [ ] Open questions captured (blocks approval if unresolved)
- [ ] Contract review summary included in Related Contracts
- [ ] No acceptance criteria invented beyond what the human described — flag suggestions separately
