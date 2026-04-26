# Contract Review Checklist

Use during Step 7 (Contract Review) of the spec workflow. Review each changed contract against these criteria before including the change summary in the Feature Spec.

---

## Summary Table Format

Present contract needs to the human using this format:

```
CONTRACT CHANGES NEEDED:
| Contract | Change | Type | Status |
|----------|--------|------|--------|
| [file]   | [desc] | additive/breaking | exists / to be created |
```

## Preference Rules

Apply in this order — always prefer the least disruptive option:

1. **No contract changes** — if the feature can be delivered without modifying contracts
2. **Additive changes** — new fields/endpoints alongside existing ones
3. **New version (v1 → v2)** — new path prefix, old version deprecated
4. **Breaking change with ADR** — last resort, requires consumer approval before implementation

---

## API Contract Review (OpenAPI)

For each changed or new endpoint:

- [ ] New endpoints are additive (don't modify existing paths)
- [ ] Changed request schemas are backward compatible (new fields are optional)
- [ ] Changed response schemas don't remove existing fields
- [ ] New required headers are documented with examples
- [ ] Error responses follow existing patterns in the spec
- [ ] Versioning: if change is breaking, use a new path prefix (v2) rather than modifying v1
- [ ] Endpoint naming follows existing conventions in the spec

## Data Schema Review

For each changed or new entity:

- [ ] New columns have defaults or are nullable (no breaking migration)
- [ ] No column type changes on existing data without migration plan
- [ ] Migration is reversible (up + down scripts)
- [ ] Indexes planned for new query patterns
- [ ] No data loss on migration
- [ ] Foreign key relationships are consistent with existing schema

## Event Schema Review

For each changed or new event:

- [ ] New fields are optional (consumers that don't know about them won't break)
- [ ] No removed fields (existing consumers may depend on them)
- [ ] Schema version is incremented
- [ ] Existing consumers tested against new schema shape

---

## Breaking Change Decision Tree

If a breaking change is unavoidable:

- [ ] ADR proposed in `docs/architecture/ADR-XXX.md`
- [ ] All consumers identified (from impact analysis)
- [ ] Consumer teams notified and have approved the change
- [ ] Migration path documented (old shape → new shape)
- [ ] Deprecation timeline set for old version
- [ ] Old version continues to work until all consumers have migrated

---

## Versioning Strategy

| Situation | Strategy |
|-----------|----------|
| New endpoint | Add to existing spec (additive) |
| Modified request schema | Add optional fields only |
| Modified response schema | Add fields, never remove |
| Fundamentally different behavior | New version prefix (v2) |
| Removed endpoint | Deprecate first, remove in next major version |
| New required field on existing endpoint | Breaking — requires new version or ADR |
