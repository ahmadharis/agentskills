# azure-pr-comments

Azure DevOps PR comment management skill for Claude Code. List, add, reply to, resolve, and delete comment threads on pull requests.

## What It Does

Manages the full comment lifecycle on Azure DevOps pull requests:

1. **List** all comment threads on a PR with status, file location, and author
2. **Add** general PR-level comments or inline file-level comments
3. **Reply** to existing comment threads
4. **Resolve** threads (fixed, won't fix, closed, by design)
5. **Delete** comments (soft-delete)

## Why a Separate Skill

The `az repos pr` CLI has **no native subcommands for comments or threads**. All operations require `az devops invoke` wrapping the REST API with JSON payloads. This is a fundamentally different interaction model from PR creation (`azure-pr`), so it lives in its own skill.

## Prerequisites

- **Azure CLI** (`az`) -- [Installation guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
  - macOS: `brew install azure-cli`
  - Windows: `winget install --exact --id Microsoft.AzureCLI`
  - Linux: `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`
- **Azure DevOps extension**: `az extension add --name azure-devops`
- **Git** (for auto-detecting repo info and current branch PR)
- **Azure CLI authentication**: `az login`
- **jq** (for parsing JSON responses)

## Usage

### List Comments

```
/azure-pr-comments
/azure-pr-comments 123
```

Without a PR ID, detects the active PR for the current branch. With a PR ID, lists threads on that specific PR.

### Add a Comment

```
/azure-pr-comments add
```

Or naturally:

```
"Add a comment to PR 123: looks good, ship it"
"Comment on src/main.ts line 15: should we add error handling here?"
```

Supports both general PR-level comments and inline file-level comments with line numbers.

### Reply to a Thread

```
/azure-pr-comments reply
```

Or naturally:

```
"Reply to thread 5 on PR 123: fixed in latest commit"
"Respond to the review feedback on my PR"
```

### Resolve Threads

```
/azure-pr-comments resolve
```

Or naturally:

```
"Resolve thread 5 on PR 123"
"Mark all threads as fixed"
"Mark thread 3 as won't fix"
```

### Plugin (Namespaced) Commands

When installed as a plugin:

```
/agentskills:azure-pr-comments
/agentskills:azure-pr-comments 123
/agentskills:azure-pr-comments add
/agentskills:azure-pr-comments reply
/agentskills:azure-pr-comments resolve
```

## Thread Statuses

| Status | Value | Meaning |
|--------|-------|---------|
| Active | 1 | Open, needs attention |
| Fixed | 2 | Resolved as fixed |
| Won't Fix | 3 | Resolved as won't fix |
| Closed | 4 | Closed |
| By Design | 5 | Resolved as by design |
| Pending | 6 | Pending resolution |

## How It Works

Since the Azure CLI has no native PR comment commands, all operations use `az devops invoke` to call the Azure DevOps REST API directly:

- **List**: `GET pullRequestThreads`
- **Add**: `POST pullRequestThreads` (creates a new thread with initial comment)
- **Reply**: `POST pullRequestThreadComments` (adds a comment to existing thread)
- **Resolve**: `PATCH pullRequestThreads` (updates thread status)
- **Delete**: `DELETE pullRequestThreadComments` (soft-delete)

JSON payloads are written to temp files and passed via `--in-file` for portability.

## Workflow

```
1. Detect org/project/repo from git remote
        |
2. Get repository ID (required by threads API)
        |
3. Resolve PR ID (from args or current branch)
        |
4. Route to action:
   |--- List: fetch and display all threads
   |--- Add: create new thread (general or inline)
   |--- Reply: add comment to existing thread
   |--- Resolve: update thread status
   |--- Delete: soft-delete a comment
```

## Auto-Detection

The skill automatically detects:

- **Org, project, repo** from the git remote URL
- **Repository ID** via `az repos show`
- **PR ID** from the current branch's active PR (if not provided)

No manual configuration needed beyond Azure CLI authentication.

## Integration with Other Skills

```
/azure-work --> pick up and implement work item
      |
/azure-pr --> push and create PR
      |
/azure-pr-comments --> manage review feedback
      |
/pr-complete --> post-merge cleanup
```

## Compatibility

Built for **Claude Code** but uses the [Agent Skills](https://agentskills.io) open standard (`SKILL.md` with YAML frontmatter), so the core workflow also works on Codex CLI, Antigravity, and other compatible tools.

- No Claude Code-specific skill dependencies -- all functionality uses `az` CLI and `jq` directly.
- On Codex CLI, install to `.agents/skills/` instead of `.claude/skills/`.

## Notes

- All operations use `az devops invoke` -- no native `az repos pr` comment commands exist
- Thread status values are numeric (1-6), not strings
- System-generated threads are filtered out when listing (only shows user comments)
- File paths for inline comments must start with `/` and be repo-root-relative
- Deleted comments are soft-deleted (marked as deleted, not physically removed)
- Max 500 comments per thread
- Requires `jq` for JSON parsing
