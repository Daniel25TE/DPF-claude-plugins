# log-time

Daily Jira time logger for the DPF team.

## What it does

Run `/log-time` and Claude will:

1. Show a summary of hours logged for the last 7 days
2. Ask which day you want to log time for
3. Fetch all your active tasks (To Do / In Progress / In Test)
4. Display each task with its original estimate and time already logged
5. Let you type in how much time to log per task (blank = skip)
6. Ask if you want to log time for additional days
7. Show a confirmation summary with per-day totals before submitting
8. Submit all worklogs to Jira at once

## Usage

```
/log-time
```

## Time format

When entering time, use any of these formats:

| Input    | Meaning        |
|----------|----------------|
| `1h`     | 1 hour         |
| `30m`    | 30 minutes     |
| `1h 30m` | 1 hour 30 min  |
| `1h30m`  | 1 hour 30 min  |
| `90m`    | 90 minutes     |

Leave a task blank to skip it — it won't be submitted.

## Installation

```bash
claude plugin install log-time@DPF-claude-plugins
```

## Requirements

- Atlassian MCP configured (for Jira access)
