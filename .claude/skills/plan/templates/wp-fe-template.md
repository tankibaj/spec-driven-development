# Frontend Work Package Template

Structure for WP-XXX-FE.md documents. The implementation agent reads ONLY this file during Phase 4. Every section must be self-contained.

---

## Template

```markdown
# WP-XXX-FE: {Feature Title} — Frontend Work Package

**Feature:** XXX-{slug}
**Target workspace:** `workspaces/{app-id}`
**Status:** awaiting_review
**Generated:** YYYY-MM-DD

---

## Objective

{One paragraph: what this WP delivers. Note which UI flow it implements.}

This WP is self-contained. An implementer should be able to complete it reading only this file.

---

## Acceptance Criteria (verbatim from FS-XXX)

{Copy only the ACs relevant to frontend. Some ACs may be shared with the BE WP — include the full text in both.}

### AC-XXX — {title}

{Full behavioral description — FE-relevant aspects.}

**Testable:** {Testable line — FE-relevant verification.}

<!-- Repeat for each AC assigned to this WP. -->

---

## Test Scenarios (verbatim from TS-XXX)

{Copy only the scenarios relevant to frontend behavior.}

### TS-XXX-NNN — {scenario title}
- **Preconditions:** {UI state, API mock responses}
- **Action:** {user action: click, fill form, navigate}
- **Expected:** {observable UI outcome: displayed text, navigation, error message}

<!-- Repeat for each scenario assigned to this WP. -->

---

## UI Flow

```
{ASCII flow diagram showing the step-by-step user journey}
[Page A] → [Page B] → [Page C] → [Confirmation]
```

### Step N — {Step name}

{What the user sees, what fields are available, what actions they can take.}

<!-- For form steps, include a field table: -->

| Field | Label | Required | Validation |
|---|---|---|---|
| `field_name` | Display Label | Yes/No | Rule (e.g., valid email, min 2 chars) |

**CTA:** "{Button text}"

<!-- Repeat for each step in the flow. -->

---

## Error Handling

### {Error scenario name}

**When:** {condition that triggers this error}
**Display:** {exact message or message pattern shown to the user}
**Behavior:** {what happens to the UI: button re-enabled, navigate back, stay on page}

<!-- Repeat for each error scenario. -->

---

## State Management

{Where checkout/feature state lives (context, store, component state).}
{What is persisted vs transient.}
{What happens when the user navigates away and returns.}

---

## Routing

| Route | Component | Notes |
|---|---|---|
| `{path}` | `{ComponentName}` | {description, guards} |

{Note any route guards: e.g., "All routes after /checkout/guest redirect to entry if no session token."}

---

## Related Contracts

{Link to the contract docs this WP references during development and testing.}

- `workspaces/{service}/docs/api/openapi.json` (auto-generated, read-only) — {which BE operations are consumed}
- `workspaces/{app}/docs/routes.md` (auto-generated, read-only) — {which routes this WP adds or modifies}
- `workspaces/{app}/docs/consumed-endpoints.md` (auto-generated, read-only) — {which BE endpoints the FE calls}

**Mock strategy:** Use {Prism / MSW} against the auto-generated OpenAPI spec above during development. Switch to the real backend URL once WP-XXX-BE is deployed.

---

## Definition of Done

Before marking this WP done:
- [ ] All TS scenarios from this WP have passing automated tests
- [ ] `biome check` passes with zero errors
- [ ] `tsc --noEmit` passes with zero errors
- [ ] No secrets, tokens, or credentials in code or test fixtures
```
