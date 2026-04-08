# Feature Spec Template

This is the reference template for writing a Feature Spec (FS). Both humans and the AI agent use this as the structural guide when creating a new FS.

For the interactive drafting procedure, see [SKILL.md](SKILL.md).

---

## Anatomy of a Good Feature Spec

| Section | Purpose | Required? |
|---|---|---|
| **Header block** | Metadata: folder, status, author, date, dependencies | Yes |
| **Goal** | One paragraph: what + why + who benefits. Name the primary persona. | Yes |
| **Background** | Business context, motivation, data backing the decision | Recommended |
| **Assumptions** | What the spec takes for granted — if wrong, the spec changes | Recommended |
| **User Flow** | Numbered steps mapping to ACs — shows the end-to-end sequence | Optional (multi-step features) |
| **Acceptance Criteria** | Numbered `AC-XXX` entries, each with a `Testable:` line | Yes |
| **Open Questions** | Unresolved items that block approval — remove when FS is Approved | Yes (for Draft status) |
| **Non-Functional Requirements** | Performance, security, scalability targets | Optional |
| **Out of Scope** | Explicit boundaries — what this feature does NOT do | Recommended |
| **Related Contracts** | Links to OpenAPI specs, data schemas, ADRs | Yes, if they exist |

---

## What Makes an AC Good

1. **Unique ID and title** — `AC-001 — Guest session creation` — enables unambiguous tracing to test scenarios.
2. **Behavioral description** — states what happens, not how to implement it.
3. **`Testable:` line** — every AC ends with a concrete verification method. If you cannot write this line, the AC is too vague.
4. **Error/edge-case pairing** — happy-path ACs are paired with error/edge-case ACs where the happy path can fail (e.g., AC-005 order placement + AC-006 out-of-stock).

---

## Testable Line Formats

Use one of these standardized patterns for the `Testable:` line:

| Type | Pattern | Example |
|---|---|---|
| **API** | `{METHOD} {path}` with `{precondition}` returns `{HTTP status}` with `{response shape}` | `POST /checkout/guest/orders` with valid inputs returns HTTP 201 with `status: confirmed` and a non-empty `reference`. |
| **UI** | When `{user action}`, then `{observable outcome}` | When the shopper submits the form with a missing postal code, then a field-level error is displayed on the postal code field. |
| **System** | After `{trigger}`, `{system behavior}` within `{time bound}` | After order placement, the Notification Service is called with `template_id: order_confirmation` within 2 minutes. |

---

## Template

<!-- Remove all guidance comments when filling this in. -->

```markdown
# FS-XXX: {Feature Title}

**Feature folder:** `Story-XXXX-{slug}`
**Status:** Draft
**Author:** {name or team}
**Last updated:** YYYY-MM-DD
**Depends on:** — (none) | `FS-XXX` — {why this dependency exists}
**Blocks:** — (none) | `FS-XXX` — {why this is blocked}

---

## Goal

<!-- What does this feature do, why does it matter, and who benefits?
     Name the primary persona from plan/reference/personas.md. -->

{What this feature delivers and the business outcome it drives.}

**Primary persona:** {Name} — {Role description} (see `plan/reference/personas.md`)

---

## Background

<!-- Business context, data, motivation. Why now? What problem exists today? -->

{Context that helps the reader understand why this feature exists.}

---

## Assumptions

<!-- Things the spec takes for granted. If any assumption is wrong, the spec changes.
     Example: "Payment gateway SDK (Stripe) is already integrated." -->

- {Assumption 1}
- {Assumption 2}

---

## User Flow

<!-- Numbered steps showing the end-to-end sequence. Map each step to the AC(s) it covers.
     Optional for single-action features. -->

1. {Step description} → AC-001
2. {Step description} → AC-002
3. {Step description} → AC-003, AC-004

---

## Acceptance Criteria

### AC-001 — {Short title}

{Behavioral description of what the system does or what the user experiences.}

**Testable:** {Concrete verification — use one of the standardized formats above.}

---

### AC-002 — {Short title}

{Behavioral description.}

**Testable:** {Concrete verification.}

---

### AC-003 — {Error/edge case related to a happy-path AC}

<!-- If a happy-path AC can fail, add an AC that specifies the failure behavior. -->

{What happens when X goes wrong. What the user sees. What the system must NOT do.}

**Testable:** {Concrete verification of the error behavior.}

---

## Open Questions

<!-- Unresolved items that need answers before the FS can be approved.
     The agent will not advance to Phase 2 while open questions remain.
     Remove this section entirely when the FS moves to Approved status. -->

- [ ] {Question 1}
- [ ] {Question 2}

---

## Non-Functional Requirements

<!-- Performance, security, scalability targets. Optional — include only when relevant. -->

- {NFR 1, e.g., "Order placement API must respond within 3 seconds (p95)."}
- {NFR 2, e.g., "No PII in application logs."}

---

## Out of Scope

<!-- Explicit boundaries. Prevents scope creep during implementation. -->

- {Thing that might be assumed but is NOT part of this feature.}
- {Future enhancement deliberately excluded from this iteration.}

---

## Related Contracts

<!-- Link to existing contracts this feature touches.
     If the feature needs contracts that don't exist yet, mark them "(to be created in Phase 3)". -->

- `contracts/api/{service}.openapi.yaml`
- `contracts/data-schema/{entity}.schema.md`
- `contracts/architecture/ADR-XXX.md` — {decision title}
```
