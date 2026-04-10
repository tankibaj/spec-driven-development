# Impact Analysis: [Feature Name]

**Feature:** `Story-XXXX-[slug]`
**Feature Concept:** PDR — [concept name]
**Analyst:** [PO + Architect names]
**Date:** YYYY-MM-DD

---

## Affected Services

<!-- Read registry/routes.yaml. For each workspace, assess whether this feature
     requires changes to its API, data models, or cross-service interactions. -->

| Service | Workspace | Impact | Change Type |
|---------|-----------|--------|-------------|
| [service-id from routes.yaml] | [path] | [what changes] | additive / breaking / none |

---

## Contract Changes Required

### API Changes

| Contract | Change | Breaking? | Consumer Approval |
|----------|--------|-----------|-------------------|
| [openapi file] | [description] | yes / no | [team — status] |

### Data Schema Changes

| Schema | Change | Migration Required? |
|--------|--------|-------------------|
| [schema file] | [description] | yes / no |

### Event Schema Changes

| Event | Change | Backward Compatible? |
|-------|--------|---------------------|
| [event] | [description] | yes / no |

---

## Blast Radius

**Level:** [High / Medium / Low]
**Services affected:** [count]
**Consumers affected:** [count]
**Reasoning:** [why this level — consider data loss risk, user-facing impact, number of dependent services]

---

## Consumers Affected

<!-- Which services or apps depend on the contracts being changed?
     What happens to them if the change breaks the current interface? -->

| Consumer | Depends On | Impact If Broken |
|----------|-----------|-----------------|
| [service/app] | [contract] | [consequence] |

---

## Recommended Approach

<!-- Choose one and explain reasoning:
     - Additive extension: new fields/endpoints alongside existing ones
     - New version (v1 → v2): new path prefix, old version deprecated
     - Breaking change with ADR: unavoidable, requires migration plan -->

**Approach:** [Additive extension / New version (v1→v2) / Breaking change with ADR]
**Reasoning:** [why this is the right choice]

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| [risk description] | High / Med / Low | High / Med / Low | [plan] |
