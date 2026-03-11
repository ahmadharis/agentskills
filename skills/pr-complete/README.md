# pr-complete

Post-merge cleanup skill for Claude Code. Switches to the default branch, pulls latest changes, and removes stale local branches after a PR has been merged.

## What It Does

After a pull request is merged, your local repository still has the old feature branch and may have stale remote tracking references. This skill automates the cleanup:

1. **Detects** the default branch from the remote (doesn't assume `main`)
2. **Switches** to the default branch and pulls the latest changes
3. **Prunes** remote tracking references that no longer exist
4. **Deletes** local branches whose upstream tracking branch is gone

## When It Triggers

Claude will use this skill when you say things like:
- "The PR was merged"
- "PR is complete"
- "Clean up after merge"
- After confirming a merge in Azure DevOps or GitHub

## Prerequisites

- **Git** — that's it. No additional tools or extensions required.

## Usage

```
/pr-complete
```

When installed as a plugin, use the namespaced command:

```
/agentskills:pr-complete
```

No arguments needed. The skill auto-detects everything from your git repository.

## Workflow

```
1. Detect default branch (from remote, not hardcoded)
        ↓
2. Check for uncommitted changes (warns if any)
        ↓
3. git checkout <default-branch> && git pull
        ↓
4. git fetch --prune
        ↓
5. Find branches with gone upstream → git branch -D
        ↓
6. Report what was cleaned up
```

## What Gets Deleted

- Local branches whose remote tracking branch no longer exists (shows `[gone]` in `git branch -vv`)

## What Does NOT Get Deleted

- The default branch (main/master/develop)
- Local branches that were never pushed
- Local branches whose remote still exists
- Uncommitted changes (you'll be warned first)

## Example Output

```
Switched to main and pulled latest.
Pruned remote tracking branches.

Deleted branches:
  - feature/12345
  - fix/login-timeout

Now on: main (up to date)
```

## Compatibility

Built for **Claude Code** but uses the [Agent Skills](https://agentskills.io) open standard, so it also works on Codex CLI, Antigravity, and other compatible tools.

- On Codex CLI, install to `.agents/skills/` instead of `.claude/skills/`.
- No platform-specific dependencies — uses only `git`.

## Safety

This skill is **read-only cleanup** — it does not push, commit, or modify any files. The only destructive action is deleting local branches that have already been merged and removed from the remote.
