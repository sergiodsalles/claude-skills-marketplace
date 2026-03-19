---
description: Full workflow orchestration — read Jira ticket, create spec, branch, update Jira, initialize state tracking
---

# Start Ticket: $ARGUMENTS

You are starting the full development workflow for Jira ticket **$ARGUMENTS**. Follow each step in order. Do NOT skip steps.

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
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, `{MAIN_BRANCH}`, `{DEV_BRANCH}`, `{BRANCH_PREFIXES}`, `{SPECS_DIR}`, `{FEATURES_DIR}`, `{SPEC_TEMPLATE}`, `{FEATURE_TEMPLATE}`, `{CONFLUENCE_SPACE}`, `{CONFLUENCE_PARENT}`, `{TRANSITIONS}`

---

## Step 1: Read Jira Ticket

If state file `.claude/tickets/$ARGUMENTS.json` exists and `steps.jira_read.done` is `true`, skip with a note and proceed to Step 2.

Use the Atlassian MCP to fetch the ticket:

```
mcp__atlassian__jira_get_issue({ issueIdOrKey: "$ARGUMENTS" })
```

Extract and display:
- **Summary**, **Description**, **Issue Type**, **Priority**, **Status**
- **Assignee**, **Labels**, **Components**
- **Sprint** (if any)

If the ticket doesn't exist or can't be read, STOP and inform the developer.

## Step 2: Clarify Scope

If state file exists and `steps.spec_created.done` is `true`, skip with a note and proceed to Step 3.

Ask the developer:
1. Which areas of the codebase are affected? (API, UI, Database, Background Jobs, Auth, Storage)
2. Any additional context not in the ticket?

## Step 3: Create Technical Specification

If state file exists and `steps.spec_created.done` is `true`, skip with a note and proceed to Step 4.

Invoke the brainstorming skill to explore the problem space, then:

1. Read the spec template from `{SPEC_TEMPLATE}` if the path is set and the file exists; skip silently if not found
2. Search the codebase for similar implementations or patterns relevant to this ticket
3. Ask clarifying questions ONE AT A TIME about:
   - Technical approach and design decisions
   - Database changes needed
   - API contract changes
   - UI/UX considerations
   - Security implications
   - Edge cases
   - Existing patterns to reuse
4. Write the spec to: `{SPECS_DIR}/{YYYY-MM-DD}-{TICKET}-{feature-name}.md`
   - Replace `{YYYY-MM-DD}` with today's date
   - Replace `{TICKET}` with the ticket key (e.g., `$ARGUMENTS`)
   - Replace `{feature-name}` with a short kebab-case name derived from the ticket summary
5. Show the spec to the developer for approval before proceeding

## Step 4: Create Branch

If state file exists and `steps.branch_created.done` is `true`, skip with a note and proceed to Step 5.

1. Map the Jira issue type to a branch prefix using `{BRANCH_PREFIXES}` from config:
   - Look up the issue type (case-insensitive) in `{BRANCH_PREFIXES}`
   - Fall back to `"feat"` if the type is not mapped
   - Common defaults: Story → `feat`, Task → `feat`, Bug → `bugfix`, Hotfix → `hotfix`
2. Generate the branch name: `{prefix}/{TICKET}_{short-description}`
   - `{short-description}`: lowercase, hyphens, 3-5 words max from ticket summary
3. Show the proposed branch name and ask for approval
4. **ASK which target branch** to create from (suggest `{DEV_BRANCH}` for features, `{MAIN_BRANCH}` for hotfixes)
5. Create the branch:
   ```bash
   git fetch origin
   git checkout {target-branch}
   git pull origin {target-branch}
   git checkout -b {branch-name}
   ```

## Step 5: Transition Jira to "In Progress"

If state file exists and `steps.jira_in_progress.done` is `true`, skip with a note and proceed to Step 6.

1. Get available transitions:
   ```
   mcp__atlassian__jira_get_transitions({ issueIdOrKey: "$ARGUMENTS" })
   ```
2. Fuzzy-match the returned transitions against the `{TRANSITIONS}` config value for "In Progress" (e.g., `inProgress`). If no match is found, display the available transitions and prompt the user to select one.
3. Execute the transition:
   ```
   mcp__atlassian__jira_transition_issue({ issueIdOrKey: "$ARGUMENTS", transitionId: "{id}" })
   ```
4. Add a comment to the Jira ticket:
   ```
   mcp__atlassian__jira_add_comment({ issueIdOrKey: "$ARGUMENTS", comment: "..." })
   ```
   Comment body:
   ```
   Development started
   - Branch: `{branch-name}`
   - Target: `{target-branch}`
   - Areas: {affected areas}
   - Spec: {spec file path}
   ```

## Step 6: Initialize State Tracking

If state file already exists and is fully populated, skip with a note.

Create the state file at `.claude/tickets/$ARGUMENTS.json`:

```json
{
  "ticket": {
    "key": "$ARGUMENTS",
    "summary": "{from Jira}",
    "issueType": "{from Jira}",
    "status": "In Progress"
  },
  "branch": {
    "name": "{branch-name}",
    "target": "{target-branch}",
    "createdAt": "{ISO timestamp}"
  },
  "affectedAreas": ["{areas from step 2}"],
  "specPath": "{spec file path}",
  "steps": {
    "jira_read": { "done": true, "at": "{ISO timestamp}" },
    "spec_created": { "done": true, "at": "{ISO timestamp}" },
    "branch_created": { "done": true, "at": "{ISO timestamp}" },
    "jira_in_progress": { "done": true, "at": "{ISO timestamp}" },
    "implementation": { "done": false },
    "code_review": { "done": false },
    "pr_created": { "done": false },
    "documentation": { "done": false },
    "confluence_updated": { "done": false },
    "jira_in_review": { "done": false }
  },
  "pr": { "url": null, "number": null },
  "docs": [],
  "confluence": { "pageId": null, "url": null },
  "history": []
}
```

Add a history entry: `{ "action": "workflow_started", "at": "{ISO timestamp}", "details": "Full workflow initiated" }`

## Step 7: Summary

Display a clear summary:

```
Ticket $ARGUMENTS — Setup Complete

Spec:     {spec path}
Branch:   {branch name} → {target branch}
Jira:     In Progress

Next steps:
1. Implement the feature following the spec
2. Run /review-code when ready
3. Run /create-pr after review passes
```
