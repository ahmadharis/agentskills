# azure-work

Azure DevOps work item skill for Claude Code. Browse, pick up, and implement work items directly from your Azure DevOps board.

## What It Does

Connects Claude Code to your Azure DevOps backlog so you can:

1. **Browse** open work items and pick one to implement
2. **Jump directly** to a specific work item by number
3. **Read** all work item details including attachments (images, PDFs, docs)
4. **Create** a feature branch and update the work item state
5. **Implement** the work using Claude's coding skills
6. **Hand off** to `azure-pr` for PR creation

## Prerequisites

- **Azure CLI** (`az`) — [Installation guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
  - macOS: `brew install azure-cli`
  - Windows: `winget install --exact --id Microsoft.AzureCLI`
  - Linux: `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`
- **Azure DevOps extension**: `az extension add --name azure-devops`
- **Git**
- **Azure CLI authentication**: `az login --allow-no-subscriptions`

## Configuration

Configuration is stored as environment variables in your project's `.claude/settings.local.json` file. This file is per-project and should be gitignored (it lives inside `.claude/` which is excluded by default).

### Automatic Setup (First Run)

On first invocation, `azure-work` will prompt you for your backlog URL and save the configuration automatically. Just run `/azure-work` and follow the prompts.

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

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ADO_BACKLOG_URL` | Yes | — | Full Azure DevOps backlog URL |
| `ADO_ORG` | Yes | Auto-extracted from URL | Azure DevOps organization name |
| `ADO_PROJECT` | Yes | Auto-extracted from URL | Azure DevOps project name |
| `ADO_WORK_ITEM_FILTER` | No | *(none)* | Tag to filter work items by |

### Example Configurations

**Minimal (all open items):**

```json
{
  "env": {
    "ADO_BACKLOG_URL": "https://dev.azure.com/myorg/myproject/_backlogs/backlog/myteam/Backlog%20items",
    "ADO_ORG": "myorg",
    "ADO_PROJECT": "myproject"
  }
}
```

**With tag filter (only items tagged "claude"):**

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

When `ADO_WORK_ITEM_FILTER` is set, only work items containing that tag are shown in browse mode. Direct mode (`/azure-work 12345`) always fetches the specified item regardless of tags.

## Usage

### Browse Mode

```
/azure-work
```

Lists all open work items (or filtered by tag if `ADO_WORK_ITEM_FILTER` is set). You then pick one by entering its number.

### Direct Mode

```
/azure-work 12345
```

Jumps directly to work item 12345 — no listing step.

## Full Workflow

```
1. Check prerequisites (az CLI, authentication)
        ↓
2. Load configuration (env vars or first-run setup)
        ↓
3. Browse or direct-fetch work item
        ↓
4. Read full details (description, acceptance criteria, repro steps)
        ↓
5. Download and read all attachments
        ↓
6. Create feature branch: feature/<work-item-id>
        ↓
7. Update work item state (New → Active/Approved)
        ↓
8. Decide approach (brainstorm vs. direct implementation)
        ↓
9. Implement the work item
        ↓
10. Run verification checks
        ↓
11. Report results, suggest: /azure-pr --work-items <id>
```

## Work Item Field Extraction

The skill reads these fields from each work item:

| Field | Description |
|-------|-------------|
| Title | Work item title |
| Description | Full description (HTML converted to text) |
| Acceptance Criteria | Definition of done |
| State | Current state (New, Active, etc.) |
| Type | User Story, Bug, Task, etc. |
| Tags | Comma-separated tags |
| Assigned To | Current assignee |
| Priority | 1 (highest) to 4 (lowest) |
| Repro Steps | Bug-specific reproduction steps |

## Attachment Handling

The skill downloads and reads all attachments before starting implementation:

- **Text/Markdown/Code** — read directly
- **Images** — viewed by Claude (multimodal)
- **PDFs** — read with page selection
- **Word/Excel** — read if document skills are available

## State Transitions

| Work Item Type | State Set By This Skill |
|----------------|------------------------|
| Product Backlog Item | `Approved` |
| Bug | `Approved` |
| Task | `In Progress` |

The work item is NOT marked as Done/Resolved by this skill. That happens when the PR is closed.

## Integration with Other Skills

```
/azure-work → pick up item, create branch, implement
      ↓
/azure-pr --work-items <id> → push and create PR
      ↓
/pr-complete → post-merge cleanup
```

## Compatibility

Built for **Claude Code** but uses the [Agent Skills](https://agentskills.io) open standard (`SKILL.md` with YAML frontmatter), so the core workflow also works on Codex CLI, Antigravity, and other compatible tools.

**Platform-specific notes:**
- The implementation steps optionally invoke Claude Code skills (`superpowers:brainstorming`, `feature-dev:feature-dev`, `superpowers:verification-before-completion`). On other platforms these are skipped gracefully — the skill falls back to direct brainstorming/implementation.
- Configuration uses `.claude/settings.local.json` on Claude Code. On other platforms, set `ADO_ORG`, `ADO_PROJECT`, and `ADO_BACKLOG_URL` as environment variables through your platform's mechanism.
- On Codex CLI, install to `.agents/skills/` instead of `.claude/skills/`.

## Notes

- Configuration is always per-project, even if skills are installed globally
- Authentication uses Azure AD OAuth via `az login` — no PAT tokens needed
- The WIQL query excludes Done, Closed, and Removed items
- Direct mode works regardless of tag filters
