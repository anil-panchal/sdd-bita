# Code review and open PR to `uat-placeos`

Run this workflow when invoked. Target branch for the PR is always **`uat-placeos`**. Base review scope on **all commits on the current branch not in `origin/uat-placeos`** (three-dot diff).

## Parameters

This command accepts **three parameters** in the user's input string:

| Parameter     | Required | Condition | Description |
|---------------|----------|-----------|-------------|
| `title`       | **Yes**  | —         | Pull request title (short, imperative) |
| `ticket`      | No       | —         | JIRA ticket key (e.g. `CONV-123`, `PROJ-456`) |
| `description` | No       | Only valid when `ticket` is also provided | Free-text description of what was done and why |

### Accepted input formats

The agent must parse the user's input to extract these values. All of the following formats are valid:

1. **Property-value pairs** (preferred):
   `/code-review-pr-uat title: Fix room filtering logic ticket: CONV-123`
   `/code-review-pr-uat title: Fix room filtering logic ticket: CONV-123 description: Fixed the null check on capacity filter that was causing rooms to disappear when switching floors`
2. **Comma-separated positional** (title first, ticket second, description third):
   `/code-review-pr-uat "Fix room filtering logic", CONV-123`
   `/code-review-pr-uat "Fix room filtering logic", CONV-123, "Fixed the null check on capacity filter"`
3. **Title only** (no JIRA integration, no description):
   `/code-review-pr-uat "Fix room filtering logic"`
   `/code-review-pr-uat title: Fix room filtering logic`

### Parsing rules

- If the input contains `title:`, `ticket:`, and/or `description:`, extract their values (everything after the keyword up to the next keyword or end of string). Trim surrounding quotes and whitespace.
- If the input has no keywords, treat the segments as comma-separated positional values: first = **title**, second = **ticket**, third = **description**. If there is only one segment, the entire string is the **title**.
- Store the parsed values as `$PR_TITLE`, `$JIRA_TICKET` (may be empty), and `$PR_DESCRIPTION` (may be empty) for use in subsequent steps.
- **`$JIRA_TICKET`** is considered **present** only if it is non-empty after trimming. Steps 5 and 6 execute **only** when `$JIRA_TICKET` is present.
- **`$PR_DESCRIPTION`** is ignored if `$JIRA_TICKET` is empty (description only applies when a ticket is also provided).
- If `$PR_DESCRIPTION` is provided, the agent must **rephrase it into clear, simple, human-readable language** if the original text is overly technical, verbose, or jargon-heavy. Store the rephrased version as `$DESCRIPTION_CLEAN` for use in Steps 3 and 5c.

---

## Prerequisites

- Remote **`origin`** (GitHub).
- Optional: **GitHub CLI** (`gh`) with `gh auth login` for automated PR creation.
- Current branch must be **pushed** to `origin` before `gh pr create` can succeed.
- **If `$JIRA_TICKET` is provided:** An authenticated Atlassian MCP server (`plugin-atlassian-atlassian` or `user-Atlassian-MCP-Server`) must be available. If not authenticated, call `mcp_auth` before Step 5.

**Uncommitted work:** This workflow includes **Step 0** so you do **not** have to commit manually first. If you prefer to commit yourself, do that before invoking; Step 0 will then do nothing.

## Edge cases (handle first)

1. **No commits ahead of `uat-placeos`**: After fetch (and after any commit in Step 0), if `git log origin/uat-placeos..HEAD --oneline` is empty, report that and **stop** (no PR needed).
2. **Uncommitted changes after Step 0**: If the tree is still dirty (e.g. user declined to commit or only partial staging), **warn** that whatever is **not** committed is **not** in the PR; either **stop** until clean or proceed reviewing/pushing only committed commits—do not silently ignore without stating this.
3. **Large diffs**: If the diff is huge, review by **commit** or by **top-level paths** (`apps/`, `libs/`) and summarize; still produce PR title and body.

---

## Step 0 — Commit working tree (if dirty)

Run this **before** Step 1 so the review and PR include local edits.

