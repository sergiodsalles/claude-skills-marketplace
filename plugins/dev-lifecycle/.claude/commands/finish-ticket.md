---
description: Complete the workflow — generate docs, update Confluence, final Jira update, close state
---

# Finish Ticket: $ARGUMENTS

You are completing the development workflow for Jira ticket **$ARGUMENTS**. Follow each step in order.

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
   - Check Atlassian MCP is available. If not: **warn and stop** — required for all ticket commands.
   - Check `gh auth status`. If not authenticated: **warn but continue** — only required for PR and report commands.
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, `{MAIN_BRANCH}`, `{DEV_BRANCH}`, `{CONFLUENCE_SPACE}`, `{CONFLUENCE_PARENT}`, `{TRANSITIONS}`, `{SPECS_DIR}`, `{FEATURES_DIR}`

---

## Step 1: Load State & Show Progress

1. Load state file: `.claude/tickets/$ARGUMENTS.json`
2. Display the current workflow progress:
   ```
   Ticket $ARGUMENTS — Workflow Progress

   Steps:
     {checkmark} Jira ticket read
     {checkmark} Spec created
     {checkmark} Branch created
     {checkmark} Jira → In Progress
     {checkmark} Implementation
     {checkmark/pending} Code review
     {checkmark/pending} PR created
     {checkmark/pending} Documentation
     {pending} Confluence updated
     {pending} Jira final update
   ```
3. Identify any pending steps before proceeding

---

## Step 2: Complete Pending Steps

**Documentation check:**

If `steps.documentation.done` is `false` or absent in the state file:
- Ask: "Feature documentation hasn't been created yet. Create it now?"
- If yes, follow the `/document-feature` workflow inline (see that command for the full process)
- If no, note it and continue

**Code review check:**

If `steps.code_review.done` is `false` or absent:
- Warn: "WARNING: Code review was not completed for this ticket."

---

## Step 3: Update Confluence

**Idempotency check:** If `steps.confluence_updated.done` is `true` in the state file, display the existing Confluence URL and ask: "Confluence was already updated at {timestamp}. Re-run this step or skip?"

**Confluence availability check:**

- If `{CONFLUENCE_SPACE}` is `null` or empty in config:
  - Ask: "No Confluence space is configured. Would you like to provide one now to update Confluence?"
  - If the user provides a space key, use it for this step (and optionally offer to save it to config)
  - If the user declines, **skip this entire step** and proceed to Step 4
- If `{CONFLUENCE_SPACE}` is set, proceed normally

### For New Features (Story, Task):

1. Search for existing Confluence pages related to this feature area:
   ```
   mcp__atlassian__confluence_search({ cql: "space = \"{CONFLUENCE_SPACE}\" AND text ~ \"$ARGUMENTS\"" })
   ```

2. Create a new Confluence page:
   ```
   mcp__atlassian__confluence_create_page({
     spaceKey: "{CONFLUENCE_SPACE}",
     title: "$ARGUMENTS — {Feature Name}",
     parentId: "{CONFLUENCE_PARENT}",   // omit if {CONFLUENCE_PARENT} is null
     body: "..."  // Formatted version of the feature doc
   })
   ```

   - If `{CONFLUENCE_PARENT}` is `null`, create at the space root (omit the `parentId` field)
   - Page content should include:
     - Ticket link (`{JIRA_URL}/browse/$ARGUMENTS`)
     - Summary of the feature
     - Technical details (from the feature doc if it exists)
     - PR link (from state)
     - Date completed

### For Bug Fixes / Enhancements:

1. Search for the existing feature's Confluence page:
   ```
   mcp__atlassian__confluence_search({ cql: "space = \"{CONFLUENCE_SPACE}\" AND text ~ \"{feature keywords}\"" })
   ```
2. If found, update it with a changelog entry:
   ```
   mcp__atlassian__confluence_update_page({
     pageId: "...",
     // Append a "Changelog" section at the bottom
   })
   ```
   Changelog entry format:
   ```
   ### {YYYY-MM-DD} — $ARGUMENTS
   - **Type:** Bug fix / Enhancement
   - **Summary:** {description}
   - **PR:** {link}
   ```

3. If no existing page found, ask: "No existing Confluence page found for this feature area. Create a new one?"

---

## Step 4: Final Jira Comment

**Idempotency check:** Add a comment only if a "workflow complete" comment has not already been posted (check the state history for a `workflow_completed` entry).

Add a comprehensive comment to the Jira ticket:

```
mcp__atlassian__jira_add_comment({ issueIdOrKey: "$ARGUMENTS", comment: "..." })
```

Comment content:
```
Development Complete — $ARGUMENTS

Spec: {spec path or "N/A"}
Branch: {branch name}
PR: {PR URL or "N/A"}
Docs: {doc paths or "N/A"}
Confluence: {Confluence URL or "N/A" if skipped}

Summary of changes:
{2-3 sentence summary of what was implemented}

Deployment notes:
- Database migration: {Yes — {migration name} / No}
- Environment variables: {Any new env vars, or "None"}
- Dependencies: {Any new packages, or "None"}
```

---

## Step 5: Transition Jira

**Idempotency check:** If `steps.jira_final_transition.done` is `true` in the state file, skip with a note.

1. Get available transitions:
   ```
   mcp__atlassian__jira_get_transitions({ issueIdOrKey: "$ARGUMENTS" })
   ```
2. Determine the correct transition based on merge target:
   - If the PR was merged to `{DEV_BRANCH}`: fuzzy-match the returned transition names against the `{TRANSITIONS}` config value for `"on_dev"` (e.g., "On Staging", "On Dev", "Ready for QA"). Pick the closest match.
   - If the PR was merged to `{MAIN_BRANCH}`: fuzzy-match against the `{TRANSITIONS}` config value for `"done"` (e.g., "Done", "Released", "Closed"). Pick the closest match.
   - If the PR has not been merged yet: fuzzy-match against `{TRANSITIONS}` config for `"in_review"` (e.g., "In Review").
   - If no match is found for any case, display available transitions and ask the developer to select one.
   - **Never hardcode a transition ID.**
3. Execute the transition:
   ```
   mcp__atlassian__jira_transition_issue({ issueIdOrKey: "$ARGUMENTS", transitionId: "{matched-id}" })
   ```

---

## Step 6: Close State

Update `.claude/tickets/$ARGUMENTS.json`:
- Set `steps.confluence_updated.done = true` with current timestamp (even if Confluence was skipped — set a `skipped: true` flag instead)
- Set `steps.jira_final_transition.done = true` with current timestamp
- Set `confluence.pageId` and `confluence.url` (or `null` if skipped)
- Add final history entry: `{ "action": "workflow_completed", "at": "{ISO timestamp}", "details": "All steps complete" }`

Display:
```
Ticket $ARGUMENTS — Workflow Complete!

All steps:
  done  Jira ticket read
  done  Spec created
  done  Branch created
  done  Jira → In Progress
  done  Implementation
  done  Code review
  done  PR created
  done  Documentation
  done  Confluence updated   (or "skipped" if Confluence was declined)
  done  Jira updated

PR: {URL or "N/A"}
Confluence: {URL or "skipped"}
```
