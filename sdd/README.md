# Spec-Driven Development toolkit

Reusable SDD pack for this repo and for copying into a **new** repository.

Commands live in `.cursor/commands/sdd/` (and `.claude/commands/sdd/`).  
This folder is the **data** those commands read: templates, rules, steering, and specs.

## Layout

```
.sdd/
├── settings/                 # Version this. Copy to every new repo.
│   ├── templates/
│   │   ├── specs/            # init, requirements, design, research, tasks
│   │   ├── steering/         # product / tech / structure scaffolds
│   │   └── steering-custom/  # optional domain steering scaffolds
│   └── rules/                # EARS, design, tasks, gap analysis, steering
├── steering/                 # Generated project memory (per repo)
└── specs/                    # Generated feature specs (per feature)
```

`settings/` is the toolkit. `steering/` and `specs/` are created by commands when you run them.

## Copy into a new repo

1. Copy these trees as-is:
   - `.sdd/settings/`
   - `.cursor/commands/sdd/`
   - `.claude/commands/sdd/` (only if that repo uses Claude Code)
2. Copy the SDD section from `AGENTS.md` (paths, phase order, approval rules).
3. Do **not** copy `.sdd/steering/` or `.sdd/specs/` from Converge. Let the new repo generate its own.
4. In the new repo `.gitignore`, ignore specs if you want them local-only, but **keep `.sdd/settings/` tracked**.

Example ignore:

```gitignore
.sdd/specs/
```

## Greenfield workflow

Run these as Cursor slash commands. Approve each phase before the next (use `-y` only as an intentional skip).

```
/sdd/steering
/sdd/spec-init "one thin vertical slice of the product"
/sdd/spec-requirements <feature>
/sdd/spec-design <feature>
/sdd/spec-tasks <feature>
/sdd/spec-impl <feature> 1.1
```

| Command | When on a new repo |
|---|---|
| `/sdd/steering` | First. If the repo is empty, tell the agent product, stack, and folder layout in the same message. |
| `/sdd/steering-custom` | Optional. One domain at a time (API, testing, auth). |
| `/sdd/spec-init` | First feature should be a thin slice, not the whole platform. |
| `/sdd/validate-gap` | Skip until there is an existing codebase. |
| `/sdd/validate-design` | Optional quality gate before tasks. |
| `/sdd/validate-impl` | After implementation tasks complete. |
| `/sdd/spec-status` | Anytime. |
| `/sdd/spec-pr-create` | When opening a PR for the feature. |

On an empty repo, `/sdd/steering` bootstraps `.sdd/steering/product.md`, `tech.md`, and `structure.md` from README / package manifests plus whatever you tell it. Review those files before the first spec.

## Phase contract

| Phase | Writes | Must not write |
|---|---|---|
| `spec-init` | `spec.json`, description stub | EARS requirements, design, tasks |
| `spec-requirements` | EARS `requirements.md` | Implementation details |
| `spec-design` | `design.md`, `research.md` | Production code |
| `spec-tasks` | `tasks.md` | Implementation |
| `spec-impl` | Code + tests, checks off tasks | Work outside the named task |

Human approval in `spec.json` (`approvals.*.approved`) is required before the next phase unless you pass `-y`.

## Source

Templates and rules are based on [cc-sdd 2.1.1](https://www.npmjs.com/package/cc-sdd/v/2.1.1) (MIT), with paths adapted from `.kiro/` to `.sdd/` to match the commands in this repo. Init templates keep `{{LANG_CODE}}` and a stub-only requirements file so phase separation holds.
