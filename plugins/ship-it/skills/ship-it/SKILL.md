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
---

# Ship It — DPF PR Workflow

Execute all steps automatically without asking for permission to proceed between steps.

---

## Step 0: Get Jira Task ID

Ask the user for the Jira task ID (e.g., `DPF-6944`). Store it — it will be used in the PR title.

---

## Step 1: Branch Setup

Check the current branch with `git branch --show-current`.

**If on `master`:**
1. Stash changes: `git stash`
2. Pull latest: `git pull origin master`
3. Create a new branch named after the work being done (kebab-case, descriptive)
4. Restore: `git stash pop`

**If already on a feature branch:**
- No branch changes needed. Proceed to Step 2.

---

## Step 2: Analyze Changes & Compose Commit Message

Run `git diff` and `git status` to understand what changed.

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

**Only if on a feature branch (not master):**

1. Fetch latest master: `git fetch origin master`
2. Rebase on top of it: `git rebase origin/master`

If there are rebase conflicts, pause and show the user which files conflict. Do not auto-resolve conflicts.

---

## Step 5: Push

1. Attempt push: `git push -u origin <branch-name>`
2. If rejected due to rebase history rewrite, force push safely:
   `git push --force-with-lease origin <branch-name>`

Always use `--force-with-lease`, never `--force`.

---

## Step 6: Create PR

Create the PR using `gh pr create` with:
- **Title format:** `[JIRA-KEY] Short title` — e.g., `[DPF-6944] Fix account selection for pending accounts`
- **Body:** Include a brief summary of the changes

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

Post to the team Slack channel `C099WMJQYRY`:

```
Hello team, this PR is ready for review: <PR_URL>
```

Use the `mcp__slack__slack_post_message` tool with:
- `channel_id`: `C099WMJQYRY`
- `text`: the message above with the real PR URL

---

## Step 8: Confirm Completion

Report to the user:
- Branch name
- Commit message title
- PR URL
- Confirmation that Slack was notified
