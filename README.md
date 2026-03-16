# Azure DevOps Skills for Claude Code

A collection of [Claude Code](https://claude.com/claude-code) skills that integrate with Azure DevOps for end-to-end work item management, PR creation, and post-merge cleanup. Works on macOS, Windows, and Linux.

These skills follow the [Agent Skills](https://agentskills.io) open standard and are primarily built for Claude Code. They also work on **Codex CLI**, **Antigravity**, and other compatible tools — see [Compatibility](#compatibility) for details.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [azure-work](skills/azure-work/) | `/azure-work` | Browse and implement Azure DevOps work items |
| [azure-pr](skills/azure-pr/) | `/azure-pr` | Create pull requests in Azure DevOps |
| [pr-complete](skills/pr-complete/) | `/pr-complete` | Post-merge branch cleanup |
| [azure-pr-comments](skills/azure-pr-comments/) | `/azure-pr-comments` | List, add, reply to, and resolve PR comment threads |
| [azure-work-complete](skills/azure-work-complete/) | `/azure-work-complete` | Full autonomous cycle: pick up → implement → review → PR → cleanup |

> When installed as a plugin, skills are namespaced: `/agentskills:azure-work`, `/agentskills:azure-pr`, `/agentskills:azure-pr-comments`, `/agentskills:pr-complete`. When installed manually, the short names above apply.

## Workflow

The skills compose into a full development cycle — run each step manually, or use `azure-work-complete` to automate the entire sequence:

```
/agentskills:azure-work → pick up item, create branch, implement
      ↓
/simplify (or inline review) → code quality review
      ↓
/agentskills:azure-pr --work-items <id> → push and create PR
      ↓
/agentskills:azure-pr-comments → manage review feedback
      ↓
/agentskills:pr-complete → post-merge cleanup
```

Or run it all at once: `/agentskills:azure-work-complete`

For continuous autonomous mode: `/loop 30m /agentskills:azure-work-complete`

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

### Claude Code (recommended)

Install as a plugin for auto-updates and one-command setup:

```
# Add the marketplace (one-time)
/plugin marketplace add ahmadharis/agentskills

# Install the plugin
/plugin install agentskills@ahmadharis-agentskills
```

Skills become available as `/agentskills:azure-work`, `/agentskills:azure-pr`, `/agentskills:azure-pr-comments`, `/agentskills:pr-complete`, and `/agentskills:azure-work-complete`.

The marketplace auto-update mechanism keeps skills current — no need to manually pull updates.

### Codex CLI

Install individual skills using the `$skill-installer`:

```bash
$skill-installer install https://github.com/ahmadharis/agentskills/tree/main/skills/azure-work
$skill-installer install https://github.com/ahmadharis/agentskills/tree/main/skills/azure-pr
$skill-installer install https://github.com/ahmadharis/agentskills/tree/main/skills/pr-complete
$skill-installer install https://github.com/ahmadharis/agentskills/tree/main/skills/azure-pr-comments
$skill-installer install https://github.com/ahmadharis/agentskills/tree/main/skills/azure-work-complete
```

### Manual Installation

Copy skills directly into your skills directory. Pick only what you need.

**Per-project (recommended for teams)** — commit to the repo so the whole team gets them:

macOS / Linux:

```bash
git clone https://github.com/ahmadharis/agentskills.git /tmp/agentskills
cp -r /tmp/agentskills/skills/azure-pr .claude/skills/
cp -r /tmp/agentskills/skills/azure-work .claude/skills/
cp -r /tmp/agentskills/skills/pr-complete .claude/skills/
cp -r /tmp/agentskills/skills/azure-pr-comments .claude/skills/
cp -r /tmp/agentskills/skills/azure-work-complete .claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/ahmadharis/agentskills.git $env:TEMP\agentskills
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-pr .claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-work .claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\pr-complete .claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-pr-comments .claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-work-complete .claude\skills\
```

**Global (for personal use across all projects):**

macOS / Linux:

```bash
git clone https://github.com/ahmadharis/agentskills.git /tmp/agentskills
cp -r /tmp/agentskills/skills/azure-pr ~/.claude/skills/
cp -r /tmp/agentskills/skills/azure-work ~/.claude/skills/
cp -r /tmp/agentskills/skills/pr-complete ~/.claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/ahmadharis/agentskills.git $env:TEMP\agentskills
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-pr $env:USERPROFILE\.claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\azure-work $env:USERPROFILE\.claude\skills\
Copy-Item -Recurse $env:TEMP\agentskills\skills\pr-complete $env:USERPROFILE\.claude\skills\
```

After manual installation, restart Claude Code and verify the skills appear in the `/` menu.

## Configuration

Configuration is always per-project via `.claude/settings.local.json`, regardless of where skills are installed. This file is inside `.claude/` which should be gitignored (it contains user-specific settings).

### Automatic Setup

`azure-work` prompts for your backlog URL on first run and saves configuration automatically. Just run `/azure-work` (or `/agentskills:azure-work`) and follow the prompts.

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
| `ADO_TEAM` | azure-work | No | Auto-extracted (if in URL) | Azure DevOps team — scopes queries to team's area paths |
| `ADO_WORK_ITEM_FILTER` | azure-work | No | *(none)* | Tag to filter work items |

`azure-pr`, `azure-pr-comments`, and `pr-complete` do not require environment variables — they extract what they need from the git remote URL.

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

| Platform | Install Method | Status |
|----------|---------------|--------|
| Claude Code | Plugin (`/plugin install`) | Full support, auto-updates |
| Claude Code | Manual (`.claude/skills/` or `~/.claude/skills/`) | Full support |
| Codex CLI | `$skill-installer` or manual (`.agents/skills/`) | Core workflow works. `azure-work` skill-chaining steps (`superpowers:brainstorming`, etc.) are skipped gracefully. |
| Antigravity / Others | Platform's skill directory | Same as Codex — core works, Claude-specific skill references ignored. |

**Configuration on non-Claude platforms:** Set `ADO_ORG`, `ADO_PROJECT`, and `ADO_BACKLOG_URL` as environment variables through your platform's mechanism instead of `.claude/settings.local.json`.

## Skill Details

See each skill's own README for detailed documentation:

- [azure-work](skills/azure-work/README.md) — work item browsing, creation, update, and implementation workflow
- [azure-pr](skills/azure-pr/README.md) — PR creation, branch naming, work item linking
- [azure-pr-comments](skills/azure-pr-comments/README.md) — PR comment threads: list, add, reply, resolve, delete
- [pr-complete](skills/pr-complete/README.md) — post-merge cleanup, branch deletion
- [azure-work-complete](skills/azure-work-complete/README.md) — full autonomous cycle, /loop integration

## License

[MIT](LICENSE)
