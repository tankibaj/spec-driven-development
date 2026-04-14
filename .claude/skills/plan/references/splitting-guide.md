# AC-to-WP Splitting Guide

Use during Step 3.2 of the plan workflow. Defines when to split ACs into separate WPs and when to keep them together.

---

## Size Rules

| Rule | Threshold |
|---|---|
| **Ideal** | 1 AC per WP |
| **Maximum** | 3 ACs per WP |
| **Split signal** | Agent would need more than one feature area in context |

---

## Split When

- **Different workspace** — AC touches BE and FE → separate WPs, always
- **Different feature area** — AC-001 is authentication, AC-002 is payment → separate WPs
- **More than 3 ACs accumulated** — split the largest group first
- **Different test infrastructure** — one AC needs DB + Redis, another only needs HTTP mocks → consider splitting
- **One AC is a blocker for others** — isolate the blocking AC so it can be completed and deployed first

## Keep Together When

- **Tightly coupled** — AC-001 (create resource) and AC-002 (validate resource) can't be tested independently because one creates the state the other needs
- **Same endpoint, multiple validations** — several validation ACs for the same API endpoint are often better together (but max 3)
- **Shared setup cost** — if splitting would require duplicating extensive test infrastructure in both WPs, keep together

---

## Decision Tree

```
FOR each AC in the FS:

  1. Which workspace(s) does this AC touch?
     → If multiple: AC appears in one WP per workspace (full text in each)

  2. Group ACs by workspace

  3. Within each workspace, sub-group by feature area

  4. For each sub-group:
     IF count > 3:
       → Split into groups of 1-3, keeping tightly coupled ACs together
     IF count ≤ 3 AND all ACs are in the same feature area:
       → Keep as one WP
     IF ACs are in different feature areas:
       → Split by feature area (even if total ≤ 3)
```

---

## Dependency Order

After splitting, determine the execution order:

```
IF BE WP creates endpoints that FE WP consumes:
  → BE first, FE second
  → FE WP note: "Mock against contracts/api/{service}.openapi.yaml
    until WP-XXX-BE is deployed. Then switch to real backend URL."

IF multiple BE WPs are independent of each other:
  → Can run in parallel with separate agents

IF multiple FE WPs share no state:
  → Can run in parallel

IF contracts for all endpoints already exist and are stable:
  → Recommend parallel execution — all WPs use mocks for dependencies
```

---

## Mock Strategy

| Situation | Strategy |
|---|---|
| FE depends on BE being built | FE uses Prism or MSW against the OpenAPI spec |
| BE depends on external services | BE uses respx (Python) or nock (Node) mocks in tests |
| BE + FE can both start immediately | Contracts are stable; both mock their dependencies |
| FE depends on another feature's BE | Check if that feature's contracts exist; if yes, mock against them |

---

## Feature Size Limits

| Total ACs in FS | Expected WPs | Guidance |
|---|---|---|
| 1-3 | 1-2 (BE, FE) | Normal — single pass |
| 4-8 | 2-4 | Normal — single pass |
| 9-15 | 4-8 | Large — flag to human: "This feature produces N WPs. Consider splitting the FS into smaller features." |
| 16+ | 8+ | Too large — STOP. Tell the human: "This feature is too large for reliable WP generation. Recommend splitting the FS into 2-3 smaller features before proceeding." |

The risk is not in generating many WPs — it's in the agent losing coherence across them. More WPs = more chances for duplicate scenarios, missed coverage, or inconsistent contract excerpts.

---

## Common Splitting Mistakes

| Mistake | Fix |
|---|---|
| One WP with 5+ ACs | Split. The implementation agent will lose track of requirements. |
| FE and BE in the same WP | Always separate. Different workspace = different WP. |
| AC appears in WP by reference ("see AC-003 in FS") | Copy the full AC text verbatim into the WP. |
| TS scenario split across WPs (half the assertions in each) | The full scenario must be present in every WP that needs it. |
| No dependency order stated | State it explicitly, even if it seems obvious. |
