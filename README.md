# Azure DevOps Skills for Claude Code

A collection of [Claude Code](https://claude.com/claude-code) skills that integrate with Azure DevOps for end-to-end work item management, PR creation, and post-merge cleanup. Works on macOS, Windows, and Linux.

These skills follow the [Agent Skills](https://agentskills.io) open standard and are primarily built for Claude Code. They also work on **Codex CLI**, **Antigravity**, and other compatible tools — see [Compatibility](#compatibility) for details.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [azure-work](skills/azure-work/) | `/azure-work` | Browse and implement Azure DevOps work items |
| [azure-pr](skills/azure-pr/) | `/azure-pr` | Create pull requests in Azure DevOps |
| [pr-complete](skills/pr-complete/) | `/pr-complete` | Post-merge branch cleanup |

## Workflow

The three skills compose into a full development cycle:

```
/azure-work → pick up item, create branch, implement
      ↓
/azure-pr --work-items <id> → push and create PR
      ↓
/pr-complete → post-merge cleanup
```

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) (`az`) with the `azure-devops` extension
- Git

### Installing Azure CLI

| Platform | Command |
|----------|---------|
| macOS | `brew install azure-cli` |
| Windows | `winget install --exact --id Microsoft.AzureCLI` |
| Linux | `curl -sL https://aka.ms/InstallAzureCLIDeb \| sudo bash` |

Then install the Azure DevOps extension:

```bash
az extension add --name azure-devops
```

And authenticate:

```bash
az login --allow-no-subscriptions
```

## Installation

Install individual skills — pick only what you need.

### Option A: Per-Project (recommended for teams)

Skills live in the project's `.claude/skills/` directory. Commit them to the repo so the whole team gets them.

**macOS / Linux:**

```bash
git clone https://github.com/ahmadharis/agentskills.git /tmp/agentskills
cp -r /tmp/agentskills/skills/azure-pr .claude/skills/
cp -r /tmp/agentskills/skills/azure-work .claude/skills/
cp -r /tmp/agentskills/skills/pr-complete .claude/skills/
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/ahmadharis/agentskills.git $env:TEMP\agentskills
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-pr .claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-work .claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\pr-complete .claude\skills\
```

### Option B: Global (for personal use across all projects)

Skills installed once in `~/.claude/skills/`. Available in every project.

**macOS / Linux:**

```bash
git clone https://github.com/ahmadharis/agentskills.git /tmp/agentskills
cp -r /tmp/agentskills/skills/azure-pr ~/.claude/skills/
cp -r /tmp/agentskills/skills/azure-work ~/.claude/skills/
cp -r /tmp/agentskills/skills/pr-complete ~/.claude/skills/
```

**Windows (PowerShell):**

```powershell
git clone https://github.com/ahmadharis/agentskills.git $env:TEMP\agentskills
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-pr $env:USERPROFILE\.claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-work $env:USERPROFILE\.claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\pr-complete $env:USERPROFILE\.claude\skills\
```

After installing, restart Claude Code and verify the skills appear in the `/` menu.

## Configuration

Configuration is always per-project via `.claude/settings.local.json`, regardless of where skills are installed. This file is inside `.claude/` which should be gitignored (it contains user-specific settings).

### Automatic Setup

`azure-work` prompts for your backlog URL on first run and saves configuration automatically. Just run `/azure-work` and follow the prompts.

### Manual Setup

Add the `env` key to your project's `.claude/settings.local.json`:

```json
{
  "env": {
    "ADO_BACKLOG_URL": "https://dev.azure.com/myorg/myproject/_backlogs/backlog/myteam/Backlog%20items",
    "ADO_ORG": "myorg",
    "ADO_PROJECT": "myproject"
  }
}
```

### All Environment Variables

| Variable | Used By | Required | Default | Description |
|----------|---------|----------|---------|-------------|
| `ADO_BACKLOG_URL` | azure-work | Yes | — | Full Azure DevOps backlog URL |
| `ADO_ORG` | azure-work | Yes | Auto-extracted | Azure DevOps organization |
| `ADO_PROJECT` | azure-work | Yes | Auto-extracted | Azure DevOps project |
| `ADO_WORK_ITEM_FILTER` | azure-work | No | *(none)* | Tag to filter work items |

`azure-pr` and `pr-complete` do not require environment variables — they extract what they need from the git remote URL.

### Advanced: Tag Filtering

To only show work items with a specific tag in browse mode, add `ADO_WORK_ITEM_FILTER`:

```json
{
  "env": {
    "ADO_BACKLOG_URL": "https://dev.azure.com/myorg/myproject/_backlogs/backlog/myteam/Backlog%20items",
    "ADO_ORG": "myorg",
    "ADO_PROJECT": "myproject",
    "ADO_WORK_ITEM_FILTER": "claude"
  }
}
```

When unset, all open work items are listed. Direct mode (`/azure-work 12345`) always fetches the specified item regardless of filter.

## Compatibility

These skills use the [Agent Skills](https://agentskills.io) open standard (`SKILL.md` with YAML frontmatter), so they work beyond Claude Code.

| Platform | Install to | Status |
|----------|-----------|--------|
| Claude Code | `.claude/skills/` or `~/.claude/skills/` | Full support |
| Codex CLI | `.agents/skills/` or `~/.agents/skills/` | Core workflow works. `azure-work` skill-chaining steps (`superpowers:brainstorming`, etc.) are skipped gracefully. |
| Antigravity / Others | Platform's skill directory | Same as Codex — core works, Claude-specific skill references ignored. |

**Configuration on non-Claude platforms:** Set `ADO_ORG`, `ADO_PROJECT`, and `ADO_BACKLOG_URL` as environment variables through your platform's mechanism instead of `.claude/settings.local.json`.

## Skill Details

See each skill's own README for detailed documentation:

- [azure-work](skills/azure-work/README.md) — work item browsing, implementation workflow, state management
- [azure-pr](skills/azure-pr/README.md) — PR creation, branch naming, work item linking
- [pr-complete](skills/pr-complete/README.md) — post-merge cleanup, branch deletion

## License

[MIT](LICENSE)
