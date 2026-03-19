---
description: Create feature documentation from the implementation
---

# Document Feature: $ARGUMENTS

You are creating feature documentation for Jira ticket **$ARGUMENTS**.

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
   - Check Atlassian MCP is available. If not: **warn and stop**.
   - Check `gh auth status`. If not authenticated: **warn but continue**.
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, `{MAIN_BRANCH}`, `{DEV_BRANCH}`, `{BRANCH_PREFIXES}`, `{SPECS_DIR}`, `{FEATURES_DIR}`, `{SPEC_TEMPLATE}`, `{FEATURE_TEMPLATE}`, `{CONFLUENCE_SPACE}`, `{CONFLUENCE_PARENT}`, `{TRANSITIONS}`

---

## Step 1: Load Context

Check state: if `.claude/tickets/$ARGUMENTS.json` exists and `steps.documentation.done` is `true`, show the existing doc path and ask: "Documentation already created at {docPath}. Re-generate it or skip?"

Otherwise, load available context:

1. Load state file: `.claude/tickets/$ARGUMENTS.json` (if it exists)
2. If a spec path is recorded in state, read it for context
3. Get the diff to understand what was implemented:
   ```bash
   git diff {DEV_BRANCH}...HEAD --stat
   ```

If no state file exists:
- Try to find the branch: `git branch -a | grep $ARGUMENTS`
- Read the Jira ticket for context using `mcp__atlassian__jira_get_issue({ issueIdOrKey: "$ARGUMENTS" })`
- Proceed with available information

---

## Step 2: Analyze Implementation

1. Read the changed files to understand the implementation
2. Identify:
   - New API endpoints and their contracts (request/response shapes)
   - New or modified database models and migrations
   - New components and their purpose
   - New background jobs or scheduled tasks
   - Configuration changes
   - New environment variables
   - Security or permission changes
   - Storage or file handling changes

---

## Step 3: Create Feature Documentation

1. Check if `{FEATURE_TEMPLATE}` exists:
   - If it exists, read it and use it as the structure for the documentation
   - If it does not exist, use the generic feature doc structure below
2. Write documentation to: `{FEATURES_DIR}/{YYYY-MM-DD}-$ARGUMENTS-{feature-name}.md`
3. Fill in ALL sections with actual implementation details:
   - Use real file paths from the codebase
   - Include actual API contracts (request/response shapes)
   - Document actual database schema changes
   - List real component names and locations
   - Include any configuration or environment variables added
4. Show the document to the developer for review

### Generic Feature Doc Structure (use when `{FEATURE_TEMPLATE}` is not found)

```markdown
# {Ticket Key}: {Feature Name}

**Date:** {YYYY-MM-DD}
**Ticket:** [{TICKET}]({JIRA_URL}/browse/{TICKET})
**Status:** Implemented

---

## Overview

{1–2 sentence summary of what this feature does and why it was built}

## Background

{Business context or user need this feature addresses}

## What Was Implemented

{Brief description of the overall implementation approach}

## API Changes

{New or modified endpoints — method, path, request/response shape — or "None"}

## Database Changes

{New models, modified fields, indexes, migrations — or "None"}

## UI Changes

{New pages, modals, components, or interactions — or "None"}

## Background Jobs

{New or modified jobs, triggers, schedules — or "None"}

## Configuration & Environment Variables

{New env vars or config keys added — or "None"}

## Security & Permissions

{Auth requirements, data access rules, input validation changes — or "None"}

## File Map

{List of files created or modified, with a one-line description of each}

## Known Limitations / Follow-ups

{Any known gaps, deferred work, or tickets created as follow-ups — or "None"}
```

---

## Step 4: Update Jira

Add a comment to the ticket:

```
mcp__atlassian__jira_add_comment({ issueIdOrKey: "$ARGUMENTS", comment: "..." })
```

Comment content:
```
📚 Feature documentation created
Path: {doc file path}
Covers: {brief list of documented areas}
```

---

## Step 5: Update State

Check `.claude/tickets/$ARGUMENTS.json`:

- If the file exists:
  - Set `steps.documentation.done = true` with current timestamp
  - Add the doc path to the `docs` array (create the array if absent)
  - Add a history entry: `{ step: "documentation", timestamp: "...", docPath: "..." }`
- If it doesn't exist, create a minimal state file:
  ```json
  {
    "ticket": "$ARGUMENTS",
    "docs": ["{doc file path}"],
    "steps": {
      "documentation": { "done": true, "timestamp": "{ISO timestamp}" }
    },
    "history": [
      { "step": "documentation", "timestamp": "{ISO timestamp}", "docPath": "{doc file path}" }
    ]
  }
  ```

Display:
```
📚 Documentation created: {doc path}

Next: Run /finish-ticket $ARGUMENTS to update Confluence and close the workflow.
```
