---
name: spec-driven-angular
description: Use this skill for Angular feature work, refactors, and bug fixes that should follow a lightweight spec-first workflow in this monorepo.
---

# Spec-Driven Angular Skill

## When to use
Use this skill when:
- implementing a feature
- fixing a bug
- refactoring Angular / TypeScript code
- changing shared code in `libs/`
- modifying behavior across `staff` or `concierge`

## Required workflow

### Step 1: Understand the target area
Identify:
- app or lib affected
- current flow and entry points
- whether the change is app-specific or reusable

### Step 2: Write a compact spec before coding
Produce:

#### Problem
What needs to change?

#### Scope
Which apps/libs/files are affected?


#### Assumptions
List assumptions explicitly.

#### Proposed approach
Describe the smallest safe implementation.

#### Risks
Mention likely regression points.

Do not start coding until this spec is stated.

### Step 3: Implement
- Make the smallest correct change.
- Respect local conventions in the touched area.
- Avoid unrelated refactors.
- Prefer reusable placement only when justified.

### Step 4: Self-review
Review the changed files for:
- correctness
- readability
- architectural fit
- duplication
- regression risk
- overengineering

### Step 5: Final response
Return:
- spec
- what changed
- findings from self-review
- risks / follow-up suggestions