1. **Detect**: `git status --short` (or `git diff --quiet && git diff --cached --quiet` exit check). If there is nothing to commit (clean index and working tree), **skip** the rest of Step 0.
2. **Show scope**: Print `git status` (and optionally `git diff --stat`) so what will be included is obvious.
3. **Safety**: Do not stage or commit secrets or env files (e.g. `.env*`), credentials, or large generated artifacts unless the user explicitly wants them—**call out** suspicious paths and confirm if unclear.
4. **Stage**: `git add` the intended paths. Prefer explicit paths or user confirmation; use `git add -A` only when the user confirms **all** current changes belong in this PR.
5. **Commit message**: **Obtain from the user** a clear message (e.g. conventional commit style if the repo uses it). Do not invent a final message without user input unless they already provided one when invoking the command.
6. **Commit**: `git commit -m "<message>"`. If nothing is staged, **stop** and explain.
7. **Continue**: Proceed to Step 1. (Push still happens later in Step 4 when creating the PR, with user consent if needed.)

---

## Step 1 — Sync and scope

1. Run: `git fetch origin uat-placeos`
2. Record current branch: `git branch --show-current`
3. List commits in scope: `git log origin/uat-placeos..HEAD --oneline`
4. Inspect changes: `git diff origin/uat-placeos...HEAD` (three-dot)

---

## Step 2 — Code review

Follow **`.cursor/skills/review-changed-code/SKILL.md`** on the diff above.

Review priorities (in order):

1. Correctness / regressions  
2. Architecture and monorepo placement (app vs `libs/`)  
3. Angular anti-patterns  
4. RxJS / async safety  
5. Readability and maintainability  
6. Performance in templates or change-heavy flows  

For each changed area: intended behavior, edge cases, placement, naming, coupling, side effects.

**Output format (internal review vs what goes in the PR):**

- **Summary** (goes in PR under **Code review**): one short paragraph on overall quality and risk of this change set.  
- **Findings**: During review, note issues at any severity for your own traceability. **In the PR description, include a Findings list only for severity High** (each item: file/area, issue, why it matters, suggested fix). **Do not** put Critical / Medium / Low findings in the PR body—those stay in chat or internal notes only. If there are **no High** findings, **omit** the Findings subsection from the PR entirely (do not add an empty Findings section).  
- If review surfaces **Critical** issues, mention them in the **Summary** paragraph so they are visible in the PR even though they are not listed under Findings.

Do not pad with praise. Prefer fewer high-value findings.

### Decision gate — Critical / High findings

After completing the review, if **any Critical or High severity findings** were identified, **stop and present two options** to the user before continuing:

> **Code review found Critical/High severity issues:**
>
> [List each Critical/High finding: file, issue, brief description]
>
> **Choose how to proceed:**
> 1. **Fix first** — I will attempt to fix the issues now. After fixing, I will re-run the review on the updated diff and repeat this gate until no Critical/High findings remain (or you choose option 2).
> 2. **Proceed anyway** — Continue to PR creation with these findings noted in the PR body.

**Rules:**
- **Do not proceed to Step 3** until the user explicitly picks an option.
- If the user picks **Fix first**:
  1. Apply fixes for the identified Critical/High issues.
  2. Stage and commit the fixes (follow the same safety rules as Step 0 — obtain a commit message from the user or propose one for approval).
  3. Re-run the code review (Step 2) on the updated `origin/uat-placeos...HEAD` diff.
  4. If new Critical/High findings surface, present this gate again. Repeat until clean or the user picks **Proceed anyway**.
- If the user picks **Proceed anyway**: continue to Step 3 with the current findings intact.
- If there are **no Critical or High findings**, skip this gate silently and proceed to Step 3.

---

## Step 3 — Build PR title and body

**Title**: Use `$PR_TITLE` parsed in the Parameters section. If it is already short and imperative, use it as-is; otherwise refine it to conventional-commit style.

### Preparing the Overview content

The agent **always** produces a diff-inferred summary first: a tight, imperative description of *what changed* and *why*, inferred from commits and diff — like a polished commit subject + optional short clause, not marketing prose or a long story. Call this `$DIFF_SUMMARY`.

Then build the final Overview:

- **If `$DESCRIPTION_CLEAN` is available:** Merge `$DESCRIPTION_CLEAN` and `$DIFF_SUMMARY` into a single cohesive paragraph. Both sources contribute — the user's intent/context from the description and the concrete technical detail from the diff. After merging, **rephrase the combined result** once more into clear, simple, human-readable language. This merged and rephrased text is the Overview.
- **If `$DESCRIPTION_CLEAN` is NOT available:** Use `$DIFF_SUMMARY` as the Overview directly.

**Body** (Markdown), use this structure:

~~~markdown
## Overview
[Final Overview content prepared above.]

