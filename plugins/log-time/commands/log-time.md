---
user-invocable: false
description: >
  Daily Jira time logger. Shows a 7-day worklog history, lets the user pick a day,
  displays assigned tasks (To Do / In Progress / In Test) with estimate and logged time,
  collects time entries via plain text input, supports multi-day logging, shows a
  per-day confirmation summary, and submits worklogs to Jira.
allowed-tools:
  - Bash
  - mcp__atlassian__getAccessibleAtlassianResources
  - mcp__atlassian__atlassianUserInfo
  - mcp__atlassian__searchJiraIssuesUsingJql
  - mcp__atlassian__addWorklogToJiraIssue
---

## JIRA TIME LOGGER

You are helping the user log their daily work hours in Jira tasks. Follow each step in order.

---

### STEP 0 — Setup

Run `date '+%A %Y-%m-%d'` via Bash to get today's weekday name and ISO date.
**Use this output as the authoritative source for today's date and day name — never calculate the weekday manually from a date string.**

Call `getAccessibleAtlassianResources` to get the `cloudId`.
Call `atlassianUserInfo` to get the user's `accountId` and display name.

Store all three values — you will need them throughout.

---

### STEP 1 — Fetch 7-day worklog summary

Calculate the dates for the last 7 calendar days including today (today = day 7).

For each day, run this JQL to find issues where the user logged time on that specific date,
and include `"worklog"` in the fields array — this returns worklog entries inline, avoiding
a separate `getJiraIssue` call (which does NOT return worklog data through the MCP wrapper):
```
JQL:    worklogAuthor = currentUser() AND worklogDate = "YYYY-MM-DD"
fields: ["summary", "timespent", "worklog"]
```

Filter the returned `worklog.worklogs` array on each issue where:
- `author.accountId` matches the user's accountId
- The `started` field date matches the target day

Sum `timeSpentSeconds` for all matching worklogs per day.
Convert seconds to a readable format: use `Xh Ym` (e.g. `5h 30m`, `2h`, `45m`).
If no worklogs found for a day, show `── no entries`.

---

### STEP 2 — Display 7-day summary

Show the following formatted block. Replace placeholders with real dates and totals:

```
╔══════════════════════════════════════════════════════════════╗
║                   JIRA TIME LOGGER                           ║
╚══════════════════════════════════════════════════════════════╝

📅  YOUR LAST 7 DAYS

     #   Day              Date         Logged
     ────────────────────────────────────────────
     1.  [Day name]       [Mon DD]     [Xh Ym | ── no entries]
     2.  [Day name]       [Mon DD]     [Xh Ym | ── no entries]
     3.  [Day name]       [Mon DD]     [Xh Ym | ── no entries]
     4.  [Day name]       [Mon DD]     [Xh Ym | ── no entries]
     5.  [Day name]       [Mon DD]     [Xh Ym | ── no entries]
     6.  [Day name]       [Mon DD]     [Xh Ym | ── no entries]
     7.  Today ([Day])    [Mon DD]     [Xh Ym | ── no entries]
```

Then ask the user:
> "Which day do you want to log time for? Type a number (1–7):"

Wait for the user's response before continuing.

Keep track of which days the user has already logged entries for in this session
(mark them as `✓ entries added` in subsequent views of this summary).

---

### STEP 3 — Fetch assigned tasks

Run this JQL to get the user's active tasks:
```
assignee = currentUser() AND status in ("To Do", "In Progress", "In Test") ORDER BY updated DESC
```

Request these fields: `summary`, `status`, `timeoriginalestimate`, `timespent`, `timetracking`, `issuetype`.

Convert all time values from seconds to readable format (`Xh Ym`).
If `timeoriginalestimate` is null → show `── not set`.
If `timespent` is null or 0 → show `0h`.

---

### STEP 4 — Display task list and collect time entries

Show the selected day as a header. Then display one card per task. After all cards,
show a single input block for all tasks together.

