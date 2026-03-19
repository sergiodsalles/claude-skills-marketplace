---
description: Generate end-of-day status report with DONE, IN-REVIEW, IN-PROGRESS, ISSUE, and NEXT PLAN sections based on git activity and Jira tickets
argument-hint: [YYYY-MM-DD]
---

# Daily Report: $ARGUMENTS

Generate an end-of-day status report. If a date argument is provided, use that date; otherwise use today's date.

---

## Step 0: Load Configuration

1. Read `.claude/dev-lifecycle.json` from the project working directory
2. If the file doesn't exist:
   - Prompt the user for required values:
     - **Jira project key** (e.g., "SP", "PROJ")
     - **Jira URL** (e.g., "https://mycompany.atlassian.net")
   - Auto-detect optional values:
     - `git.mainBranch`: from `git remote show origin | grep HEAD` (fallback: `"main"`)
     - `git.developBranch`: check branches in priority order `develop` > `staging` > `stg` > `dev`, first match wins. If multiple exist, prompt user to confirm. (fallback: same as mainBranch)
   - Create the config file with detected/prompted values and defaults for the rest
3. If the file exists, check `configVersion`:
   - If missing or less than current version (1), offer to migrate by backfilling new fields with defaults while preserving existing values
4. Verify required tools:
   - Check Atlassian MCP is available. If not: **warn and stop** — required for this command.
   - Check `gh auth status`. If not authenticated: **warn and stop** — required for this command.
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, etc.

---

## Step 1: Gather Data

Run these in parallel:

- `git log --all --oneline --since="<date> 00:00" --until="<next-day> 00:00"` to find commits for the target date
- `git log --all --oneline --since="<date> 00:00" --until="<next-day> 00:00" --format="%s"` to extract ticket numbers (`{PROJECT_KEY}-XXX` pattern)
- `git diff --stat HEAD@{<date>}..HEAD@{<next-day>}` for a sense of scope
- Check for any open PRs via `gh pr list --author=@me --state=open`
- Check current branch and uncommitted work via `git status` and `git branch --show-current`

## Step 2: Filter to Assigned Tickets

For each `{PROJECT_KEY}-XXX` ticket found in Step 1:

- Resolve the current user's email via `git config user.email`
- Query Jira to check assignee:
  ```
  mcp__atlassian__jira_search({
    jql: "project = {PROJECT_KEY} AND key in ({PROJECT_KEY}-XXX, {PROJECT_KEY}-YYY, ...) AND assignee = \"<user-email>\"",
    fields: ["summary", "status", "assignee"]
  })
  ```
- **Only include tickets assigned to the user** — discard any tickets assigned to others, even if they have commits from this repo on the target date

## Step 3: Categorize Tickets

For each assigned `{PROJECT_KEY}-XXX` ticket:

- **DONE**: Tickets whose PRs were merged on this date, or whose work was completed and pushed
- **IN-REVIEW**: Tickets with open PRs (found via `gh pr list`)
- **IN-PROGRESS**: Tickets with commits today but no PR yet, or the current working branch

## Step 4: Identify Issues

Look for:

- Failed CI checks on open PRs
- Reverted commits
- Bug-fix commits that suggest problems encountered
- Any blockers mentioned in commit messages

## Step 5: Infer Next Plan

Based on:

- Current working branch (likely continues tomorrow)
- Open PRs awaiting review
- Any TODO patterns in recent commits

## Step 6: Format the Report

Format the report exactly like this:

```
DONE
{PROJECT_KEY}-XXX: Brief description of what was completed
{PROJECT_KEY}-YYY: Brief description

IN-REVIEW
{PROJECT_KEY}-XXX: Brief description (PR #NNN)

IN-PROGRESS
{PROJECT_KEY}-XXX: Brief description of current state

ISSUE
Description of any blockers or problems encountered

NEXT PLAN
Continue work on {PROJECT_KEY}-XXX
Address review feedback on {PROJECT_KEY}-YYY
```

## Rules

- Each ticket appears in exactly ONE section (highest status wins: DONE > IN-REVIEW > IN-PROGRESS)
- Descriptions should be concise (one line each)
- For DONE/IN-REVIEW/IN-PROGRESS: always lead with the ticket number
- For ISSUE and NEXT PLAN: use plain descriptions (no ticket prefix required, but include ticket numbers if relevant)
- If a section has no items, include a single `None` entry
- Do NOT prefix items with bullets, dashes, or any list markers — each item is a plain line
- After presenting the report, remind the user: **"Tip: After pasting into Slack, select the list items and apply bulleted list formatting."**
- Ask the user if anything needs adjustment before finalizing — they may want to add issues or next plan items that aren't visible from git
