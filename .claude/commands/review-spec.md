# /review-spec [feature-folder]

Review a Feature Spec for Phase 2 readiness.

1. Ask for `feature-folder` if not provided.
2. Load `plan/reference/glossary.md`, `personas.md`, `roles.md`.
3. Read the FS. Evaluate each AC: unambiguous? testable? contracts consistent?
4. Output:
   - **Ready ACs** — one-line rationale each.
   - **Problem ACs** — issue + suggested rewrite.
   - **Missing ACs** — anything implied by the goal but absent.
   - **Verdict** — Ready for Phase 2 / Needs revision.
