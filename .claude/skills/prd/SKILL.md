---
name: prd
description: Writes a Feature Concept (PRD) for Spec-Driven Development. Defines what to build, why, and for whom before any spec work begins. Use when starting a new feature.
argument-hint: "[feature name or idea]"
allowed-tools: Read Glob
disable-model-invocation: true
---

# SDD Feature Concept

## Overview

The Feature Concept (PRD) is the business case that precedes all specification work. It defines WHAT we're building and WHY — before anyone thinks about HOW.

Every artifact in the SDD chain (FS, TS, WPs) traces back to this document. If a Work Package can't trace to a business reason in the concept, it shouldn't exist.

If `$ARGUMENTS` is empty, ask the user what feature to work on. Do not infer from conversation context without confirmation.

## When to Use

- Starting a new feature or product capability
- Evaluating whether an idea is worth investing in
- Defining the business case before specification work
- Deciding whether UX/UI design is needed

**When NOT to use:** Writing the Feature Spec (use `/spec`), implementation work, TS/WP generation, trivial changes that don't need a business case.

## The Workflow

```
CONCEPT ──→ CLARIFY ──→ SCOPE ──→ PRESENT ──→ HAND OFF
   │           │          │          │            │
   ▼           ▼          ▼          ▼            ▼
 What &      Surface    One or    Human        Ready for
 why         assumptions many?    reviews      /spec
```

Each phase has a human review gate. Do not advance until confirmed.

### Step 1: Concept

Define the feature from a business perspective. Answer six questions:

```
FEATURE CONCEPT: [Name]

WHAT:       [capability from the user's perspective — not implementation]
WHY:        [business justification — problem, opportunity, or risk]
WHO:        [persona from plan/reference/personas.md]
SUCCESS:    [measurable outcome — business metric, not test case]
UX/UI:      [needed / not needed — with reasoning]
DEPENDS ON: [existing features or "none"]
BLOCKS:     [downstream features or "none"]

→ Confirm the direction before I continue.
```

Use full template from `${CLAUDE_SKILL_DIR}/assets/feature-concept-template.md` for the written artifact.

**STOP.** Wait for human to confirm the direction before Step 2.

### Step 2: Clarify

Gather project context to ground the concept:

- Read `plan/reference/glossary.md`, `personas.md`, `roles.md`
- Read `registry/routes.yaml` for the service landscape
- Scan `contracts/` for existing APIs and data schemas
- Check `plan/spec/` for related features (depends_on, blocks)

Surface assumptions explicitly:

```
ASSUMPTIONS:
1. [assumption about scope, technology, or behavior]
2. [assumption about users or environment]
3. [assumption about constraints or dependencies]
→ Correct me now or I'll proceed with these.
```

Ask the human about anything ambiguous. Don't proceed until requirements are concrete.

### Step 3: Scope Decision

Assess whether this concept is one feature or multiple:

```
SCOPE CHECK:
- Does this concept have >3 distinct capabilities?  → consider splitting
- Does it affect >3 services?                       → consider splitting
- Would the FS likely have >10 ACs?                 → consider splitting
```

If splitting is recommended, propose separate concepts — each with its own Story-XXXX folder and its own PRD.

Present the recommendation. Human decides.

### Step 4: Present

Show the complete Feature Concept:
- Concept (WHAT / WHY / WHO / SUCCESS / UX-UI)
- Assumptions (confirmed or pending)
- Scope decision (one feature or split)
- Dependencies (depends_on, blocks)

Wait for human approval.

### Step 5: Hand Off

After approval:

1. Ask the user for the Story-ID and slug (e.g., `Story-0003-user-notifications`)
2. Check `plan/spec/` for existing folders to avoid ID collisions
3. Create the feature folder under `plan/spec/`
4. Write the Feature Concept file to the folder
5. Create `status.yaml`:

```yaml
feature: Story-XXXX-slug
current_phase: 0
artifacts:
  PRD: { status: approved, date: YYYY-MM-DD }
phase_4: {}
blockers: []
notes: ~
```

6. Tell the user:

```
Feature Concept approved and saved to plan/spec/Story-XXXX-slug/.
Ready to write the Feature Spec with /spec [story-id] [slug]
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "Skip the concept, go straight to FS" | The concept prevents scope drift. Without it, assumptions hide in the FS and surface during implementation — the most expensive time to discover them. |
| "The concept is obvious" | If it's obvious, writing it takes 5 minutes. If it's wrong, discovering that during implementation takes days. |
| "UX/UI can happen later" | If the feature needs design, starting the FS without it means rewriting ACs after design review. Decide now. |
| "Success metrics are premature" | Without a success metric, you can't tell if the feature worked. Define it now or accept that you'll never measure it. |
| "This is too small for a concept" | Small features don't need long concepts, but they still need a WHAT and WHY. Two sentences is fine. Zero is not. |

## Red Flags

- Starting an FS without an approved Feature Concept
- WHAT describes implementation ("Add a Redis cache") instead of capability
- No WHO — if no persona benefits, question whether to build it
- UX/UI marked "not needed" without reasoning
- No success metric
- Concept with >3 distinct capabilities that hasn't been split

## Verification

Before presenting to the human:

- [ ] WHAT describes a user capability, not implementation
- [ ] WHY states a business justification
- [ ] WHO names a persona from `plan/reference/personas.md`
- [ ] SUCCESS is a measurable business outcome
- [ ] UX/UI decision is explicit with reasoning
- [ ] DEPENDS ON / BLOCKS checked against existing `plan/spec/` features
- [ ] Assumptions are listed and confirmed by human
- [ ] Scope check completed (one feature or split)
- [ ] Domain terminology matches `plan/reference/glossary.md`
