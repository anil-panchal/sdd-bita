<meta>
description: Create or update a spec-driven PR with automatic linking to SDD artifacts and labels
argument-hint: [feature-name] [base-branch]
</meta>

# SDD Pull Request Create

Create or update a GitHub Pull Request with automatic linking to Spec-Driven Development (SDD) artifacts. If a PR already exists for the current branch, it will be updated with SDD markers (body and labels) instead of creating a new one.

## Usage

```
/sdd/spec-pr-create [feature-name] [base-branch]
```

**Parameters:**
- `feature-name` (optional): The spec feature name. Auto-detected from branch if omitted.
- `base-branch` (optional): Target branch for the PR. Defaults to repository default branch.

## Prerequisites Check

Before proceeding, verify these prerequisites:

### 1. Check GitHub CLI Installation

Run `gh --version` to verify GitHub CLI is installed.

**If not installed:**
```
GitHub CLI is required but not installed.
Install it from: https://cli.github.com

macOS:    brew install gh
Ubuntu:   sudo apt install gh
Windows:  winget install GitHub.cli
```

### 2. Check GitHub CLI Authentication

Run `gh auth status` to verify authentication.

**Note:** This command requires network access. If it fails with TLS/network errors, the token may still be valid - proceed and handle errors at PR creation time.

**If not authenticated (shows "not logged in"):**
```
GitHub CLI is not authenticated.
Run: gh auth login
```

**If network/TLS errors occur:**
```
Network error during auth check. Proceeding - will verify at PR creation.
```

### 3. Verify Git Repository

Ensure you're in a git repository with `git rev-parse --git-dir`.

**If not a git repo:**
```
This command must be run from within a git repository.
```

## Spec Detection

### Step 1: Determine Feature Name

**If feature name provided:** Use it directly.

**If feature name NOT provided:** Auto-detect from branch name:

1. Get current branch: `git branch --show-current`
2. Match against patterns:
   - `feat/<name>-*` → extract `<name>`
   - `feature/<name>-*` → extract `<name>`
   - `sdd/<name>-*` → extract `<name>`
   - `feat/<name>` → extract `<name>`
   - `feature/<name>` → extract `<name>`
   - `sdd/<name>` → extract `<name>`

3. If no pattern match, scan `.sdd/specs/` directory:
   - If 0 specs: Error "No specs found. Create one with `/sdd/spec-init`"
   - If 1 spec: Use that spec automatically
   - If multiple specs: List available specs and ask user to specify

### Step 2: Validate Spec Exists

Check that `.sdd/specs/<feature>/spec.json` exists.

**If not found:**
```
Spec '<feature>' not found.

Available specs:
- spec-a
- spec-b

Usage: /sdd/spec-pr-create <feature-name>
```

### Step 3: Load Spec Context

Read and parse these files (continue if some are missing):

1. `.sdd/specs/<feature>/spec.json` - Phase and approval status
2. `.sdd/specs/<feature>/requirements.md` - Requirements document
3. `.sdd/specs/<feature>/design.md` - Design document
4. `.sdd/specs/<feature>/tasks.md` - Tasks with completion status

**If a file is missing:** Warn but continue with available files.

## Existing PR Detection

Before creating a new PR, check if one already exists for the current branch:

```bash
gh pr view --json number,url,title,body,labels --jq '{number,url,title,body,labels: [.labels[].name]}'
```

- **If a PR exists:** Store its number, URL, current body, and labels. The flow continues to PR Body Generation and then to **Update Existing PR** (instead of creating a new one).
- **If no PR exists** (command returns non-zero): Proceed with normal creation flow.

## PR Body Generation

### For New PRs (no existing PR detected)

#### Step 1: Check for Existing PR Template

Check if `.github/PULL_REQUEST_TEMPLATE.md` exists in the repository.

#### Step 2: Generate Full PR Body

**If repo has existing PR template:**
1. Read the existing template content
2. Fill in sections based on spec context
3. Append SDD section after a separator (`---`)

**If no existing PR template:**
Use the SDD-only format below.

### For Existing PRs (updating)

Skip the PR template logic. Only generate the SDD section (`<!-- SDD:BEGIN -->` ... `<!-- SDD:END -->`), then merge it into the existing PR body as described in Step 3b.

### PR Body Template

#### With Existing Repo Template (Merged Format)

```markdown
<!-- Content from .github/PULL_REQUEST_TEMPLATE.md -->
[Existing template content preserved here]
[User fills in the existing template sections based on the spec context]

---

<!-- SDD:BEGIN -->
## 📋 Spec-Driven Development

**Feature:** `<feature-name>`
**Phase:** <phase from spec.json>

| Document | Link |
|----------|------|
| 📄 Requirements | [requirements.md](<repo-url>/blob/<head-sha>/.sdd/specs/<feature>/requirements.md) |
| 🏗️ Design | [design.md](<repo-url>/blob/<head-sha>/.sdd/specs/<feature>/design.md) |
| ✅ Tasks | [tasks.md](<repo-url>/blob/<head-sha>/.sdd/specs/<feature>/tasks.md) |

### Tasks Addressed

<list of completed tasks from tasks.md>

### Progress

<progress-bar> <percentage>% (<completed>/<total> tasks)
<!-- SDD:END -->
```

#### Without Existing Repo Template (SDD-Only Format)

