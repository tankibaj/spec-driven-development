# Splitting Techniques

Strategies for breaking large stories into sprint-sized work. Load when stories exceed 8 points during Step 5 (Size) of the product-owner workflow.

## When to Split

- Estimated at 13+ points
- More than 8 acceptance criteria
- Involves more than 2 personas
- Team cannot agree on estimate (hidden complexity)
- Title contains "and" connecting distinct capabilities

## Techniques

| Technique | When to Use | Before → After |
|-----------|-------------|----------------|
| By workflow step | Linear process | "Checkout" → "Add to cart" + "Payment" + "Confirm" |
| By persona | Multiple user types | "Dashboard" → "Admin dashboard" + "Member dashboard" |
| By data type | Multiple formats | "Import" → "Import CSV" + "Import Excel" |
| By operation | CRUD functionality | "Manage users" → "Create" + "Edit" + "Delete" |
| Happy path first | Risk reduction | "Feature" → "Basic flow" + "Error handling" + "Edge cases" |
| By platform | Cross-platform | "Mobile" → "iOS" + "Android" |
| By AC cluster | Too many ACs | 12 ACs → two stories of 6, grouped by theme |

## Decision Guide

```
Story >8 points?
├── Multiple user types?       → Split by persona
├── Multi-step process?        → Split by workflow step
├── Multiple data formats?     → Split by data type
├── CRUD on single entity?     → Split by operation
├── One flow, complex edges?   → Split happy path first
├── Cross-platform?            → Split by platform
└── Still too large?           → Split by AC clusters
```

## After Splitting — Verify Each Story

- [ ] Delivers standalone value (not "part 1 of 3")
- [ ] Passes INVEST independently
- [ ] Is ≤8 points
- [ ] Has its own acceptance criteria (not shared with siblings)
