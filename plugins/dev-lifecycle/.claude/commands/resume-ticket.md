---
description: Resume work on a ticket from a previous session — loads state and shows progress
---

# Resume Ticket: $ARGUMENTS

You are resuming work on Jira ticket **$ARGUMENTS**. Load context and help the developer continue.

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

## Step 1: Load State

Read the state file: `.claude/tickets/$ARGUMENTS.json`

### If the state file exists:

Load all fields and proceed to Step 2.

### If the state file is missing:

Reconstruct state from available sources:

**1. Check for a matching branch:**

```bash
git branch -a | grep $ARGUMENTS
```

If no branch is found, suggest running `/start-ticket $ARGUMENTS` instead and stop.

**2. Identify the branch and target:**

- Use the matching branch name (strip `remotes/origin/` prefix if needed)
- Determine the target branch: run `git log --oneline {branch}...{DEV_BRANCH}` and `git log --oneline {branch}...{MAIN_BRANCH}` — the one with fewer diverging commits is the likely target. If ambiguous, default to `{DEV_BRANCH}`.

**3. Fetch ticket data from Jira:**

```
mcp__atlassian__jira_get_issue({ issueIdOrKey: "$ARGUMENTS" })
```

Extract: `summary`, `issueType`, `status`, `description`.

**4. Check for existing spec:**

Search `{SPECS_DIR}/` for a file matching `*$ARGUMENTS*`. Record the path if found.

**5. Check for an open PR:**

```bash
gh pr list --head {branch-name} --json number,url,state
```

Record PR number/URL if found.

**6. Detect completed steps from evidence:**

| Evidence | Inferred Step |
|---|---|
| Branch exists | `branch_created: done`, `jira_read: done` |
| Spec file found in `{SPECS_DIR}/` | `spec_created: done` |
| Jira status is "In Progress" or later | `jira_in_progress: done` |
| Git commits on branch beyond base | `implementation: done` (partial — mark done) |
| PR found via `gh pr list` | `pr_created: done`, `code_review: done` |
| Jira status is "In Review" | `jira_in_review: done` |

**7. Create the reconstructed state file** at `.claude/tickets/$ARGUMENTS.json`:

```json
{
  "ticket": {
    "key": "$ARGUMENTS",
    "summary": "{from Jira}",
    "issueType": "{from Jira}",
    "status": "{from Jira}"
  },
  "branch": {
    "name": "{detected branch name}",
    "target": "{detected target branch}",
    "createdAt": null
  },
  "affectedAreas": [],
  "specPath": "{spec path if found, else null}",
  "steps": {
    "jira_read": { "done": {true/false}, "at": null },
    "spec_created": { "done": {true/false}, "at": null },
    "branch_created": { "done": {true/false}, "at": null },
    "jira_in_progress": { "done": {true/false}, "at": null },
    "implementation": { "done": {true/false}, "at": null },
    "code_review": { "done": {true/false}, "at": null },
    "pr_created": { "done": {true/false}, "at": null },
    "documentation": { "done": false },
    "confluence_updated": { "done": false },
    "jira_in_review": { "done": {true/false}, "at": null }
  },
  "pr": { "url": "{pr url or null}", "number": "{pr number or null}" },
  "docs": [],
  "confluence": { "pageId": null, "url": null },
  "history": [
    { "action": "state_reconstructed", "at": "{ISO timestamp}", "details": "State reconstructed from git and Jira — no prior state file found" }
  ]
}
```

Notify the developer: "No state file found. Reconstructed state from git and Jira. Please verify the detected progress below."

---

## Step 2: Display Progress

Show the workflow progress as a checklist:

```
{TICKET} — {summary}
Branch: {branch name} → {target branch}
Status: {Jira status}

Workflow Progress:
  [done] Jira ticket read
  [done] Spec created — {spec path}
  [done] Branch created — {branch name}
  [done] Jira → In Progress
  [    ] Implementation
  [    ] Code review
  [    ] PR created
  [    ] Documentation
  [    ] Confluence updated
  [    ] Jira → In Review
```

Use `[done]` for completed steps and `[    ]` for pending steps.

---

## Step 3: Identify Next Action

Map the next incomplete step to the relevant action:

| Next Step | Suggested Action |
|---|---|
| `implementation` | "Ready to implement. Read the spec at {path} to continue." |
| `code_review` | "Implementation seems done. Run `/review-code` to review changes." |
| `pr_created` | "Review passed. Run `/create-pr` to create the pull request." |
| `documentation` | "PR created. Run `/document-feature $ARGUMENTS` to document." |
| `confluence_updated` | "Run `/finish-ticket $ARGUMENTS` to complete the workflow." |
| `jira_in_review` | "Run `/finish-ticket $ARGUMENTS` to transition Jira." |

---

## Step 4: Context Loading

1. If the spec exists, briefly summarize its key points
2. Check `git log --oneline -10` on the current branch for recent work
3. Check `git diff --stat {target}...HEAD` to show what's changed so far
4. Offer to read any specific files the developer wants to review

---

## Step 5: Propose Action

Based on the analysis, propose the single most useful next action and ask the developer to confirm.
