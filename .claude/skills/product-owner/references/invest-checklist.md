# INVEST Checklist

Detailed validation criteria for user stories. Load during Step 4 (Validate) of the product-owner workflow.

## Criteria

| Criterion | Question | Pass If |
|-----------|----------|---------|
| **I**ndependent | Can this be developed without depending on uncommitted work? | No blocking dependencies on other uncommitted stories |
| **N**egotiable | Is the implementation approach flexible? | Story describes outcome, not solution; multiple approaches possible |
| **V**aluable | Does this deliver value to a user or the business? | Clear benefit in "so that" clause; someone would notice if this wasn't built |
| **E**stimable | Can the team estimate with reasonable confidence? | Domain and technology are understood well enough to size |
| **S**mall | Can this be completed in one sprint? | ≤8 story points |
| **T**estable | Can we write a test that proves this is done? | Acceptance criteria are specific, measurable, and unambiguous |

## Failure Patterns

| Criterion | Red Flag | Fix |
|-----------|----------|-----|
| Independent | "After story X is done..." | Combine stories, resequence, or mock the dependency |
| Negotiable | Story specifies technology: "Use Redis for..." | Rewrite as outcome: "Sub-second response time for..." |
| Valuable | No "so that" clause, or benefit is developer-only | Add user or business benefit; if none exists, question whether to build it |
| Estimable | Team says "no idea" or "too many unknowns" | Create a spike (timeboxed investigation) first, then write the story |
| Small | Estimated at 13+ points | Split using techniques in splitting-techniques.md |
| Testable | Criteria use "easy," "fast," "better" without numbers | Replace with measurable conditions: "loads in <2s," "error rate <1%" |

## Validation by Readiness Level

| Readiness | INVEST Enforcement | Sizing |
|-----------|-------------------|--------|
| Sprint planning | **Must pass all 6.** Fix before presenting. | Fibonacci (1-13) |
| Backlog grooming | **Should pass.** Flag failures as warnings. | Fibonacci, provisional |
| Discovery | **Skip.** Focus on concept clarity. | T-shirt (S/M/L/XL) |

## Edge Cases

| Situation | Guidance |
|-----------|----------|
| Story depends on an external team | Independent if the external work is already committed and scheduled |
| Technical enabler with no direct user value | Valuable if it enables a user-facing story — link to the parent |
| Team disagrees on estimate | Sign the story needs more clarification, not a bigger number |
| Story has 10+ acceptance criteria | Too large — split by AC clusters into separate stories |
