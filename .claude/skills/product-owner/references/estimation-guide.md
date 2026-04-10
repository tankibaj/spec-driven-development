# Estimation Guide

Fibonacci scale reference and estimation factors. Load during Step 5 (Size) of the product-owner workflow.

## Fibonacci Scale

| Points | Complexity | Typical Scope | Example |
|--------|------------|---------------|---------|
| 1 | Trivial | Single change, no unknowns | Fix typo, change label, update config |
| 2 | Simple | One component, minimal logic | Add form field, simple validation rule |
| 3 | Small | One feature, straightforward | New form with validation, basic CRUD endpoint |
| 5 | Medium | Multiple components, some integration | Feature with UI + API + database changes |
| 8 | Large | Complex feature, multiple integrations | Multi-step workflow, third-party integration |
| 13 | Too large | **Must split.** Hidden unknowns likely. | — |
| 21+ | Epic | **Must split into multiple stories.** | — |

## Estimation Factors

| Factor | Adds Complexity When | Reduces Complexity When |
|--------|---------------------|------------------------|
| Unknowns | New technology, unclear requirements | Well-understood domain, similar prior work |
| Dependencies | External APIs, other team's work | Self-contained, no external calls |
| Testing | Complex integration tests, many edge cases | Simple unit tests, happy-path dominant |
| Data | Schema changes, migrations, transformations | Existing models, no schema changes |
| UI | New components, complex interactions | Minor changes, existing component library |

## Reasoning Format

Always state reasoning when sizing:

```
[N] points — [factor 1], [factor 2], [confidence]

Examples:
3 points — single endpoint, existing data model, high confidence
5 points — new UI component + API change, moderate complexity
8 points — third-party integration, some unknowns in their API
```

## T-Shirt Sizing (Discovery Stage)

For stories not yet sprint-ready:

| Size | Rough Equivalent | Meaning |
|------|-----------------|---------|
| S | 1-2 points | A day or less |
| M | 3-5 points | A few days |
| L | 8 points | Most of a sprint |
| XL | 13+ points | Needs splitting before sprint planning |
