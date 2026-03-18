# Claude Skills Marketplace

A curated collection of Claude Code skills for project management and team workflows.

## Available Skills

### project-pulse

Project management dashboard that aggregates **Jira**, **Slack**, and **GitHub/git** data into actionable summaries.

**Commands:**
| Command | Description |
|---------|-------------|
| `/start-my-day` | Morning briefing with blockers, team workload, PR queue, and priorities |
| `/end-my-day` | End-of-day wrap-up with progress, carry-forward items, and async messages |
| `/team-status [name]` | Deep dive on a team member's workload and activity |
| `/blockers` | Quick view of everything that's stuck |
| `/weekly-recap` | Stakeholder-ready weekly summary with metrics and trends |

**Requirements:**
- [Atlassian MCP server](https://github.com/anthropics/claude-code-plugins/tree/main/atlassian) (Jira access)
- Slack MCP server
- GitHub CLI (`gh`) authenticated

## Installation

### Option 1: Marketplace (recommended)

```bash
# Add this marketplace
/plugin marketplace add sergiodsalles/claude-skills-marketplace

# Install the skill
/plugin install project-pulse@sergio-claude-skills
```

### Option 2: Manual

Copy the plugin contents into your project:

```bash
# Clone the repo
git clone https://github.com/sergiodsalles/claude-skills-marketplace.git /tmp/skills

# Copy skill and commands to your project
cp -r /tmp/skills/plugins/project-pulse/.claude/skills/project-pulse .claude/skills/
cp /tmp/skills/plugins/project-pulse/.claude/commands/*.md .claude/commands/

# Clean up
rm -rf /tmp/skills
```

## Adding New Skills

1. Create a new directory under `plugins/<skill-name>/`
2. Add `.claude/skills/<skill-name>/SKILL.md` with frontmatter
3. Add any `/commands` as `.claude/commands/<command>.md`
4. Add a `plugin.json` with metadata
5. Register the plugin in the root `marketplace.json`

## License

MIT
