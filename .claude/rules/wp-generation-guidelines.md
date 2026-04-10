# Rule: Work Package Generation Guidelines

When generating Work Packages (WPs), optimize for AI agent executability:

1. **One AC per WP is ideal.** Maximum 3 ACs per WP. More ACs = more context for the agent to juggle = higher hallucination risk.
2. **Copy ACs and TS scenarios verbatim** into each WP. The agent reads only the WP — not the FS or TS.
3. **Each WP targets exactly one workspace.** Never mix backend and frontend in a single WP.
4. **State dependency order explicitly:** "WP-001-BE must deploy before WP-002-FE can switch from mock to real endpoint."
5. **Mark parallel opportunities.** If two WPs have no dependency, note that they can execute simultaneously with separate agents.
6. **Include contract excerpts** — the relevant OpenAPI paths and data schemas — directly in the WP. The agent should not need to read `contracts/` during implementation.

A WP is too large if an agent would need to hold more than one feature area in context to complete it. When in doubt, split.
