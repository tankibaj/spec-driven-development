# Rule: Run the Definition of Done Before Marking a WP Complete

Before setting `phase_4.WP-XXX.status` to `done` in `status.yaml`, confirm every item:

1. All TS scenario tests pass
2. Linter passes (`ruff check` / `biome check`)
3. Type checker passes (`mypy` / `tsc --noEmit`)
4. Contract validation passes (Schemathesis / Pact / MSW)
5. Observability endpoints implemented (`/health`, `/ready`, `/metrics`, structured logging)
6. No secrets, tokens, or credentials in code or test fixtures

If any item fails, fix it before marking done. Do not mark done with known failures.
