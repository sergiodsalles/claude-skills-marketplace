---
name: dev-lifecycle
description: >
  Use when the developer wants to work on a Jira ticket, create a branch, write a spec,
  review code, create a pull request, document a feature, or generate status reports.
  Triggers on: /start-ticket, /resume-ticket, /create-ticket, /create-spec, /create-branch,
  /review-code, /create-pr, /document-feature, /finish-ticket, /daily-report, /weekly-report,
  or any request like "start working on ticket", "create a PR", "review my code",
  "what did I do today", "generate a status report", "I'm done with this ticket".
---

# Dev Lifecycle

Full development workflow from ticket to closure, integrated with Jira, Git, and GitHub.

## Setup

Requires `.claude/dev-lifecycle.json` in the project root. Created automatically on first use —
you'll be prompted for your Jira project key and URL. Optional fields are auto-detected from git.

## Required Integrations

- **Atlassian MCP** — Jira and Confluence operations (required)
- **GitHub CLI (`gh`)** — PR and report commands (required for those commands)
- **Slack MCP** — Optional enrichment for daily/weekly reports

## Workflow

Ticket → Spec → Branch → Implement → Review → PR → Docs → Close

| Phase | Command | What it does |
|-------|---------|-------------|
| Create | `/create-ticket` | Create or standardize Jira tickets |
| Start | `/start-ticket TICKET` | Spec → branch → Jira update → state init |
| Resume | `/resume-ticket TICKET` | Reload context from previous session |
| Spec | `/create-spec TICKET` | Standalone technical specification |
| Branch | `/create-branch TICKET` | Named branch with Jira transition |
| Review | `/review-code` | Pre-PR quality gate with specialist agents |
| PR | `/create-pr` | Push, create PR, update Jira |
| Docs | `/document-feature TICKET` | Feature documentation from implementation |
| Close | `/finish-ticket TICKET` | Docs → Confluence → Jira → done |
| Status | `/daily-report [DATE]` | End-of-day status report |
| Status | `/weekly-report [DATE]` | Weekly compilation report |

## State Tracking

All commands share state via `.claude/tickets/{TICKET}.json`. This enables resuming
across sessions and enforcing workflow order (e.g., review before PR).
