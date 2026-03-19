---
description: Pre-PR quality gate — dispatches parallel specialist agents to review code changes
---

# Code Review

You are performing a comprehensive code review before creating a PR. This is a REQUIRED quality gate.

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
   - If missing or less than current version (1), offer to migrate by backfilling new fields with defaults
4. Verify required tools:
   - Check Atlassian MCP is available. If not: **warn and stop**.
   - Check `gh auth status`. If not authenticated: **warn but continue**.
5. Set variables from config: `{PROJECT_KEY}`, `{JIRA_URL}`, `{MAIN_BRANCH}`, `{DEV_BRANCH}`, `{BRANCH_PREFIXES}`, `{SPECS_DIR}`, `{FEATURES_DIR}`, `{SPEC_TEMPLATE}`, `{FEATURE_TEMPLATE}`, `{CONFLUENCE_SPACE}`, `{CONFLUENCE_PARENT}`, `{TRANSITIONS}`

## Step 1: Load Context

1. Find the current ticket from branch name:
   ```bash
   git branch --show-current
   ```
   Extract the ticket key (e.g., `{PROJECT_KEY}-123`) from the branch name.

2. Load state file if it exists: `.claude/tickets/{TICKET}.json`
3. Determine the target branch from state, or ask the developer.

## Step 2: Gather Changes

Get the full diff against the target branch:

```bash
git diff {target-branch}...HEAD --stat
git diff {target-branch}...HEAD
```

If the diff is empty, inform the developer and stop.

List all changed files and categorize them using generic directory pattern detection:

- **API/Backend:** files in `api/`, `routes/`, `server/`, `services/`, `lib/`, or containing route/endpoint definitions
- **Frontend:** files in `components/`, `pages/`, `app/`, `views/`, or `.tsx`/`.jsx` files
- **Database:** files in `prisma/`, `migrations/`, `models/`, `schema/`, or containing schema definitions
- **Background Jobs:** files in `trigger/`, `jobs/`, `workers/`, `queues/`
- **Auth/Security:** files in `auth/`, `permissions/`, `middleware/`, or containing auth/permission logic

## Step 3: Dispatch Specialist Agents

Based on the files changed, dispatch the appropriate agents **in parallel** using the Agent tool.

### Always Include:

**code-quality-specialist** — Reviews all changed files for code quality.

- Try `subagent_type="code-quality-specialist"` first.
- If unavailable, fall back to `subagent_type="general-purpose"` with the prompt: "You are a code quality reviewer. Review all changed files for correctness, maintainability, code smells, naming conventions, and adherence to project conventions found in CLAUDE.md or AGENTS.md."

### Conditionally Include (based on changed file categories):

- **Backend files detected** → try `subagent_type="backend-specialist"`, fallback `subagent_type="general-purpose"` with prompt: "You are a backend code reviewer. Review API routes, database changes, services, background jobs for correctness, performance, and security."
- **Frontend files detected** → try `subagent_type="frontend-specialist"`, fallback `subagent_type="general-purpose"` with prompt: "You are a frontend code reviewer. Review components, pages, hooks, client-side patterns for correctness, accessibility, and performance."
- **UI/UX changes detected** (new UI components, layout files, significant visual changes) → try `subagent_type="ui-ux-specialist"`, fallback `subagent_type="general-purpose"` with prompt: "You are a UI/UX reviewer. Review design quality, responsiveness, accessibility, and user experience patterns."
- **Auth/security files detected** → try `subagent_type="security-specialist"`, fallback `subagent_type="general-purpose"` with prompt: "You are a security reviewer. Review authentication, authorization, input validation, data protection, and secrets management."

Each agent must receive:
1. The full diff for their relevant files
2. The ticket context (summary, spec path if available)
3. Instructions to read `CLAUDE.md`, `AGENTS.md`, or equivalent project configuration files to extract project-specific conventions and apply them during review
4. Instructions to categorize findings as:
   - **BLOCKING** — Must fix before PR (bugs, security issues, convention violations)
   - **RECOMMENDED** — Should fix, improves quality (naming, patterns, readability)
   - **OPTIONAL** — Nice to have (style, minor optimizations)

## Step 4: Convention Checks

Read the project's `CLAUDE.md`, `AGENTS.md`, or equivalent configuration files to extract project-specific conventions. Pass these conventions to the code-quality agent as part of its review scope. If no project config files exist, skip convention checks.

## Step 5: Compile Results

Aggregate all findings into a single report:

```
Code Review Report — {TICKET}

BLOCKING ({count})
  1. {file}:{line} — {description}
  2. ...

RECOMMENDED ({count})
  1. {file}:{line} — {description}
  2. ...

OPTIONAL ({count})
  1. {file}:{line} — {description}
  2. ...

Convention Check: Pass / {violations}
```

## Step 6: Handle Blocking Issues

If there are BLOCKING findings:
1. Ask the developer if they want help fixing them
2. Fix the blocking issues
3. Re-run the review on the changed files to verify fixes

If no BLOCKING findings:
```
Code review passed! Run /create-pr to create the pull request.
```

## Step 7: Update State

If state file exists:
- Set `steps.code_review.done = true` with timestamp
- Add history entry with review summary (counts per category)
