---
name: docs-action-comments
description: Fetch unresolved PR/MR review comments and interactively action them on the local repo. Checks out the PR branch locally when a URL is provided. When invoked without arguments, uses AskUserQuestion to gather inputs. Works with both GitHub PRs and GitLab MRs.
argument-hint: "[<url>] [--include-resolved]"
allowed-tools: Read, Write, Glob, Grep, Edit, Bash, Skill, AskUserQuestion
---

# Action Review Comments

Fetch unresolved review comments from a GitHub PR or GitLab MR and interactively walk through them, applying edits to local files.

## Parse arguments

- `$1` — PR/MR URL (optional)
- `--include-resolved` — Include resolved comments alongside unresolved ones

## Determine mode

**Direct mode**: If a PR/MR URL is present in args, skip to Step 1.

**Interactive mode**: If no arguments are provided, use AskUserQuestion to gather inputs.

---

## Interactive mode — gather inputs

### Question 1: PR source — call AskUserQuestion

You MUST call the AskUserQuestion tool now. Do not skip this.

**Where are the review comments?**

| Option | Description |
|--------|-------------|
| Auto-detect from current branch | Detect the PR/MR associated with the current Git branch |
| Enter a PR/MR URL | Provide a GitHub PR or GitLab MR URL |

Wait for the answer before proceeding.

- If **"Auto-detect from current branch"**: set `PR_SOURCE=detect` and proceed to Question 2.
- If **"Enter a PR/MR URL"**: call AskUserQuestion with `textInput: true`:

  > Enter the PR/MR URL (e.g., https://github.com/org/repo/pull/123):

  Set `PR_URL` to the answer and proceed to Question 2.

### Question 2: Comment scope — call AskUserQuestion

**Which comments should be included?**

| Option | Description |
|--------|-------------|
| Unresolved only (default) | Only show comments that have not been resolved |
| All comments | Include both resolved and unresolved comments |

- If **"All comments"**: set `INCLUDE_RESOLVED=true`.
- Otherwise: set `INCLUDE_RESOLVED=false`.

Proceed to Step 1.

---

## Step 1: Resolve PR/MR URL

If `PR_SOURCE=detect` or no URL was provided:

```bash
PR_URL=$(python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py detect 2>/dev/null)
```

If detection fails, stop with:

> Could not detect a PR/MR for the current branch. Please provide a URL and try again.

## Step 2: Get PR info and check out the branch locally

Fetch PR metadata to determine the source branch:

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py info "${PR_URL}" --json
```

From the JSON output, extract:
- `head_ref` — the source branch name
- `base_ref` — the target branch name
- `repo_url` — the repository URL (for remote setup if needed)
- `title` — PR title (for display)

### Check out the PR branch

Check whether the current branch matches `head_ref`:

```bash
CURRENT_BRANCH=$(git branch --show-current)
```

**If already on the correct branch**: proceed to Step 3.

**If on a different branch**:

1. Check for uncommitted changes:
   ```bash
   git status --porcelain
   ```
   If there are uncommitted changes, stop with:
   > You have uncommitted changes on `{CURRENT_BRANCH}`. Please commit or stash them before switching branches.

2. Fetch and check out the PR branch:
   ```bash
   git fetch origin "${HEAD_REF}"
   git checkout "${HEAD_REF}"
   ```

   If the branch does not exist locally, create a tracking branch:
   ```bash
   git checkout -b "${HEAD_REF}" "origin/${HEAD_REF}"
   ```

Report to the user:

> Checked out branch `{HEAD_REF}` for PR: {title}

## Step 3: Fetch review comments

```bash
python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py comments "${PR_URL}" --json
```

Add `--include-resolved` if `INCLUDE_RESOLVED=true`.

The script automatically filters bot comments, resolved threads (unless `--include-resolved`), and returns top-level comments with: `id`, `path`, `line`, `body`, `author`, `resolved`.

If no comments are returned, report:

> No unresolved review comments found on this PR/MR.

And stop.

## Step 4: Categorize comments

Before presenting comments, categorize each one:

| Category | Criteria | Action |
|----------|----------|--------|
| **Required** | Style violations, technical errors, broken examples | Must fix |
| **Suggestion** | Wording improvements, reorganization | User discretion |
| **Question** | Requests for clarification, questions from reviewer | Needs discussion — present but do not auto-suggest a fix |
| **Outdated** | Already addressed by subsequent commits | Skip automatically |

For **Outdated** detection: read the file at the comment's `path` and `line`. If the content no longer matches what the comment references (the reviewer's quoted text or the line context), mark as outdated.

## Step 5: Process each comment interactively

For each non-outdated comment, present:

```markdown
## Comment {N} of {total} from @{author} on `{path}:{line}` [{category}]

> {comment_body}

### Current content (local file)
{relevant lines from the local file around the comment's line}

### Suggested change
{your analysis and proposed edit}
```

Call AskUserQuestion with these options:

| Option | Description |
|--------|-------------|
| Apply | Apply the suggested change |
| Edit | Apply with modifications — ask for user's preferred text |
| Skip | Skip this comment |
| View context | Show more surrounding lines, then re-ask |

**When Apply is selected**: Read the target file, apply the edit using Edit tool, confirm the change was applied, move to next comment.

**When Edit is selected**: Call AskUserQuestion with `textInput: true`:

> Enter the text you'd like to use instead:

Apply the user's text using Edit tool, confirm, move to next.

**When View context is selected**: Read 20 lines before and after the comment's line from the local file, display them, then re-present the same options.

**When Skip is selected**: Move to next comment.

## Step 6: Summary

After all comments are processed, present:

```markdown
## Action Comments Summary

**PR/MR**: {PR_URL}
**Branch**: {HEAD_REF}

| Metric | Count |
|--------|-------|
| Total comments | X |
| Applied | Y |
| Edited | Z |
| Skipped | S |
| Outdated (auto-skipped) | O |
| Bot comments (filtered) | B |

### Changes applied

1. `{path}:{line}` — {brief description of change}
2. ...

### Comments skipped

1. `{path}:{line}` — @{author}: "{truncated comment}" — Reason: {user skipped / outdated}
```

If any changes were applied, remind the user:

> Changes have been applied to your local files on branch `{HEAD_REF}`. Review them with `git diff` and commit when ready.

---

## Notes

- Always use `python3 ${CLAUDE_PLUGIN_ROOT}/skills/git-pr-reader/scripts/git_pr_reader.py` for all Git platform interactions
- Work on local files only — this skill never pushes to remote
- If a comment references a file that does not exist locally, report it and skip
- For **Question** category comments, present them for awareness but do not suggest code changes — suggest the user reply on the PR/MR directly
