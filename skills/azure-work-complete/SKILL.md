---
name: azure-work-complete
description: >
  Full autonomous development cycle: picks up a work item, implements it,
  reviews the code, creates a PR, and cleans up the branch.
  Use when you want the complete workflow end-to-end, or pair with /loop
  for continuous development. Also triggers on "run the full cycle",
  "pick up and ship a work item", or "autonomous mode".
---

# Azure DevOps Full Cycle Workflow

Orchestrates the complete development cycle: pick up a work item, implement it, review the code, create a PR, and clean up. Designed to run autonomously with minimal user intervention.

## Invocation

| Mode | Command | Behavior |
|------|---------|----------|
| Single run | `/azure-work-complete` | Run one full cycle |
| Continuous | `/loop 30m /azure-work-complete` | Repeat on a schedule (session-scoped, 3-day max) |

> **Context in loop mode:** Each `/loop` iteration runs in the same session. If context grows large, use `/compact` between cycles or run in headless mode (`claude -p "/azure-work-complete"`) for full isolation per cycle.

---

## Step 1: Pick Up a Work Item

Invoke the `azure-work` skill in **browse mode** (no arguments).

The skill will list open work items. **Automatically pick the first item from the list** — do not prompt the user for selection.

**If the list is too large:** If `azure-work` indicates there are too many items to display, or returns an error about result limits, or the list appears truncated, re-run the WIQL query from `azure-work` with `TOP 5` added after `SELECT`:

```
SELECT TOP 5 [System.Id], [System.Title], ...
```

Then pick the first item from the reduced list.

**If no items are returned:** Stop the cycle and report: "No open work items found. Cycle complete." If running in a loop, the next iteration will check again.

---

## Step 2: Implement the Work Item

The `azure-work` skill handles the full implementation:
- Reads work item details and attachments
- Creates a feature branch
- Updates work item state
- Brainstorms or implements directly (based on complexity)
- Runs verification checks

Wait for `azure-work` to complete fully before proceeding.

---

## Step 3: Code Review

After implementation, assess whether the changes need review and simplification.

**When to skip this step:**
- The change is trivial (single-file fix, config change, documentation-only)
- The change is a one-liner or a few-line bug fix with clear intent
- The verification checks from Step 2 already confirmed quality

**When to run this step:**
- Multi-file feature implementation
- Refactoring or architectural changes
- Complex logic added
- Significant amount of new code

### Claude Code (primary)

If `/simplify` is available, invoke it:

```
/simplify
```

This spawns three parallel review agents (code reuse, quality, efficiency), aggregates findings, and applies fixes automatically.

### Codex / Other Platforms (fallback)

If `/simplify` is not available, perform an inline code review. Review all changed files (from `git diff --name-only` against the default branch) for:

1. **Code reuse** — duplicated logic or patterns that should be consolidated into shared functions or modules
2. **Code quality** — naming clarity, consistent style, overly complex conditionals, dead code, unclear intent
3. **Efficiency** — unnecessary allocations, redundant iterations, suboptimal data structures, avoidable I/O

For each issue found, fix it directly. Do not report issues without fixing them. After all fixes are applied, re-run the project's verification checks (build, test, lint) to confirm nothing broke.

---

## Step 4: Create Pull Request

Invoke the `azure-pr` skill with the work item ID from Step 1:

```
/azure-pr --work-items <WORK_ITEM_ID>
```

Wait for the PR to be created. Note the PR URL from the output.

---

## Step 5: Post-Merge Cleanup

> **Important:** This step only applies if the PR was auto-completed (merged automatically). If the PR requires manual review/approval, stop here and report the PR URL to the user. The user can run `/pr-complete` manually after the PR is merged.

If the PR was merged immediately (e.g., auto-complete policies), invoke:

```
/pr-complete
```

This switches to the default branch, pulls latest, prunes remotes, and deletes stale local branches.

---

## Cycle Complete

Report a summary:

```
Cycle complete:
- Work item: #<ID> — <Title>
- Branch: feature/<ID>
- PR: <URL>
- Status: <awaiting review | merged and cleaned up>
```

If running in a `/loop`, the next iteration will begin at the scheduled time and pick up the next available work item.

---

## Notes

- Each cycle is independent — it picks up whatever is first on the backlog at that moment
- Direct mode is not supported for this skill — it always browses and picks the top item
- Work item priority ordering comes from the WIQL query in `azure-work` (priority ASC, ID ASC)
- If any step fails (authentication, no items, build failure), the cycle stops and reports the error
- The `azure-work` skill handles all Azure DevOps interaction — this skill only orchestrates the sequence
