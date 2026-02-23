# ship-it

Full PR workflow for the DPF team.

## What it does

Run `/ship-it` and Claude will automatically:

1. Ask for your Jira task ID (e.g. `DPF-6944`)
2. Handle branch setup (even if you forgot to branch off master)
3. Analyze your changes and write a commit message
4. Commit, rebase on latest master, and push
5. Open a PR with `[JIRA-KEY]` in the title (auto-links in Jira)
6. Post the PR link to the team Slack channel

## Usage

```
/ship-it
```

That's it. Claude handles the rest.

## Installation

```bash
claude plugin install ship-it@DPF-claude-plugins
```

## Requirements

- `gh` CLI authenticated with GitHub
- Slack MCP configured (for Slack notifications)
- Atlassian MCP configured (optional, for Jira lookups)
