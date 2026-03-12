# azure-work-complete

Full autonomous development cycle skill for Claude Code. Picks up a work item from your Azure DevOps backlog, implements it, reviews the code, creates a PR, and cleans up — all in one command.

## What It Does

Orchestrates the complete development workflow by chaining four skills in sequence:

1. **azure-work** — browse open work items, auto-pick the first one, implement it
2. **/simplify** (or inline review) — assess and review code quality, reuse, and efficiency
3. **azure-pr** — commit, push, and create a PR with work item linking
4. **pr-complete** — return to default branch and clean up

## Prerequisites

- Everything required by [azure-work](../azure-work/README.md) (Azure CLI, authentication, configuration)
- Everything required by [azure-pr](../azure-pr/README.md) (Git, Azure DevOps remote)

No additional configuration needed — this skill reuses the existing setup.

## Usage

### Single Run

```
/azure-work-complete
```

When installed as a plugin:

```
/agentskills:azure-work-complete
```

### Continuous Mode (with /loop)

Pair with Claude Code's `/loop` command to continuously pick up and ship work items:

```
/loop 30m /azure-work-complete
```

This runs the full cycle every 30 minutes. Each iteration picks up the next available work item from the backlog.

**Loop behavior:**
- Session-scoped — stops when you close the terminal
- 3-day maximum — auto-expires after 3 days
- No catch-up — if a cycle is still running when the next is due, it waits
- Use `/compact` between cycles if context grows large

To stop the loop: say "cancel the loop" or exit the session.

### Headless Mode

For full context isolation per cycle, run in headless mode:

```bash
claude -p "/azure-work-complete"
```

Each invocation starts a fresh session with no prior context.

## How It Picks Work Items

The skill auto-selects the **first work item** from the backlog (sorted by priority then ID). No user prompt — it just picks the top item and starts.

If the backlog returns too many items (truncated or errored), it retries with a `TOP 5` WIQL limit.

If no items are available, the cycle ends cleanly.

## Code Review Step

The skill assesses whether code review is needed based on change complexity:

- **Trivial changes** (one-liner, config, docs) — review is skipped
- **Substantial changes** (multi-file, new logic, refactoring) — review runs

On **Claude Code**, this invokes the built-in `/simplify` skill (three parallel review agents for reuse, quality, efficiency).

On **Codex / other platforms**, an inline review pass covers the same three dimensions and applies fixes directly.

## Workflow Diagram

```
/azure-work-complete
      ↓
1. /azure-work → auto-pick first item, implement
      ↓
2. Assess → /simplify or inline review (if needed)
      ↓
3. /azure-pr --work-items <id> → push and create PR
      ↓
4. /pr-complete → return to default branch, clean up
      ↓
   Cycle complete — ready for next item
```

## Compatibility

| Platform | Support |
|----------|---------|
| Claude Code | Full support — uses `/simplify` for code review |
| Claude Code + /loop | Full support — continuous autonomous mode |
| Codex CLI | Core workflow works. `/simplify` replaced by inline review. Skill chaining may require manual steps. |
| Antigravity / Others | Same as Codex |

## Integration

This skill composes:
- **azure-work** — work item pickup and implementation
- **azure-pr** — PR creation with work item linking
- **pr-complete** — post-merge cleanup
- **/simplify** — built-in Claude Code code review (or inline fallback)
