---
name: workspace-context
description: Generate or refresh CLAUDE.md files inside workspace repos. Scans source code and CI-generated docs to produce per-workspace context files that help AI agents work effectively. Use with "refresh workspace context", "generate workspace CLAUDE.md", or "workspace docs are stale".
argument-hint: "[--all | --workspace <name> | --check-only]"
allowed-tools: Read Glob Grep Bash Agent Edit Write
---

# Skill: Workspace Context

## Purpose

Generate and maintain `CLAUDE.md` files inside each workspace submodule. These files give an AI agent immediate context about a workspace — its API surface, data model, directory layout, dev commands, and inter-service dependencies — without scanning source files from scratch.

## When to Use

- First-time setup: workspace has no `CLAUDE.md` yet
- After a major implementation phase completes (new endpoints, models, routes)
- When the agent detects a mismatch during implementation
- Phrases: "refresh workspace context", "generate workspace CLAUDE.md", "workspace docs are stale"

### When NOT to Use

- To modify CI-generated docs (`openapi.json`, `entities.md`, `routes.md`, `consumed-endpoints.md`) — those are read-only
- To write spec artifacts (PRD, FS, TS, WP) — use the dedicated SDD skills
- To document the spec-hub itself — edit the root `CLAUDE.md` directly

## Modes

| Flag | Behaviour |
|---|---|
| `--all` | Scan all workspaces in `routes.yaml` (parallel sub-agents) |
| `--workspace {name}` | Scan one workspace only |
| `--check-only` | Report drift without updating files (dry run) |

Default: `--all` if no flag specified.

## Workflow

```
routes.yaml ──► enumerate workspaces
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼     (parallel sub-agents, one per workspace)
     [scan ws-1] [scan ws-2] [scan ws-3]
          │          │          │
          ▼          ▼          ▼
     [diff report per workspace]
          │
          ▼
     [present drift report to human]
          │
          ▼
     [update CLAUDE.md after confirmation]
```

### Step 1 — Load Registry

1. Read `routes.yaml` — enumerate all workspaces with `type` and `path`
2. For `--workspace {name}`, filter to that single entry
3. Check which workspaces already have a `CLAUDE.md`

### Step 2 — Scan Source Code

Launch parallel sub-agents (one per workspace). Each agent scans the workspace and produces a structured context report.

#### Backend services — scan these:

| Location | What to Extract |
|---|---|
| `docs/api/openapi.json` | API endpoints, request/response schemas (CI-generated, authoritative) |
| `docs/schema/entities.md` | Tables, columns, relationships (CI-generated, authoritative) |
| `src/main.py` | Middleware stack, exception handlers, Prometheus setup |
| `src/config.py` | Settings class fields (env vars, types, defaults) |
| `src/api/` | Router structure, endpoint grouping |
| `src/models/` | SQLAlchemy model files, base class location |
| `src/clients/` | Inter-service HTTP clients (who this service calls) |
| `src/services/` | Business logic layer |
| `pyproject.toml` | Dependencies, Python version, linter/test config |
| `alembic/` | Migration count, alembic.ini presence |
| `tests/` | Test directory structure, conftest patterns |
| `Dockerfile`, `docker-compose.yml` | Container setup if present |

#### Frontend apps — scan these:

| Location | What to Extract |
|---|---|
| `docs/routes.md` | Route table (CI-generated, authoritative) |
| `docs/consumed-endpoints.md` | Backend API dependencies (CI-generated, authoritative) |
| `src/App.tsx` | Route tree, layout wrappers, auth guards |
| `src/main.tsx` | Provider stack, MSW setup |
| `vite.config.ts` | Plugins, path aliases, test config |
| `src/api/` or `src/hooks/` | API client files, fetch patterns |
| `src/stores/` | Zustand stores, state shape |
| `src/components/` | Shared component inventory |
| `src/features/` | Feature module structure |
| `package.json` | Dependencies, scripts, Node version |
| `tests/` or `src/test/` | Test directory structure, setup files |
| `.env.example` | Expected environment variables |

### Step 3 — Diff: Existing CLAUDE.md vs Source

If CLAUDE.md already exists, compare each section against the scan results:

| Dimension | What to Check |
|---|---|
| **Endpoints / Routes** | New endpoints/routes not in doc, removed ones still in doc |
| **Data Model** | New tables/fields, removed entities (BE only) |
| **Dependencies** | Major version changes, new/removed packages |
| **Inter-service calls** | New client files, new consumed endpoints |
| **Env vars** | New Settings fields (BE) or VITE_* vars (FE) |
| **Directory structure** | New top-level dirs, new feature modules |
| **Dev commands** | Changed scripts in package.json or pyproject.toml |
| **Middleware / Providers** | New middleware (BE), new context providers (FE) |

If CLAUDE.md does not exist, skip diffing — everything is "new".

### Step 4 — Produce Drift Report

```markdown
## Workspace Context Drift Report — {date}

### Summary
- **Workspaces scanned**: {count}
- **New (no CLAUDE.md)**: {count}
- **Drifted**: {count}
- **Up to date**: {count}

### {workspace-name} — {NEW | DRIFTED | UP TO DATE}
| Dimension | Status | Details |
|---|---|---|
| Endpoints | +2 new | `POST /auth/login`, `POST /auth/mfa/verify` added |
| Data Model | No change | — |
| Dependencies | 1 bump | FastAPI 0.115 → 0.116 |
| Env vars | +1 new | `STRIPE_API_KEY` added to Settings |

{Repeat for each workspace}
```

### Step 5 — Update CLAUDE.md (after human confirmation)

If mode is NOT `--check-only`:

1. Present the drift report to the human
2. Wait for confirmation: "update all", "update {workspace} only", or "skip"
3. Generate or update `CLAUDE.md` in each confirmed workspace using the appropriate template:
   - Backend: `.claude/skills/workspace-context/templates/be-claudemd.md`
   - Frontend: `.claude/skills/workspace-context/templates/fe-claudemd.md`
4. Fill the template with scan results — every section must reflect actual source code, not assumptions

### Step 6 — Verify

For each updated CLAUDE.md:
- [ ] Every endpoint in `openapi.json` (or `routes.md`) appears in the doc
- [ ] Every env var from Settings (or VITE_* usage) is listed
- [ ] Directory structure matches actual `ls` output
- [ ] Dev commands match `pyproject.toml` scripts or `package.json` scripts
- [ ] Inter-service dependencies match `src/clients/` (BE) or `consumed-endpoints.md` (FE)
- [ ] No references to files or functions that don't exist

## Rules

- ALWAYS scan source code — do not trust existing CLAUDE.md as current
- ALWAYS use CI-generated docs (`openapi.json`, `entities.md`, `routes.md`, `consumed-endpoints.md`) as the authoritative source for API surface and data model — they are ground truth
- ALWAYS use parallel sub-agents when scanning multiple workspaces
- NEVER auto-update CLAUDE.md without showing the drift report first
- NEVER include secrets, tokens, or real credentials in the generated docs
- If a workspace submodule is not cloned (empty dir), report it as "not available" and skip
- If a workspace has zero drift, report it as "up to date" — do not skip silently
- Keep the template structure when updating — only change content that drifted
- The generated CLAUDE.md lives inside the workspace repo and is committed to it, not to the spec-hub
