# Spec-first feature or fix

Follow the repo workflow in `AGENTS.md` and `.cursor/rules/01-project-general.mdc`.

1. **Understand** the request and locate the relevant code (staff vs concierge vs `libs/`).
2. **Write a short spec** before coding: problem, affected files/modules, assumptions, constraints, implementation steps, risks/regressions. Skip only for trivial one-line fixes.
3. **Place changes correctly**: shared logic in `libs/`; app-only orchestration in `apps/staff` or `apps/concierge`.
4. **Implement** the smallest safe change; match Angular 13 and existing patterns (services for API/transforms, simple templates).
5. **Summarize** what changed, why, and any follow-up risks.

Then implement or review as the user asked.
