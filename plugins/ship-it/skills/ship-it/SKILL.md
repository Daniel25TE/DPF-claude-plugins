---
user-invocable: false
description: >
  Full PR workflow for the DPF team. Handles branch setup, commit,
  rebase on master, push, PR creation with Jira key in title,
  and Slack notification to the team channel.
allowed-tools:
  - Bash(git *)
  - Bash(gh *)
  - mcp__slack__slack_post_message
  - mcp__atlassian__getJiraIssue
  - mcp__atlassian__getAccessibleAtlassianResources
  - mcp__atlassian__addWorklogToJiraIssue
---

# Ship It — DPF PR Workflow

Execute all steps automatically. Only pause when user input is explicitly required (Step 0 and Step 8).

---

## Pre-check: Verify There Is Work to Ship

Before doing anything else, confirm there is actually something to ship.

Run both of these commands:
- `git status --porcelain` — checks for uncommitted changes (staged or unstaged)
- `git log origin/master..HEAD --oneline` — checks for commits on this branch not yet in master

If **both return empty**, stop immediately and tell the user:

```
Nothing to ship. There are no uncommitted changes and no commits ahead of master on this branch.
Make sure you have work in progress before running /ship-it.
```

If either has output, proceed to Step 0.

---

## Step 0: Get Jira Task ID

Ask the user for the Jira task ID (e.g., `DPF-6944`). Store it — it will be used in the PR title and Jira worklog.

If the user does not provide one, continue without it. Steps that depend on a Jira ID will be skipped or adapted.

---

## Step 1: Branch Setup

Check the current branch with `git branch --show-current`.

There are only two valid starting scenarios:

**Scenario A — Starting from master (work done but not yet on a branch):**
The user is on master with uncommitted changes ready to be shipped.
1. Check for uncommitted changes: `git status --porcelain`
   - If changes exist: stash them with `git stash`
   - If no changes: skip stash
2. Pull latest: `git pull origin master`
3. Create a new branch: run `git diff --stat HEAD` and `git status` to analyze the files that were changed. Infer a short, descriptive kebab-case branch name from what was actually changed — not the Jira task title. Focus on the nature of the change and the files or modules involved.
   - Example: changes in account selection logic → `fix-account-selection`
   - Example: new validation in the payment flow → `add-payment-flow-validation`
   - Example: UI updates across multiple components → `update-dashboard-ui`
4. Restore stashed changes (only if stash was done in step 1): `git stash pop`
5. **Remember: branch was freshly created from master → skip rebase in Step 4.**

**Scenario B — Already on a feature branch (work in progress or completed):**
The user already created a branch and has been working on it.
- No branch changes needed. Proceed to Step 2.
- **Remember: branch existed before this workflow → rebase is required in Step 4.**

> Any other starting point (stale/unrelated branch, detached HEAD, etc.) is not supported.
> If this is detected, warn the user and stop.

---

## Step 2: Analyze Changes & Compose Commit Message

Run `git diff HEAD` to see all changes (both staged and unstaged) and `git status` to understand what files are involved.

Compose a commit message in this exact format:
- **Line 1:** Short title (imperative, under 72 chars)
- **Line 2:** Empty
- **Lines 3+:** 2–4 line description of what and why

Example:
```
Add tooltip to expense report field

Added a tooltip component to the expense report field to provide
users with additional context about the expected input format.
This improves user experience and reduces form submission errors.
```

---

## Step 3: Stage and Commit

1. Stage all relevant changed files: `git add <files>`
2. Commit using a HEREDOC to preserve formatting:

```bash
git commit -m "$(cat <<'EOF'
<title>

<description>
EOF
)"
```

---

## Step 4: Sync with Latest Master

**Only if Scenario B from Step 1 (branch already existed before this workflow):**

The branch was created some time ago and master may have moved forward — rebase is needed to keep history clean before opening the PR.

1. Fetch latest master: `git fetch origin master`
2. Rebase on top of it: `git rebase origin/master`

If there are rebase conflicts, pause and tell the user which files conflict. Do not auto-resolve conflicts.
Once the user manually resolves the conflicts, they should run `git rebase --continue`.
After `git rebase --continue` succeeds, proceed to Step 5.

