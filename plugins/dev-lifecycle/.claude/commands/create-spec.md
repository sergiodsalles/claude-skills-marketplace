---
description: Create a standalone technical specification from a Jira ticket
---

# Create Specification: $ARGUMENTS

You are creating a technical specification for Jira ticket **$ARGUMENTS**.

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

## Step 1: Read Jira Ticket

Check state: if `.claude/tickets/$ARGUMENTS.json` exists and `steps.spec_created.done` is `true`, show the existing spec path and ask: "Spec already created at {specPath}. Re-generate it or skip?"

Otherwise, fetch the ticket:

```
mcp__atlassian__jira_get_issue({ issueIdOrKey: "$ARGUMENTS" })
```

Extract: Summary, Description, Issue Type, Priority, Components, Labels.

---

## Step 2: Check for Existing Specs

Search for existing specs matching this ticket:

```
ls {SPECS_DIR}/*$ARGUMENTS* 2>/dev/null
```

If a spec already exists, show it and ask: "A spec already exists at {path}. Update it or create a new version?"

---

## Step 3: Clarifying Questions

Ask these questions ONE AT A TIME (wait for each answer before asking the next):

1. **Technical approach:** "What's your preferred approach for implementing this? Any design constraints?"
2. **Affected areas:** "Which areas of the codebase will this touch?" (API, Components, Database, Jobs, Auth, Storage)
3. **Database changes:** "Are there any database schema changes needed? New models, fields, indexes?"
4. **API changes:** "Any new or modified API endpoints?"
5. **UI considerations:** "Any UI/UX requirements? New pages, modals, components?"
6. **Security implications:** "Any security considerations? Auth changes, data access, input validation?"
7. **Edge cases:** "What edge cases should we handle?"
8. **Existing patterns:** "Are there similar features we should follow as a pattern?"

After collecting answers, search the codebase for relevant patterns and similar implementations.

---

## Step 4: Write Specification

1. Check if `{SPEC_TEMPLATE}` exists:
   - If it exists, read it and use it as the structure for the spec
   - If it does not exist, use the generic spec structure below
2. Write the spec to: `{SPECS_DIR}/{YYYY-MM-DD}-$ARGUMENTS-{feature-name}.md`
3. Fill in ALL sections with concrete details from the ticket and clarifying answers
4. Include specific file paths from the codebase where changes will be made
5. Show the completed spec to the developer

### Generic Spec Structure (use when `{SPEC_TEMPLATE}` is not found)

```markdown
# {Ticket Key}: {Feature Name}

**Date:** {YYYY-MM-DD}
**Ticket:** [{TICKET}]({JIRA_URL}/browse/{TICKET})
**Status:** Draft

---

## Overview

{1–2 sentence summary of what this feature does and why it's needed}

## Background

{Business context, user pain point, or motivation from the Jira ticket}

## Scope

### In Scope
- {item}

### Out of Scope
- {item}

## Technical Approach

{Description of the implementation strategy and any key design decisions}

## Affected Areas

- [ ] API Routes
- [ ] UI Components / Pages
- [ ] Database Schema
- [ ] Background Jobs
- [ ] Auth / Permissions
- [ ] Storage / Upload
- [ ] Other: ___

## Data Model Changes

{New models, modified fields, indexes, migrations — or "None"}

## API Changes

{New or modified endpoints with method, path, request/response shape — or "None"}

## UI Changes

{New pages, modals, components, or interactions — or "None"}

## Security Considerations

{Auth requirements, data access rules, input validation, or "None"}

## Edge Cases

{Known edge cases and how they should be handled}

## File Paths

{List of specific files expected to be created or modified}

## Open Questions

- [ ] {Question}

## Acceptance Criteria

- [ ] {Testable criterion}
- [ ] {Testable criterion}
```

---

## Step 5: Update Jira

Add a comment to the ticket:

```
mcp__atlassian__jira_add_comment({ issueIdOrKey: "$ARGUMENTS", comment: "..." })
```

Comment content:
```
📄 Technical specification created
Path: {spec file path}
Areas: {affected areas}
Key decisions: {1-2 sentence summary of approach}
```

---

## Step 6: Update State

Check `.claude/tickets/$ARGUMENTS.json`:

- If the file exists:
  - Set `steps.spec_created.done = true` with current timestamp
  - Set `specPath` to the spec file path
  - Add a history entry: `{ step: "spec_created", timestamp: "...", specPath: "..." }`
- If it doesn't exist, create a minimal state file:
  ```json
  {
    "ticket": "$ARGUMENTS",
    "specPath": "{spec file path}",
    "steps": {
      "spec_created": { "done": true, "timestamp": "{ISO timestamp}" }
    },
    "history": [
      { "step": "spec_created", "timestamp": "{ISO timestamp}", "specPath": "{spec file path}" }
    ]
  }
  ```
