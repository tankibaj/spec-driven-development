# Rule: Branching Strategy

Always create feature branches automatically. Never ask the human — just follow the conventions below.

## Branch Naming

| Repo | Pattern | Example |
|---|---|---|
| Spec-hub | `spec/{feature-ID}-{slug}` | `spec/001-guest-checkout` |
| Workspace | `feat/{feature-ID}-{WP-ID}` | `feat/001-WP-001-BE` |

- **Spec-hub:** One branch per feature. Covers all spec artifacts (TS, WPs, contract changes, status updates). Create the branch before the first spec commit.
- **Workspace:** One branch per Work Package. BE and FE always get separate branches. Create the branch before the first implementation commit.

## Commit Messages

```
{type}({feature-ID}): {short description}

{optional body}

WP: {WP-ID}
```

Types: `feat`, `fix`, `test`, `chore`, `refactor`, `spec`

## Rules

- Never commit directly to `main` in any repo.
- If already on a correctly named feature branch, use it — do not create a new one.
- If on `main` or a wrong branch, create the correct branch and switch to it before committing.
- One branch per feature (spec-hub) or per WP (workspace). Do not reuse branches across features.
- Spec-hub uses regular merge (preserve history). Workspaces use squash merge.
