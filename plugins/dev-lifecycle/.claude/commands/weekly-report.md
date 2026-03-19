---
description: Generate weekly status report compiling daily reports from the current work week (Sunday through today). Run any day.
argument-hint: [YYYY-MM-DD]
---

Generate a weekly status report that compiles the current work week. The week starts on the most recent Sunday and ends on the date provided (or today if none given).

## Step 0: Load Configuration

1. Read `.claude/dev-lifecycle.json` from the project working directory
2. If the file doesn't exist: prompt for Jira project key and URL, auto-detect git branches, create config
3. If file exists, check `configVersion` and migrate if needed
4. Verify required tools: Atlassian MCP (warn and stop if missing), `gh auth status` (warn and stop — required for this command)
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, etc.

## Steps

1. **Determine the week range and confirm with user** — calculate the range from the most recent Sunday through today (or the provided date). Present the range to the user and ask them to confirm before proceeding.

2. **Gather data for the week** — run these in parallel:
   - `git log --all --oneline --since="<sunday> 00:00" --until="<day-after-end> 00:00"` for all commits in the range
   - `git log --all --oneline --since="<sunday> 00:00" --until="<day-after-end> 00:00" --format="%s"` to extract all ticket numbers matching the pattern `{PROJECT_KEY}-XXX`
   - `gh pr list --author=@me --state=all --search="created:>=$SUNDAY"` for PRs created this week
   - `gh pr list --author=@me --state=merged --search="merged:>=$SUNDAY"` for PRs merged this week
   - `gh pr list --author=@me --state=open` for still-open PRs

3. **Build a day-by-day breakdown** — for each day (Mon-Fri), note which tickets had activity (commits, PRs opened/merged). This is used internally to build the summary, not included in output.

4. **Filter to assigned tickets** — for each `{PROJECT_KEY}-XXX` ticket found in step 2:
   - Resolve the current user's email via `git config user.email`
   - Query Jira to check assignee: use `mcp__atlassian__jira_search` with JQL `project = {PROJECT_KEY} AND key in ({PROJECT_KEY}-XXX, {PROJECT_KEY}-YYY, ...) AND assignee = "<user-email>"` (fields: `summary,status,assignee`)
   - **Only include tickets assigned to the user** — discard any tickets assigned to others, even if they have commits from this repo during the week

5. **Categorize tickets across the whole week**:
   - **DONE**: All tickets whose work was fully completed and merged this week
   - **IN-REVIEW**: Tickets with open PRs at end of week
   - **IN-PROGRESS**: Tickets with commits this week but no PR yet

6. **Format the report** exactly like this:

```
Weekly Report: <Sunday date> - <end date>

DONE
{PROJECT_KEY}-XXX: Brief description of what was completed
{PROJECT_KEY}-YYY: Brief description

IN-REVIEW
{PROJECT_KEY}-XXX: Brief description (PR #NNN)

IN-PROGRESS
{PROJECT_KEY}-XXX: Brief description of current state

ISSUE
Description of any blockers or problems encountered during the week

NEXT PLAN
Plans for next week based on current state
```

## Rules

- Each ticket appears in exactly ONE section (highest status wins: DONE > IN-REVIEW > IN-PROGRESS)
- Descriptions should be concise but can be slightly more detailed than daily reports since they summarize the week's effort
- For DONE/IN-REVIEW/IN-PROGRESS: always lead with the ticket number
- For ISSUE and NEXT PLAN: use plain descriptions
- If a section has no items, include a single `None` entry
- Do NOT prefix items with bullets, dashes, or any list markers — each item is a plain line
- After presenting the report, remind the user: **"Tip: After pasting into Slack, select the list items and apply bulleted list formatting."**
- Ask the user if anything needs adjustment — they may want to add context about issues or plans not visible from git
