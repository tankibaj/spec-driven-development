# Prioritization Guide

Frameworks for ordering backlog items. Load when the user asks to prioritize work.

## Priority Levels

| Priority | Definition | Sprint Target |
|----------|------------|---------------|
| Critical | Blocking users, security risk, data loss | Immediate — current sprint |
| High | Core functionality, key user need | This sprint or next |
| Medium | Improvement, enhancement | Next 2-3 sprints |
| Low | Nice-to-have, minor improvement | Backlog — revisit quarterly |

## Frameworks

### Value vs. Effort Matrix

Quick prioritization for small backlogs:

```
            Low Effort       High Effort
           ┌────────────────┬────────────────┐
High Value │ DO FIRST       │ PLAN CAREFULLY │
           │ Quick wins     │ Major projects │
           ├────────────────┼────────────────┤
Low Value  │ FILL GAPS      │ SKIP / DEFER   │
           │ If time allows │ Not worth it   │
           └────────────────┴────────────────┘
```

### MoSCoW

| Category | Meaning | Guideline |
|----------|---------|-----------|
| **Must** | Product fails without this | ~60% of capacity |
| **Should** | Important but has workaround | ~20% of capacity |
| **Could** | Nice-to-have, improves experience | ~10% of capacity |
| **Won't** | Explicitly deferred — not this time | Documented for future |

### WSJF (Weighted Shortest Job First)

For larger backlogs needing numerical ranking:

```
WSJF = Cost of Delay / Job Duration

Cost of Delay = User Value + Time Criticality + Risk Reduction
Scale: 1, 2, 3, 5, 8, 13, 20

Example:
Feature A: CoD = 13, Duration = 5 → WSJF = 2.6
Feature B: CoD = 8,  Duration = 2 → WSJF = 4.0 ← Higher priority
```

### RICE

For data-driven teams with usage metrics:

```
RICE = (Reach × Impact × Confidence) / Effort

Reach:      Users affected per quarter (number)
Impact:     0.25 / 0.5 / 1 / 2 / 3 (minimal → massive)
Confidence: 50% (low) / 80% (medium) / 100% (high)
Effort:     Person-months (number)
```

## Choosing a Framework

| Situation | Use |
|-----------|-----|
| Quick decision, small backlog | Value vs. Effort matrix |
| Sprint planning, clear requirements | MoSCoW |
| Cross-team, competing features | WSJF |
| Data-driven team with metrics | RICE |
| Uncertain | Default to Value vs. Effort, ask the user |