```markdown
## What

<Summary of changes based on design.md and completed tasks>

## How

<Implementation approach from design.md>

## Tests

<Test coverage information - list test files added/modified>

---

<!-- SDD:BEGIN -->
## 📋 Spec-Driven Development

**Feature:** `<feature-name>`
**Phase:** <phase from spec.json>

| Document | Link |
|----------|------|
| 📄 Requirements | [requirements.md](<repo-url>/blob/<head-sha>/.sdd/specs/<feature>/requirements.md) |
| 🏗️ Design | [design.md](<repo-url>/blob/<head-sha>/.sdd/specs/<feature>/design.md) |
| ✅ Tasks | [tasks.md](<repo-url>/blob/<head-sha>/.sdd/specs/<feature>/tasks.md) |

### Tasks Addressed

<list of completed tasks from tasks.md>

### Progress

<progress-bar> <percentage>% (<completed>/<total> tasks)
<!-- SDD:END -->
```

### Step 3: Parse Tasks for Progress

Parse `tasks.md` to extract task completion:

1. Find all task checkboxes: `- [x]` (completed) and `- [ ]` (pending)
2. Count completed vs total
3. Generate progress bar: `▓` for complete, `░` for remaining (10 characters total)
4. Format: `▓▓▓▓▓▓▓▓░░ 80% (8/10 tasks)`

### Step 4: Generate Spec Links

Get the current HEAD commit SHA for permanent links:
```bash
git rev-parse HEAD
```

Get the repository URL:
```bash
gh repo view --json url -q .url
```

Format links as: `<repo-url>/blob/<sha>/.sdd/specs/<feature>/<file>`

## Label Management

### Required Labels

| Label | Color | Description |
|-------|-------|-------------|
| `sdd` | `#6f42c1` (purple) | Indicates spec-driven development PR |
| `spec/<feature>` | `#0366d6` (blue) | Links to specific feature spec |
| `phase/<phase>` | varies | Current SDD phase |

**Phase colors:**
- `phase/implementation` → `#28a745` (green)
- `phase/design` → `#fbca04` (yellow)
- `phase/requirements` → `#d93f0b` (orange)
- `phase/tasks` → `#1d76db` (blue)

### Create Missing Labels

For each label, check if it exists and create if missing:

```bash
# Check if label exists
gh label list --search "<label-name>"

# Create label if missing
gh label create "<label-name>" --color "<color>" --description "<description>"
```

**If label creation fails** (e.g., no permission): Warn and continue without that label.

## PR Creation or Update

### Step 1: Generate PR Title

Format: `feat(<feature>): <summary from first completed task or design.md overview>`

Example: `feat(ci-testing): Add GitHub Actions workflow for automated testing`

### Step 2: Write PR Body to Temp File

Write the generated PR body to a temporary file for `--body-file` (handles long content).

### Step 3a: Create New PR (no existing PR)

```bash
gh pr create \
  --title "<title>" \
  --body-file "<temp-file>" \
  --base "<base-branch>" \
  --label "sdd" \
  --label "spec/<feature>" \
  --label "phase/<phase>"
```

### Step 3b: Update Existing PR (PR already exists)

When a PR already exists for the current branch, update it with SDD markers:

#### Update PR Body

Merge the SDD section into the existing PR body:

1. Read the current PR body from the detection step
2. Check if it already contains an SDD section (look for `<!-- SDD:BEGIN -->`)
   - **If SDD section exists:** Replace everything from `<!-- SDD:BEGIN -->` to `<!-- SDD:END -->` (inclusive) with the updated SDD block
   - **If no SDD section:** Append `---` separator followed by the SDD block to the existing body
3. The SDD block must always be wrapped in markers: `<!-- SDD:BEGIN -->` ... `<!-- SDD:END -->`
4. Preserve all non-SDD content the user or other tools wrote in the PR body

```bash
gh pr edit <number> \
  --body-file "<temp-file>"
```

#### Add Missing Labels

Check which SDD labels are already on the PR and add only the missing ones:

```bash
gh pr edit <number> \
  --add-label "sdd" \
  --add-label "spec/<feature>" \
  --add-label "phase/<phase>"
```

`--add-label` is additive — it won't remove existing labels.

### Step 4: Display Result

**On new PR created:**
```
✓ Pull Request created successfully!

PR URL: <url>
Labels: sdd, spec/<feature>, phase/<phase>

Spec documents linked:
- requirements.md
- design.md
- tasks.md

Progress: <completed>/<total> tasks (<percentage>%)
```

**On existing PR updated:**
```
✓ Pull Request #<number> updated with SDD markers!

PR URL: <url>
Labels ensured: sdd, spec/<feature>, phase/<phase>

Spec documents linked:
- requirements.md
- design.md
- tasks.md

Progress: <completed>/<total> tasks (<percentage>%)
```

## Error Handling

| Error | Message | Recovery |
|-------|---------|----------|
| gh not installed | "Install GitHub CLI: https://cli.github.com" | Install gh |
| gh not authenticated | "Run `gh auth login` to authenticate" | Authenticate |
| Not a git repo | "Run from a git repository" | cd to repo |
| Spec not found | "Spec '<name>' not found. Available: [list]" | Use valid name |
| No specs exist | "No specs found. Create with `/sdd/spec-init`" | Initialize spec |
| Missing spec file | "Warning: <file> missing, continuing..." | Continue |
| Label creation failed | "Warning: Could not create label, skipping" | Continue |
| PR creation failed | Surface gh stderr | Fix and retry |
| PR update failed | Surface gh stderr | Fix and retry |
| Branch detection failed | "Specify feature: /sdd/spec-pr-create <name>" | Provide name |

## Examples

### Basic Usage (Auto-detect)
```
/sdd/spec-pr-create
```

### Specify Feature
```
/sdd/spec-pr-create ci-testing
```

### Specify Feature and Base Branch
```
/sdd/spec-pr-create ci-testing develop
```