```
📋  LOGGING TIME FOR: [Day name] – [Weekday, Month DD, YYYY]
─────────────────────────────────────────────────────────────────

  Task 1 of N
  ┌──────────────────────────────────────────────────────────────┐
  │  [KEY]  ·  [Status]                                          │
  │  [Full task summary — wrap at ~60 chars if needed]           │
  │                                                              │
  │  Estimate: [Xh Ym | ── not set]   │   Logged: [Xh Ym | 0h]  │
  └──────────────────────────────────────────────────────────────┘

  Task 2 of N
  ┌──────────────────────────────────────────────────────────────┐
  │  [KEY]  ·  [Status]                                          │
  │  [Full task summary]                                         │
  │                                                              │
  │  Estimate: [Xh Ym | ── not set]   │   Logged: [Xh Ym | 0h]  │
  └──────────────────────────────────────────────────────────────┘

  [... repeat for all tasks ...]

─────────────────────────────────────────────────────────────────
  Enter time for each task below.
  Leave a field blank to skip that task.
  Accepted formats: 1h · 30m · 1h 30m · 1h30m
─────────────────────────────────────────────────────────────────

  [KEY-1]:
  [KEY-2]:
  [KEY-N]:
```

Wait for the user to fill in all fields and submit their response.

---

### STEP 5 — Parse and validate entries

Parse each line from the user's response in the format `KEY: value`.

For each entry:
- If blank or missing → skip entirely, do not add to submission list
- If valid time format (`Xh`, `Xm`, `XhYm`, `Xh Ym`, `Xh30m`, etc.) → convert to seconds and store
- If invalid format → show only the invalid lines again and ask the user to re-enter those specific ones

Accepted patterns: any combination of hours (`h`) and/or minutes (`m`) with numeric values.
Examples of valid: `1h`, `30m`, `1h 30m`, `1h30m`, `90m`, `2h`, `0h 45m`.

Store the validated entries mapped to: `{ date, issueKey, timeSpentSeconds, timeSpent (readable) }`.

---

### STEP 6 — Ask about additional days

Use `AskUserQuestion` with:
- Question: "Do you want to log time for another day?"
- Options:
  - "Yes, log another day"
  - "No, proceed to submit"

If "Yes, log another day":
- Return to STEP 2, showing the 7-day summary again
- Mark days that already have entries in this session with `✓ entries added`
- Collect entries for the new selected day
- Accumulate all entries across days

If "No, proceed to submit":
- Continue to STEP 7

---

### STEP 7 — Confirmation summary

Display the full summary of what is about to be submitted.
Group entries by day. For each day show a block:

```
─────────────────────────────────────────────────────────────────
📋  REVIEW YOUR ENTRIES BEFORE SUBMITTING
─────────────────────────────────────────────────────────────────

  [Day name], [Month DD, YYYY]
  ┌──────────────────────────────────────────────────────────────┐
  │  · [KEY-1]  [task summary ~50 chars]         [time to log]   │
  │  · [KEY-2]  [task summary ~50 chars]         [time to log]   │
  │  · [KEY-N]  ...                              [time to log]   │
  │                                                              │
  │  Already logged on this day:   [Xh Ym | 0h]                 │
  │  Submitting now:               [Xh Ym]                       │
  │  Day total after submission:   [Xh Ym]                       │
  └──────────────────────────────────────────────────────────────┘

  [Repeat block for each day that has entries]

─────────────────────────────────────────────────────────────────
  Total to submit across all days:  [Xh Ym]
─────────────────────────────────────────────────────────────────
```

Then use `AskUserQuestion` with:
- Question: "Ready to submit these entries?"
- Options:
  - "Confirm and submit"
  - "Go back and edit"

If "Go back and edit": return to STEP 2.
If "Confirm and submit": continue to STEP 8.

---

### STEP 8 — Submit worklogs

For each validated entry (task/day pair with non-zero time):

Call `addWorklogToJiraIssue` with:
- `cloudId`: from STEP 0
- `issueIdOrKey`: the task key (e.g. `DPF-7778`)
- `timeSpent`: the readable time string (e.g. `1h 30m`)
- `started`: the selected day at 9:00 AM in ISO 8601 format: `YYYY-MM-DDT09:00:00.000+0000`

Submit all entries sequentially. If any call fails, report which one failed and continue submitting the rest.

---

### STEP 9 — Done

Display the success message:

```
✅  Time logged successfully!

  Submitted entries:
  ─────────────────────────────────────────────
  [KEY-1]  ·  [Day name, Mon DD]  ·  [time]
  [KEY-2]  ·  [Day name, Mon DD]  ·  [time]
  [KEY-N]  ·  ...
  ─────────────────────────────────────────────
  Total submitted:  [Xh Ym]
```

If any entries failed to submit, list them separately with an error note so the user can retry manually.
