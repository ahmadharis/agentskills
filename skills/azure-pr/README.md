# azure-pr

Azure DevOps pull request skill for Claude Code. Handles the full PR workflow: branch creation, committing, pushing, and PR creation via `az repos pr create`.

## What It Does

Automates the entire pull request lifecycle for Azure DevOps repositories:

1. Creates a properly named feature branch (if needed)
2. Stages and commits changes with conventional commit messages
3. Pushes the branch to the remote
4. Creates a PR in Azure DevOps with title, description, and optional work item links

## Prerequisites

- **Azure CLI** (`az`) — [Installation guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
  - macOS: `brew install azure-cli`
  - Windows: `winget install --exact --id Microsoft.AzureCLI`
  - Linux: `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`
- **Azure DevOps extension**: `az extension add --name azure-devops`
- **Git**
- **Azure CLI authentication**: `az login`

## How Org/Project/Repo Are Detected

The skill extracts these values automatically from the git remote URL:

```
https://{org}@dev.azure.com/{org}/{project}/_git/{repo}
```

No manual configuration needed — just make sure your `origin` remote points to Azure DevOps.

## Branch Naming Conventions

| Prefix | Use Case |
|--------|----------|
| `feature/<name>` | New features |
| `fix/<name>` | Bug fixes |
| `refactor/<name>` | Refactoring |

## Usage

```
/azure-pr
/azure-pr --work-items 12345
/azure-pr --work-items 12345 67890
```

You can also just mention work item numbers naturally:
- "Create a PR for item 12345"
- "Push and create a PR, link #12345"

## Workflow

```
1. Verify az CLI is available
        ↓
2. Extract org/project/repo from git remote
        ↓
3. Create feature branch (if not on one)
        ↓
4. Stage and commit with conventional commit format
        ↓
5. Push branch to origin
        ↓
6. az repos pr create with title + description
        ↓
7. Link work items (if provided)
        ↓
8. Report PR URL to user
```

## Work Item Linking

Work items can be linked to the PR in several ways:

- **Explicit flag**: `/azure-pr --work-items 12345 67890`
- **Bare numbers**: "create a PR for 12345"
- **Hash prefix**: "link #12345 to the PR"

Multiple work items are space-separated in the `--work-items` flag.

## Commit Message Format

Uses conventional commit format:

```
<type>(<scope>): <subject>

<body>

Co-Authored-By: Claude <noreply@anthropic.com>
```

## Example

```
> /azure-pr --work-items 54321

Pushed branch feature/54321 to origin.
Created PR: feat(auth): add SSO login support
PR URL: https://dev.azure.com/org/project/_git/repo/pullrequest/789
Linked work item: 54321
```

## Compatibility

Built for **Claude Code** but uses the [Agent Skills](https://agentskills.io) open standard, so the core workflow also works on Codex CLI, Antigravity, and other compatible tools.

- On Codex CLI, install to `.agents/skills/` instead of `.claude/skills/`.
- No Claude Code-specific skill dependencies — all functionality uses `az` CLI and `git` directly.

## Integration

This skill works alongside:
- **azure-work** — picks up a work item and implements it, then hands off to `azure-pr`
- **pr-complete** — cleans up local branches after the PR is merged
