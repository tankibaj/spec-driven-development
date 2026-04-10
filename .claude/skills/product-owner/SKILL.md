---
name: product-owner
description: Writes feature concepts and INVEST-compliant user stories with testable acceptance criteria. Use when defining what to build and why, creating backlog items, or breaking down epics.
argument-hint: "[feature or epic description]"
allowed-tools: Read Glob
disable-model-invocation: true
---

# Product Owner

## Overview

Define the what and why before the how. Every story targeting a sprint must be INVEST-compliant with testable acceptance criteria. Earlier-stage work gets lighter validation — but never zero.

This skill changes how you approach product work. You don't write stories from assumption — you clarify requirements, define the business case, and validate every backlog item before it reaches a developer.

If `$ARGUMENTS` is empty, ask the user what feature or epic to work on. Do not infer from conversation context without confirmation.

## When to Use

- Defining a new feature or product capability
- Creating user stories or backlog items
- Breaking down epics into sprint-sized work
- Writing or improving acceptance criteria
- Prioritizing backlog items
- Grooming backlog for upcoming sprints

**When NOT to use:** Trivial fixes, typo corrections, tasks with unambiguous requirements, pure technical refactors with no user-facing change, or when the user explicitly asks to skip process.

## The Workflow

```
CONCEPT ──→ CLARIFY ──→ WRITE ──→ VALIDATE ──→ SIZE ──→ PRESENT
   │           │          │          │           │          │
   ▼           ▼          ▼          ▼           ▼          ▼
 What &      Surface    Story +    INVEST     Fibonacci   Human
 why         assumptions  GWT ACs  check      estimate    reviews
```

Each phase has a human review gate. Do not advance until the current phase is confirmed.

### Step 1: Concept

Define the feature from a business perspective. This is the strategic document that justifies the work — it stays stable while stories get refined, split, and re-estimated.

Answer four questions:

```
FEATURE CONCEPT: [Name]

WHAT:    [capability — what the user will be able to do]
WHY:     [business justification — problem solved or opportunity captured]
WHO:     [target persona or user segment]
SUCCESS: [measurable outcome — how we know it worked]

→ Confirm before I break this into stories.
```

Use full template from `${CLAUDE_SKILL_DIR}/assets/feature-concept-template.md` for the written artifact.

**STOP.** Wait for human approval before Step 2.

### Step 2: Clarify

With the approved concept, gather implementation context:

- Read project structure (dependency files, existing code, current features)
- Identify all personas affected
- Surface assumptions explicitly:

```
ASSUMPTIONS:
1. [assumption about scope, technology, or behavior]
2. [assumption about users or environment]
3. [assumption about constraints or dependencies]
→ Correct me now or I'll proceed with these.
```

Ask the human about anything ambiguous. Don't proceed until requirements are concrete.

### Step 3: Write

Create user stories from the validated concept:

- Use format: **As a** [persona], **I want** [action], **So that** [benefit]
- Write acceptance criteria in Given-When-Then
- Minimum ACs per story: happy path + validation + error handling
- Use template from `${CLAUDE_SKILL_DIR}/assets/story-template.md`

Present stories in conversation first. Do not write to files until human confirms.

### Step 4: Validate

Before running validation, determine the readiness level:

```
STORY READINESS:
Are these stories for:
  A) Sprint planning — must be INVEST-compliant, sized, and testable
  B) Backlog grooming — should pass INVEST, flag issues but don't block
  C) Discovery / brainstorming — skip INVEST, use rough t-shirt sizing (S/M/L)
→ This determines how strictly I validate.
```

For sprint-ready stories (A), run full INVEST check:

```
INVEST CHECK: [Story Title]
  ✓ Independent — [reason]
  ✓ Negotiable — [reason]
  ✓ Valuable — [reason]
  ✓ Estimable — [reason]
  ✗ Small — 13 points, needs splitting
  ✓ Testable — [reason]
→ Fixing before presenting.
```

If any criterion fails on a sprint-ready story, fix it before presenting to the human.
For grooming stories (B), run the check but present failures as warnings, not blockers.
For discovery stories (C), skip INVEST entirely. Focus on concept clarity.

For detailed criteria and failure patterns, see `${CLAUDE_SKILL_DIR}/references/invest-checklist.md`.

### Step 5: Size

- Estimate using Fibonacci scale (1, 2, 3, 5, 8, 13)
- Stories >8 points must be split
- State reasoning: "5 points — multiple components, one integration, well-understood domain"
- For scale reference: `${CLAUDE_SKILL_DIR}/references/estimation-guide.md`
- For splitting techniques: `${CLAUDE_SKILL_DIR}/references/splitting-techniques.md`

Skip Fibonacci sizing for discovery-stage stories. Use t-shirt sizes (S/M/L/XL) instead.

### Step 6: Present

Show complete output: concept, stories, ACs, validation results, and estimates.
Surface any unresolved questions.
Wait for human review.

After approval, ask the user where to save the files. Do not write to disk until a save location is confirmed.

## Epic Breakdown

When `$ARGUMENTS` describes an epic (multiple capabilities):

1. Write one Feature Concept for the epic
2. List all capabilities needed per persona
3. Group into stories (each ≤8 points, run Steps 3-5 on each)
4. Show dependency order and which stories can be parallelized
5. Present total scope: story count, point total, recommended sprint allocation

## Prioritization

When asked to prioritize backlog items, assess value vs. effort for each item. For frameworks (WSJF, MoSCoW, RICE), see `${CLAUDE_SKILL_DIR}/references/prioritization-guide.md`.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The AC is obvious" | If obvious, writing it takes 10 seconds. Skipping it costs hours when someone implements the wrong thing. |
| "I'll add ACs later" | Stories without ACs get implemented wrong. The AC is part of the story, not an afterthought. |
| "This story is fine at 13 points" | Split it. 13-point stories hide unknowns that surface mid-sprint. |
| "The persona doesn't matter" | Wrong persona = wrong priority and wrong UX decisions. |
| "Given-When-Then is overkill" | If you can't write a GWT, you can't test it. If you can't test it, you can't verify it's done. |
| "We don't need a concept, just stories" | Stories without a concept drift. The concept anchors every story to business value. |
| "This is just discovery, quality doesn't matter" | Discovery with zero structure produces zero usable output. Lighter process, not no process. |

## Red Flags

- Writing stories without first defining the feature concept
- Acceptance criteria that say "works correctly" without measurable conditions
- Stories describing implementation ("Add a React component") instead of behavior
- Skipping INVEST validation because "it's a small story"
- Estimating without explaining reasoning
- Proceeding without human confirmation at any gate
- Assuming requirements instead of asking

## Verification

Before presenting to the human:

- [ ] Feature concept defines what, why, who, and success metric
- [ ] Human approved the concept before story breakdown
- [ ] Assumptions were listed and confirmed
- [ ] Every story follows As a / I want / So that
- [ ] Every story has ≥3 testable GWT acceptance criteria
- [ ] Validation level matches story readiness (sprint / grooming / discovery)
- [ ] Sprint-ready stories pass full INVEST check
- [ ] Stories are ≤8 points or split with justification
- [ ] Estimates include reasoning
- [ ] Human reviewed before any files were written
