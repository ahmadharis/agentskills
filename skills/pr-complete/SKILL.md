---
name: pr-complete
description: >
  Post-merge cleanup after a PR has been merged. Switches to the default branch,
  pulls latest, and deletes all local branches whose remote tracking branch is gone.
  Use when the user says a PR was merged, or after confirming a merge in Azure DevOps
  or GitHub. Works with any git repository by auto-detecting the default branch.
---

# Post-Merge Cleanup

Run these steps in order after a PR has been merged into the default branch.

## Step 1: Detect the default branch

```bash
DEFAULT_BRANCH=$(git remote show origin | sed -n 's/.*HEAD branch: //p')
```

If the command fails or returns empty, fall back to `main`.

## Step 2: Switch to the default branch and pull latest

```bash
git checkout "$DEFAULT_BRANCH"
git pull
```

## Step 3: Prune remote tracking branches and delete stale local branches

Fetch with prune to remove remote tracking refs that no longer exist:

```bash
git fetch --prune
```

Then find and delete all local branches whose upstream is gone:

```bash
GONE_BRANCHES=$(git branch -vv | grep ': gone]' | awk '{print $1}')
if [ -n "$GONE_BRANCHES" ]; then
  echo "$GONE_BRANCHES" | xargs git branch -D
fi
```

## Step 4: Confirm

Report to the user:
- Which branch you're now on
- Which branches were deleted (if any)
- That the workspace is up to date

## Notes

- This command does NOT push, commit, or modify any files. It is read-only cleanup.
- If there are uncommitted changes on the current branch, warn the user before switching.
- Always auto-detect the default branch from the remote rather than assuming `main`.
