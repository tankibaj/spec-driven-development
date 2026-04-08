---
name: draft-feature-spec
description: Draft a new Feature Spec (FS) from scratch with acceptance criteria. Use when a human wants to create a spec for a new feature.
disable-model-invocation: true
argument-hint: "[story-id] [slug]"
---

# Skill: Draft a Feature Spec

Use this skill to help the human create a new Feature Spec from scratch. The human owns the final FS — you produce a draft for their review.

For the FS template, anatomy table, and AC guidance, see [template.md](template.md).

---

## Steps

### 1. Setup

If arguments were provided, use `$0` as the Story ID and `$1` as the slug. Otherwise ask the human for both.

```
Feature folder name: Story-{ID}-{slug}
Example: Story-0002-user-registration
```

1. Check `plan/spec/` for existing feature folders to avoid ID collisions.
2. Create the folder under `plan/spec/`.
3. Load domain context:
   - `plan/reference/glossary.md`
   - `plan/reference/personas.md`
   - `plan/reference/roles.md`
4. Load [template.md](template.md) as the structural guide.

### 2. Goal and Background

Ask the human:

- "What should this feature do and why? Who benefits?"
- "What is the business context? Why now?"

Draft the **Goal** section — name the primary persona from `plan/reference/personas.md`.

Draft the **Background** section. This section is recommended but the human may skip it.

### 3. Assumptions

Ask the human: "What are we taking for granted? What, if wrong, would change this spec?"

List each assumption as a bullet point. Common categories:

- Existing integrations or services assumed to be available
- Data or state assumed to exist
- User capabilities or access levels assumed

### 4. User Flow

For multi-step features, collaborate with the human to map out the end-to-end sequence. Number each step and note which future AC(s) it will cover.

For single-action features, this section may be skipped.

### 5. Acceptance Criteria

This is the core of the spec. For each step in the user flow (or each aspect of the goal):

1. **Propose ACs one at a time.** Present each AC with:
   - `AC-XXX` ID and descriptive title
   - Behavioral description (what happens, not how to implement)
   - `Testable:` line using the standardized formats from [template.md](template.md)

2. **After each happy-path AC, propose error/edge-case ACs** where the happy path can fail. Ask the human: "Can this fail? What should happen when it does?"

3. **Cross-check each AC** against:
   - `contracts/api/` — do referenced endpoints exist?
   - `contracts/data-schema/` — do referenced entities exist?
   - `plan/reference/glossary.md` — is domain terminology correct?

4. **Flag missing contracts.** If an AC references an API or entity that does not exist yet, note it as "(to be created in Phase 3)" in the Related Contracts section.

5. **Flag new domain terms.** If the feature introduces terms not in `glossary.md`, note them for glossary addition.

### 6. Open Questions

Capture anything that came up during the conversation that remains unresolved. Each question is a checkbox item.

If open questions remain, the FS stays at `Draft` status and cannot advance to Phase 2.

### 7. Remaining Sections

Complete these sections with the human:

- **Non-Functional Requirements** — ask: "Any performance, security, or scalability requirements?" This section is optional.
- **Out of Scope** — ask: "What should we explicitly exclude from this feature?" This section should be non-empty.
- **Related Contracts** — scan `contracts/` and link relevant OpenAPI specs, data schemas, and ADRs. Mark missing ones as "(to be created in Phase 3)".
- **Depends on / Blocks** — ask: "Does this feature depend on any other feature? Does it block any?" Cross-check against existing `status.yaml` files in `plan/spec/`.

### 8. Assemble and Present

1. Determine the next available `FS-XXX` ID by listing existing files in the feature folder.
2. Assemble the full draft FS using the template structure from [template.md](template.md).
3. For reference, see `plan/spec/Story-0001-guest-checkout/FS-001.md` as a complete example of a well-formed FS.
4. Run the self-verification checklist (below).
5. Present the complete draft to the human.
6. **STOP.** Wait for the human to review and sign off on the draft content.

### 9. Finalize

After the human signs off on the draft content:

1. Write the FS file to the feature folder as `FS-XXX.md`.
2. Create `status.yaml` in the feature folder (status stays `draft` — formal approval happens in Phase 1 review via `/review-spec`):

```yaml
feature: Story-{ID}-{slug}
current_phase: 1

artifacts:
  FS-XXX: { status: draft, date: YYYY-MM-DD }

phase_4: {}

blockers: []
notes: ~
```

3. Confirm to the human: "FS is saved. Ready to proceed to Phase 2 (Test Spec generation) when you approve."

---

## Checklist before presenting to human

Run this checklist before presenting the draft. Every item must pass.

- [ ] Header block complete (folder, status: Draft, author, date, depends on, blocks)
- [ ] Goal names a primary persona from `plan/reference/personas.md`
- [ ] Every AC has a unique `AC-XXX` ID and descriptive title
- [ ] Every AC has a `Testable:` line with a concrete verification method
- [ ] `Testable:` lines use the standardized format (API / UI / System)
- [ ] Error/edge-case ACs exist for every happy-path AC that can fail
- [ ] Domain terminology matches `plan/reference/glossary.md`
- [ ] ACs referencing APIs are cross-checked against `contracts/api/`
- [ ] ACs referencing data models are cross-checked against `contracts/data-schema/`
- [ ] Open questions are captured (or section is empty if all resolved)
- [ ] Out of Scope section is present and non-empty
- [ ] Related Contracts section links existing contracts and flags missing ones
- [ ] New domain terms are flagged for glossary addition
- [ ] No acceptance criteria are invented beyond what the human described — flag suggestions separately
- [ ] `Depends on` and `Blocks` header fields are populated (or explicitly set to `— (none)`)
