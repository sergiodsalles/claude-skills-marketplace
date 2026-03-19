---
description: Create a properly named branch from a Jira ticket
---

# Create Branch: $ARGUMENTS

You are creating a development branch for Jira ticket **$ARGUMENTS**.

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

Check state file `.claude/tickets/$ARGUMENTS.json` — if `steps.branch_created.done` is `true`, skip to Step 6 and display the existing branch name.

Call:
```
mcp__atlassian__jira_get_issue({ issueIdOrKey: "$ARGUMENTS" })
```

Extract: Summary, Issue Type.

---

## Step 2: Generate Branch Name

1. Map issue type to prefix using `{BRANCH_PREFIXES}` config:
   - Look up the issue type (case-insensitive) in `{BRANCH_PREFIXES}`
   - If no match found, fall back to `"feat"`
   - Example defaults: Story/Task → `feat`, Bug → `bugfix`, Hotfix → `hotfix`
   - Sub-task: inherit parent type if available, otherwise apply the same lookup with fallback to `"feat"`

2. Generate description from summary:
   - Lowercase
   - Replace spaces with hyphens
   - Remove special characters
   - Keep to 3-5 words max
   - Example: "Add audio generation with tone selection" → `audio-generation-tone`

3. Compose branch name: `{prefix}/$ARGUMENTS_{description}`

4. Show the proposed name and ask for approval:
   ```
   Proposed branch: feat/$ARGUMENTS_audio-generation-tone
   Approve? (or suggest alternative)
   ```

---

## Step 3: Select Target Branch

**ASK the developer:** "Which branch should this be based on?"

Suggest based on issue type:
- Features/Stories/Tasks: `{DEV_BRANCH}`
- Bug fixes: `{DEV_BRANCH}`
- Hotfixes: `{MAIN_BRANCH}`

Show current branches for context:
```bash
git branch -a | grep -E "(main|develop|staging|stg|release)" | head -10
```

---

## Step 4: Create the Branch

```bash
git fetch origin
git checkout {target-branch}
git pull origin {target-branch}
git checkout -b {branch-name}
```

Verify the branch was created:
```bash
git branch --show-current
```

---

## Step 5: Transition Jira

1. Get available transitions:
   ```
   mcp__atlassian__jira_get_transitions({ issueIdOrKey: "$ARGUMENTS" })
   ```
2. Fuzzy-match the returned transition names against the `{TRANSITIONS}` config value for `"in_progress"` (e.g., "In Progress", "In Development"). Pick the closest match — do NOT hardcode a transition ID.
3. Execute the matched transition to move the ticket to In Progress.
4. Add a comment via `mcp__atlassian__jira_add_comment`:
   ```
   Branch created: `{branch-name}`
   Target: `{target-branch}`
   ```

---

## Step 6: Update State

Check `.claude/tickets/$ARGUMENTS.json` — if it exists, merge the new fields; otherwise create it:

```json
{
  "ticket": { "key": "$ARGUMENTS", "summary": "...", "issueType": "...", "status": "In Progress" },
  "branch": { "name": "{branch-name}", "target": "{target-branch}", "createdAt": "..." },
  "steps": {
    "branch_created": { "done": true, "at": "..." },
    "jira_in_progress": { "done": true, "at": "..." }
  },
  "history": [{ "action": "branch_created", "at": "...", "details": "{branch-name} from {target-branch}" }]
}
```

Display: "Branch `{branch-name}` created from `{target-branch}`. Ready to implement!"
