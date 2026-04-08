# CLAUDE.learnings.md — Institutional Memory

> Append-only. Prefix every entry with the date. Do not remove existing entries.
> Mark stale entries `[obsolete: reason]` if they no longer apply.
> Read at the start of every session. Append to the correct category when you discover
> something future sessions should know.

---

## Domain Rules

> Canonical facts about the domain, terminology decisions, and invariants that never change.


---

## Human Preferences

> Decisions the human has made about process, style, or approach that the agent should always honour.

- 2026-04-08: `/new-spec` replaced by `/draft-feature-spec` skill (`.claude/skills/draft-feature-spec/SKILL.md`) following the Anthropic recommended skill format (YAML frontmatter + supporting files). The old `.claude/commands/new-spec.md` is deprecated. The skill helps the human draft a Feature Spec from scratch — the human reviews and owns the final FS.
- 2026-04-08: Feature Spec template lives at `.claude/skills/draft-feature-spec/template.md` — includes Anatomy table, AC quality criteria, standardised Testable line formats (API / UI / System), and the full FS markdown template with guidance comments.

---

## Technical Gotchas

> Implementation-level findings: edge cases discovered, framework quirks, patterns that failed,
> constraints that are not obvious from the specs.
