---
name: review-changed-code
description: Review code changes made by another agent or developer and report actionable findings with severity and rationale.
---

# Review Changed Code Skill

## When to use
Use this skill when:
- asked to review recent code changes
- reviewing another agent’s implementation
- validating a refactor
- checking applied suggestions after a previous review round

## Goal
Find real issues, not cosmetic noise.

## Review priorities
Review in this order:
1. correctness / regressions
2. architecture and monorepo placement
3. Angular anti-patterns
4. RxJS / async safety
5. readability and maintainability
6. performance issues in Angular templates or change-heavy flows

## Review method
For each changed area:
- identify intended behavior
- compare implementation against the spec or request
- inspect for broken edge cases
- check if logic belongs in app vs shared lib
- check naming, complexity, duplication, and coupling
- check whether the implementation introduces hidden side effects

## Output format
### Summary
One paragraph on overall quality and risk.

### Findings
For each finding, use:
- Severity: Critical / High / Medium / Low
- File / area
- Issue
- Why it matters
- Suggested fix

### Clean bill of health
If no material issues are found, say:
- No material findings
- Residual risks
- Optional improvements

## Important behavior
- Do not praise unnecessarily.
- Prefer fewer high-value findings over many weak comments.
- Distinguish between true defects and optional refactors.
- If reviewing a second pass, explicitly verify whether prior findings were resolved.