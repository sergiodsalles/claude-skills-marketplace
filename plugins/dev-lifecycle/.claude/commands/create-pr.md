---
description: Create a pull request with comprehensive description, push to remote, update Jira
---

# Create Pull Request

You are creating a pull request for the current branch.

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
   - Check `gh auth status`. If not authenticated: **warn and stop** — required for this command.
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, `{MAIN_BRANCH}`, `{DEV_BRANCH}`, `{BRANCH_PREFIXES}`, `{SPECS_DIR}`, `{FEATURES_DIR}`, `{SPEC_TEMPLATE}`, `{FEATURE_TEMPLATE}`, `{CONFLUENCE_SPACE}`, `{CONFLUENCE_PARENT}`, `{TRANSITIONS}`

---

## Step 1: Pre-Flight Checks

1. Get the current branch:
   ```bash
   git branch --show-current
   ```
   Extract the ticket key (e.g., `PROJ-123`) from the branch name using the `{PROJECT_KEY}` config value.

2. Load state file: `.claude/tickets/{TICKET}.json`

3. **Check idempotency:** If `steps.pr_created.done` is `true` in the state file, display the existing PR URL and ask whether to proceed anyway or stop.

4. **Check code review status:**
   - If `steps.code_review.done` is false or state file doesn't exist:
     ```
     ⚠️ WARNING: Code review has not been run for this ticket.
     It is REQUIRED to run /review-code before creating a PR.
     Proceed anyway? (not recommended)
     ```
   - If the developer insists on proceeding, note it in the PR description.

---

## Step 2: Analyze Changes

1. Get the full diff:
   ```bash
   git diff {target-branch}...HEAD --stat
   git log {target-branch}...HEAD --oneline
   ```

2. If a spec exists (from state file), read it for context.

3. Categorize changed files by area (API, Components, Database, Jobs, Lib).

---

## Step 3: Confirm Target Branch

**ASK the developer:** "Which branch should this PR target?"

Show the current state's target branch as the default suggestion (use `{DEV_BRANCH}` from config if no state exists). Display:
```
Current branch: {branch-name}
Suggested target: {target-branch} (from branch creation / config default)
Confirm or specify different target:
```

---

## Step 4: Push to Remote

```bash
git push -u origin {branch-name}
```

If push fails (e.g., diverged history), inform the developer and ask how to proceed. Never force push without explicit approval.

---

## Step 5: Generate PR Description

Create a comprehensive PR description. Fill in ALL sections:

- **Ticket:** Link to Jira ticket (`{JIRA_URL}/browse/{TICKET}`)
- **Target Branch:** The confirmed target
- **Type of Change:** Derived from issue type
- **Issue Description:** From Jira ticket description
- **Changes Summary:** High-level overview from analyzing the diff
- **Files Changed:** Grouped by area (API, Components, Database, Jobs, Lib)
- **API Changes:** Table of new/modified endpoints (if any)
- **UI Changes:** Description of visual changes (if any)
- **Testing Instructions:** Step-by-step QA guide based on the spec
- **Checklist:** Pre-filled based on what was done

---

## Step 6: Create PR

Use the GitHub CLI:

```bash
gh pr create --title "{type}: {short description} ({TICKET})" --body "$(cat <<'EOF'
{PR body from step 5}
EOF
)" --base {target-branch}
```

Title format examples:
- `feat: add audio generation with tone selection (PROJ-123)`
- `fix: upload timeout for large files (PROJ-456)`
- `hotfix: auth crash on expired sessions (PROJ-789)`

If `gh` fails for any reason, output the branch name and target branch so the developer can open the PR manually via the repository's web interface.

---

## Step 7: Update Jira

1. Get available transitions:
   ```
   mcp__atlassian__jira_get_transitions({ issueIdOrKey: "{TICKET}" })
   ```
2. Fuzzy-match the returned transition names against the `{TRANSITIONS}` config value for `"in_review"` (e.g., "In Review", "Code Review"). Pick the closest match — do NOT hardcode a transition ID.
3. Execute the matched transition via `mcp__atlassian__jira_transition_issue`.
4. Add a comment via `mcp__atlassian__jira_add_comment`:
   ```
   🔍 Pull Request created
   PR: {PR URL}
   Target: `{target-branch}`
   Changes: {1-2 sentence summary}
   Review status: {passed / skipped}
   ```

---

## Step 8: Update State

Update `.claude/tickets/{TICKET}.json`:
- Set `steps.pr_created.done = true` with timestamp
- Set `steps.jira_in_review.done = true` with timestamp
- Set `pr.url` and `pr.number`
- Add history entry

Display:
```
✅ PR created: {PR URL}
   Target: {target-branch}
   Jira: Transitioned to In Review

Next steps:
- Wait for PR review/approval
- Run /document-feature {TICKET} to create feature docs
- Run /finish-ticket {TICKET} when PR is merged
```