**Skip this step entirely if Scenario A from Step 1 (branch was just created from a freshly pulled master).**
The branch already contains the latest master — no rebase needed.

---

## Step 5: Push

1. Attempt push: `git push -u origin <branch-name>`
2. If rejected due to rebase history rewrite, force push safely:
   `git push --force-with-lease origin <branch-name>`

Always use `--force-with-lease`, never `--force`.

---

## Step 6: Create PR

First, check if a PR already exists for this branch:
```bash
gh pr list --head <branch-name> --state open
```
- If a PR already exists: capture its URL, skip `gh pr create`, and use the existing PR URL for the remaining steps.
- If no PR exists: create one using `gh pr create`.

**Title format:**
- If a Jira ID was provided: `[JIRA-KEY] Short title` — e.g., `[DPF-6944] Fix account selection for pending accounts`
- If no Jira ID was provided: use a plain descriptive title — e.g., `Fix account selection for pending accounts`

**Body:** Include a brief summary of the changes. Omit the Jira section if no Jira ID was provided.

Example:
```bash
gh pr create --title "[DPF-6944] Fix account selection for pending accounts" --body "$(cat <<'EOF'
## Summary
- Fixed account selection logic for accounts in pending status
- Updated validation to handle edge cases in the approval flow

## Jira
[DPF-6944](https://familysearch.atlassian.net/browse/DPF-6944)
EOF
)"
```

Capture the PR URL from the output.

---

## Step 7: Post to Slack

Post to the team Slack channel `C099WMJQYRY`.

Write a short, personalized message (1–2 lines) that describes what this PR actually does — based on the commit message and changed files analyzed in previous steps. Do not use a generic template. Make it specific enough that the team understands the context without clicking the link.

Format:
```
Hey team, this PR [1–2 line description of what was done and why it matters]. Can you review it when you have a minute? <PR_URL>
```

Examples:
```
Hey team, this PR fixes account selection for users in pending status and closes an edge case in the approval flow that was causing incorrect rejections. Can you review it when you have a minute? https://github.com/...
```
```
Hey team, this PR adds a tooltip to the expense report field to guide users on the expected input format, reducing form submission errors. Can you review it when you have a minute? https://github.com/...
```

Use the `mcp__slack__slack_post_message` tool with:
- `channel_id`: `C099WMJQYRY`
- `text`: the personalized message with the real PR URL

---

## Step 8: Log Time in Jira

**Skip this step if no Jira task ID was provided.**

1. Use `mcp__atlassian__getAccessibleAtlassianResources` to get the `cloudId`.
2. Use `mcp__atlassian__getJiraIssue` with the Jira task ID to retrieve:
   - `timeoriginalestimate` (original estimate in seconds)
   - `timespent` (time already logged in seconds)
3. Analyze the git log to estimate time worked on this PR:
   - Run `git log --format="%ai" origin/master..HEAD` to get timestamps of commits on this branch only
   - Calculate the span between the earliest and latest commit
   - If only one commit exists, use a heuristic based on diff size: small change (<50 lines) = 30m, medium (50–200 lines) = 1h, large (>200 lines) = 2h
4. Convert all times from seconds to a human-readable format (e.g., `2h 30m`).
5. Display the following to the user:

```
📋 Jira Time Tracking for [JIRA-KEY]:
- Original estimated time for this task: Xh
- Time already logged:                   Xh Xm
- Suggested time to log for this PR:     Xh Xm

Would you like to accept the suggested time, or enter a specific time?
(Type "yes" to accept, or provide a value like "2h", "45m", "1h 30m")
```

6. Wait for the user's response:
   - If "yes" or confirmed → use the suggested time
   - If a specific time is given → use that value
7. Log the time using `mcp__atlassian__addWorklogToJiraIssue` with the accepted time and the Jira task ID.

---

## Step 9: Confirm Completion

Report to the user:
- Branch name
- Commit message title
- PR URL
- Confirmation that Slack was notified
- Confirmation that time was logged in Jira (if applicable), including the amount logged
