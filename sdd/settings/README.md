# SDD settings (templates + rules)

What `/sdd/*` commands read. Do not document this folder inside steering files.

## Templates — `templates/`

### Specs — `templates/specs/`

| File | Used by | Role |
|---|---|---|
| `init.json` | `spec-init` | Initial `spec.json` (phase, language, approvals) |
| `requirements-init.md` | `spec-init` | Project-description stub only |
| `requirements.md` | `spec-requirements` | EARS document structure |
| `design.md` | `spec-design` | Architecture / contracts structure |
| `research.md` | `spec-design` | Discovery log structure |
| `tasks.md` | `spec-tasks` | Task checklist format, `(P)` markers |

Placeholders the init command replaces: `{{FEATURE_NAME}}`, `{{TIMESTAMP}}`, `{{PROJECT_DESCRIPTION}}`, `{{LANG_CODE}}`.

### Steering — `templates/steering/`

| File | Becomes |
|---|---|
| `product.md` | `.sdd/steering/product.md` |
| `tech.md` | `.sdd/steering/tech.md` |
| `structure.md` | `.sdd/steering/structure.md` |

Used by `/sdd/steering` on first run (bootstrap) or when those files are missing.

### Custom steering — `templates/steering-custom/`

Scaffolds for `/sdd/steering-custom`: `api-standards`, `authentication`, `database`, `deployment`, `error-handling`, `security`, `testing`.

## Rules — `rules/`

| File | Used by |
|---|---|
| `steering-principles.md` | `steering`, `steering-custom` |
| `ears-format.md` | `spec-requirements` |
| `design-principles.md` | `spec-design` |
| `design-discovery-full.md` | `spec-design` (new / complex features) |
| `design-discovery-light.md` | `spec-design` (extensions) |
| `design-review.md` | `validate-design` |
| `tasks-generation.md` | `spec-tasks` |
| `tasks-parallel-analysis.md` | `spec-tasks` (omit with `--sequential`) |
| `gap-analysis.md` | `validate-gap` |

## Customizing for a new project

Keep heading shapes, numeric requirement IDs, task checkboxes, and approval field names.  
Add project-specific sections or examples after those contracts. Changing IDs or checkbox syntax will break later commands.
