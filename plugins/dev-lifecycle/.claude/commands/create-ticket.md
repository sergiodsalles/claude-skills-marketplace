---
description: Create a new Jira ticket with standardized description, or enrich an existing ticket's description
---

# Create / Standardize Ticket: $ARGUMENTS

Create a new Jira ticket with a standardized description, or enrich an existing ticket's description.

**Two modes:**
- `/create-ticket` (no arguments) — create a new ticket interactively
- `/create-ticket {PROJECT_KEY}-XXXX` — standardize an existing ticket's description

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

## Mode Detection

- If `$ARGUMENTS` is empty → **New Ticket Mode**
- If `$ARGUMENTS` matches `{PROJECT_KEY}-\d+` → **Enrich Mode**

---

## Mode 1: New Ticket (no arguments)

### Step 1: Gather Intent

Ask the user: "Describe what you need — a feature, bug, or task. Be as detailed or brief as you like."

### Step 2: Infer Fields

From the user's description, infer:
- **Issue Type**: Story (feature/enhancement), Bug (defect/broken behavior), Task (chore/configuration/refactor)
- **Affected Areas**: Look for signals:
  - "API", "endpoint", "route" → API Routes
  - "UI", "page", "component", "modal" → UI Components / Pages
  - "schema", "model", "migration", "database" → Database Schema
  - "job", "task", "background", "trigger" → Background Jobs
  - "auth", "permission", "role" → Auth/Permissions
  - "upload", "storage", "file" → Storage/Upload
- **Summary**: A concise one-line title

If the issue type is ambiguous, ask the user to confirm. Do NOT ask about things you can reasonably infer.

### Step 3: Generate Description

Use the standardized format based on issue type:

**Story / Enhancement:**
```
## Context
[Why this is needed — business context, user pain point]

## Description
[What needs to be built/changed]

## Affected Areas
- [ ] API Routes
- [ ] UI Components
- [ ] Pages
- [ ] Database Schema
- [ ] Background Jobs
- [ ] Auth/Permissions
- [ ] Storage/Upload
- [ ] Other: ___

## Acceptance Criteria
- [ ] [Specific, testable criterion]
- [ ] [Another criterion]

## Notes
[Optional: links, screenshots, related tickets]
```

**Bug:**
```
## Context
[Where and when this occurs]

## Current Behavior
[What happens now]

## Expected Behavior
[What should happen]

## Steps to Reproduce
1. [Step]
2. [Step]

## Affected Areas
- [ ] API Routes
- [ ] UI Components
- [ ] Pages
- [ ] Database Schema
- [ ] Background Jobs
- [ ] Auth/Permissions
- [ ] Storage/Upload
- [ ] Other: ___

## Notes
[Optional: logs, screenshots, browser/device info, related tickets]
```

**Task / Sub-task:**
```
## Context
[Why this task is needed]

## Description
[What needs to be done]

## Checklist
- [ ] [Action item]
- [ ] [Action item]

## Affected Areas
- [ ] API Routes
- [ ] UI Components
- [ ] Pages
- [ ] Database Schema
- [ ] Background Jobs
- [ ] Auth/Permissions
- [ ] Storage/Upload
- [ ] Other: ___

## Notes
[Optional: links, dependencies]
```

Only check the "Affected Areas" boxes that are actually relevant — leave the rest unchecked.

### Step 4: Present Preview

Show the user what will be created:

```
─── Ticket Preview ───────────────────────────
Project:    {PROJECT_KEY}
Type:       {Story | Bug | Task}
Summary:    {generated summary}

Description:
{formatted description}
───────────────────────────────────────────────
```

Ask: "Does this look good? You can ask me to adjust anything, or approve to create."

### Step 5: Create Ticket

Once approved, create the ticket:

1. Call `mcp__atlassian__jira_create_issue` with:
   - `projectKey`: "{PROJECT_KEY}"
   - `issueType`: the inferred type
   - `summary`: the generated summary
   - `description`: the standardized description (pure markdown, actual line breaks — NEVER use escaped `\n`)

2. Display the result: **"Created [{PROJECT_KEY}-XXXX]({JIRA_URL}/browse/{TICKET}) — {summary}"**

3. Ask: "Would you like to start working on this ticket now? (`/start-ticket {PROJECT_KEY}-XXXX`)"

---

## Mode 2: Enrich Existing Ticket (with {PROJECT_KEY}-XXXX argument)

### Step 1: Read Ticket

Call `mcp__atlassian__jira_get_issue` with the provided ticket key.

Extract: summary, current description, issue type, status.

### Step 2: Analyze & Rewrite

Analyze the current description:
- Identify what information is present but unstructured
- Identify what's missing from the standard format
- Preserve ALL original intent and details — do not lose information

Rewrite the description into the standardized format for that issue type.

### Step 3: Present Comparison

Show both versions:

```
─── Current Description ──────────────────────
{original description}

─── Proposed Description ─────────────────────
{standardized description}
───────────────────────────────────────────────
```

Ask: "Here's the standardized version. Approve, adjust, or cancel?"

### Step 4: Update Ticket

Once approved:

1. Call `mcp__atlassian__jira_update_issue` to update the description
   - Use pure markdown with actual line breaks — NEVER use escaped `\n`

2. Call `mcp__atlassian__jira_add_comment`:
   "Description standardized to follow team template format."

3. Confirm: **"Updated [{PROJECT_KEY}-XXXX]({JIRA_URL}/browse/{TICKET}) — description standardized."**

---

## Important: Markdown Formatting

All descriptions sent to Jira MUST be pure markdown with real line breaks.
NEVER use escaped `\n` characters in any Jira field — they render as literal text and break formatting.