## Technical summary
- [Key areas: staff / concierge / shared libs]
- [Notable files or modules touched]

## Code review
**Summary**  
[Paste the Summary paragraph from Step 2.]

**Findings**  
[Only if there are High-severity items: bullet list. Otherwise omit this heading and everything under it—no "residual risks," "optional improvements," or filler.]

## How to test
[Concrete steps, or placeholder if unknown.]
~~~

---

## Step 4 — Create the PR

**Base branch:** `uat-placeos`  
**Head:** current branch on `origin`

1. Check if the branch exists on `origin` (`git push -u origin <branch>` if missing). **Ask user before pushing** if not already pushed.
2. Try automated creation:
   - Write the PR body to a temp file, then:
   - `gh pr create --base uat-placeos --head <current-branch> --title "<title>" --body-file <path-to-body.md>`
3. **If `gh` is unavailable, not authenticated, or the command fails**: Print the **exact title** and **full markdown body** for the user to paste into GitHub: **New pull request** → base `uat-placeos`, compare head = current branch.

4. On success, print the PR URL.

---

## Step 5 — Update JIRA ticket (only when `$JIRA_TICKET` is provided)

> **Gate:** If `$JIRA_TICKET` is empty or was not supplied, **skip Steps 5 and 6 entirely** and jump straight to **Done**.

### 5a — Authenticate Atlassian MCP (if needed)

If the Atlassian MCP server is not already authenticated, call `mcp_auth` on the appropriate server (`plugin-atlassian-atlassian` or `user-Atlassian-MCP-Server`) with empty arguments `{}` and wait for the user to complete authentication before proceeding.

### 5b — Fetch the JIRA ticket

Use the Atlassian MCP tool `getJiraIssue` (or equivalent) to fetch the ticket identified by `$JIRA_TICKET`.

- If the ticket does not exist or the fetch fails, **warn the user** and continue to **Done** (do not block the overall workflow).
- Record the ticket summary, status, **issue type**, and any relevant fields for use in the comment.
- Determine `$COMMENT_HEADING` based on the issue type:

| Issue type (case-insensitive) | `$COMMENT_HEADING` |
|-------------------------------|---------------------|
| Bug                           | **Issue Fixed** |
| Story                         | **Feature Implemented** |
| Any other type                | **Changes Submitted for Review** |

### 5c — Post a comment on the ticket

Use `addCommentToJiraIssue` to add a comment to `$JIRA_TICKET` with the following structure:

~~~markdown
## $COMMENT_HEADING

**PR Title:** $PR_TITLE
**PR Link:** [$PR_URL]($PR_URL)
**Branch:** $CURRENT_BRANCH → uat-placeos

### Overview
[Paste the same final merged Overview from the PR body (Step 3). This is the version that already blends `$DESCRIPTION_CLEAN` + `$DIFF_SUMMARY` and is rephrased into clear, human-readable language. If `$DESCRIPTION_CLEAN` was not available, paste the `$DIFF_SUMMARY` version used in the PR.]

### How to Test
[Paste the **How to test** section from the PR body (Step 3).]
~~~

Where `$PR_URL` is the pull request URL obtained in Step 4 (or the manually constructed GitHub URL if `gh` was unavailable). The PR link **must** use markdown link syntax `[text](url)` so JIRA renders it as a clickable hyperlink, not plain text.

### 5d — Transition ticket status to "Code Review"

Use `transitionJiraIssue` to move `$JIRA_TICKET` to the **"Code Review"** status (or the closest matching transition name, e.g. "Code review", "In Code Review", "Review").

- First call `getJiraIssueTransitions` for `$JIRA_TICKET` to list available transitions and pick the one whose name best matches "Code Review" (case-insensitive partial match).
- If no matching transition exists, **warn the user** that the status could not be updated and list the available transitions so they can do it manually.
- If the ticket is already in "Code Review" status, skip the transition and note that.

---

## Step 6 — JIRA confirmation (only when `$JIRA_TICKET` is provided)

Print a summary of the JIRA actions taken:

```
JIRA Ticket: $JIRA_TICKET
  ✓ Comment posted with PR overview, test steps, and link
  ✓ Status transitioned to "Code Review"
```

If any JIRA action failed, list what succeeded and what did not, with the reason.

---

## Done

Confirm: commits reviewed (range), review outcome, PR URL or paste-ready title/body, and (if `$JIRA_TICKET` was provided) JIRA update status.
