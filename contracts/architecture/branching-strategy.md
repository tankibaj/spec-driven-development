# Branching Strategy

**Status:** Active
**Last updated:** 2026-04-04
**Applies to:** Spec-hub repo and all workspace repos

---

## 1. Core Rules

- `main` is always deployable. No one — human or agent — commits directly to `main`.
- Every unit of work gets a branch. Branches are cheap; broken `main` is not.
- Branch names carry the Story ID, making every branch traceable to a Jira ticket without
  opening any tooling.

---

## 2. Branch Naming Conventions

### Spec-hub (this repo)

One branch per feature, covering all spec phases (Phase 2 through Phase 3, plus status updates).

```
spec/{story-ID}-{slug}
```

| Example | Meaning |
|---|---|
| `spec/story-0001-guest-checkout` | All spec artifacts for guest checkout feature |
| `spec/story-0002-user-registration` | All spec artifacts for user registration feature |

A single branch covers the TS, all WPs, contract changes, and `status.yaml` updates for that
feature. The human reviews and merges it when the feature reaches Phase 4.

### Workspace repos (where code lives)

One branch per Work Package. BE and FE always get separate branches.

```
feat/{story-ID}-{WP-ID}
```

| Example | Meaning |
|---|---|
| `feat/story-0001-WP-001-BE` | Backend implementation of guest checkout |
| `feat/story-0001-WP-001-FE` | Frontend implementation of guest checkout |
| `feat/story-0002-WP-002-BE` | Backend implementation of user registration |

Using the WP ID means two agents can work on different WPs in the same workspace repo
simultaneously without branch conflicts.

---

## 3. Commit Message Convention

Reference both the Story ID and WP ID in every commit message.

```
{type}({story-ID}): {short description}

{optional body}

WP: {WP-ID}
```

### Types

| Type | When to use |
|---|---|
| `feat` | New feature code |
| `fix` | Bug fix |
| `test` | Adding or updating tests |
| `chore` | Scaffold, dependencies, config |
| `refactor` | Code restructure without behaviour change |
| `spec` | Spec-hub only — new or updated spec artifacts |

### Examples

```
feat(story-0001): implement guest order placement saga

Implements the 6-step saga: validate cart → reserve stock → create payment
intent → capture payment → create order record → dispatch notification.

WP: WP-001-BE
```

```
spec(story-0002): generate test spec and work packages

Covers 6 ACs across 18 test scenarios. Parallel implementation recommended.

WP: TS-002, WP-002-BE, WP-002-FE
```

```
chore(story-0001): bootstrap order-service workspace

Applies standard FastAPI scaffold per workspace-bootstrap.md.

WP: WP-001-BE
```

---

## 4. Merge Strategy

| Repo | Strategy | Why |
|---|---|---|
| Workspace repos | **Squash merge** | Keeps `main` history clean. Implementation commits are noise; the squashed commit message references the Story + WP. |
| Spec-hub | **Regular merge (no squash)** | Spec artifact history is valuable. Each commit shows exactly which artifact was added or changed and when. |

---

## 5. Branch Protection (Recommended)

Configure the following on `main` in every repo:

- Require pull request before merging
- Require at least 1 approval
- Require status checks to pass (CI must be green)
- No direct pushes — not even from admins

---

## 6. Agent Instructions

### Before the first commit in the spec-hub (Phase 2)

Before creating the TS or any spec artifact, ask the human:

> "Should I create a feature branch `spec/{story-ID}-{slug}` for this work, or will you manage branching?"

- If **yes** → create the branch and commit all spec artifacts there.
- If **human manages** → commit to whatever branch is currently active. Do not create a branch.

One question. One time per feature. Do not ask again for Phase 3 work on the same feature —
it continues on the same branch.

### Before the first commit in a workspace (Phase 4)

Before the bootstrap commit or any implementation commit, ask the human:

> "Should I create a feature branch `feat/{story-ID}-{WP-ID}` for this work, or will you manage branching?"

- If **yes** → create the branch and commit all implementation work there.
- If **human manages** → commit to whatever branch is currently active. Do not create a branch.

One question. One time per WP. Do not ask again mid-implementation.

### Never

- Commit directly to `main` in any repo.
- Create a branch without asking first (unless explicitly told to proceed autonomously).
- Push to a remote branch without first confirming the branch exists locally.
