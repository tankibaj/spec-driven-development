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

If arguments provided: `$0` = Story-ID, `$1` = slug. Otherwise ask the human.

1. Navigate to `plan/spec/Story-XXXX-slug/`
2. Verify PDR exists and status is approved in `status.yaml`
3. Load domain context:
   - `plan/reference/glossary.md`, `personas.md`, `roles.md`
4. Load template: `${CLAUDE_SKILL_DIR}/template.md`
5. Check `plan/spec/` for existing FS IDs to avoid collisions

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

Output format:

```
IMPACT ANALYSIS: [Feature Name]

SERVICES AFFECTED:
  - [service-id]: [what changes] — [additive / breaking / none]

CONTRACTS TO CHANGE:
  - [contract file]: [description of change]

CONSUMERS AFFECTED:
  - [service/app that depends on changed contracts]

BLAST RADIUS: [High / Medium / Low] — [reasoning]
BREAKING CHANGES: [Yes / No]
RECOMMENDED APPROACH: [no contract changes / additive / new version / breaking with ADR]

ASSUMPTIONS (if human could not confirm):
  - [item assumed and why]

→ Confirm before I write ACs.
```

Use template: `${CLAUDE_SKILL_DIR}/assets/impact-analysis-template.md`

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

1. **Propose ACs one at a time.** Each AC must have:
   - `AC-XXX` ID and descriptive title
   - Behavioral description (what happens, not how to implement)
   - `Testable:` line using standardized formats from `${CLAUDE_SKILL_DIR}/template.md`

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

Based on impact analysis + ACs written, summarize contract needs:

```
CONTRACT CHANGES NEEDED:
| Contract | Change | Type | Status |
|----------|--------|------|--------|
| [file]   | [desc] | additive/breaking | exists / to be created |
```

For detailed review criteria: `${CLAUDE_SKILL_DIR}/references/contract-review-checklist.md`

Rules:
- Prefer additive changes over breaking
- Prefer v1 → v2 versioning over modifying v1
- Breaking changes require an ADR proposal
- Consumer teams must approve before implementation begins

Present to human. Contract changes will be executed in Phase 3 (WP generation) per CLAUDE.md.

### Step 8: Open Questions and Remaining Sections

- **Open Questions** — capture unresolved items as checkboxes. FS stays at Draft while questions remain.
- **Non-Functional Requirements** — ask: "Any performance, security, or scalability requirements?" Optional.
- **Out of Scope** — ask: "What should we explicitly exclude?" Must be non-empty.
- **Related Contracts** — link existing contracts + flag ones to be created. Include contract change summary from Step 7.

### Step 9: Assemble and Present

1. Determine the next available `FS-XXX` ID by listing existing files in the feature folder
2. Assemble the full draft FS using the template structure from `${CLAUDE_SKILL_DIR}/template.md`
3. Include impact analysis summary in the Related Contracts section
4. Run the verification checklist (below)
5. Present the complete draft to the human
6. **STOP.** Wait for human review and sign-off.

### Step 10: Finalize

After the human signs off on the draft content:

1. Write `FS-XXX.md` to the feature folder
2. Write impact analysis to the feature folder as `IA-XXX.md`
3. Update `status.yaml`:

```yaml
artifacts:
  PDR: { status: approved }
  IA-XXX: { status: approved, date: YYYY-MM-DD }
  FS-XXX: { status: draft, date: YYYY-MM-DD }
```

4. Tell the user: "FS saved. Ready to proceed to Phase 2 (Test Spec generation) when you approve."

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
