---
name: azure-pr
description: >
  Create pull requests in Azure DevOps. Use when the user asks to create a PR,
  submit changes for review, push a feature branch, or when finishing development
  work that needs to be merged. Also use proactively after completing work on a
  feature branch. Handles the full workflow: branch creation, committing, pushing,
  and PR creation via `az repos pr create`. Works with any Azure DevOps repository
  by extracting org/project/repo from the git remote URL.
---

# Azure DevOps PR Workflow

## Prerequisites

Verify `az` CLI is available before proceeding. Run:

```bash
az --version
```

If the command fails, inform the user how to install it:

- **macOS**: `brew install azure-cli`
- **Windows**: `winget install --exact --id Microsoft.AzureCLI`
- **Linux**: `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`

Then ensure the Azure DevOps extension is installed:

```bash
az extension add --name azure-devops
```

## Detect Repository Info

Extract organization, project, and repository from the git remote:

```bash
# Remote URL format: https://{org}@dev.azure.com/{org}/{project}/_git/{repo}
REMOTE_URL=$(git remote get-url origin)
ORG=$(echo "$REMOTE_URL" | sed -n 's|.*dev.azure.com/\([^/]*\)/.*|\1|p')
PROJECT=$(echo "$REMOTE_URL" | sed -n 's|.*dev.azure.com/[^/]*/\([^/]*\)/.*|\1|p')
REPO=$(echo "$REMOTE_URL" | sed -n 's|.*/_git/\(.*\)|\1|p')
```

Use these values for `--organization`, `--project`, and `--repository` flags below.

## Branch Naming

- `feature/<name>` — new features
- `fix/<name>` — bug fixes
- `refactor/<name>` — refactoring

All development happens through feature branches. Never commit directly to `main`.

## Full Workflow

### 1. Create branch (if not already on one)

```bash
git checkout -b <type>/<short-name>
```

### 2. Stage and commit

```bash
git add <specific files>
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject>

<body>

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### 3. Push branch

```bash
git push -u origin <branch-name>
```

### 4. Create PR

```bash
az repos pr create \
  --title "<type>(<scope>): <subject>" \
  --description "$(cat <<'EOF'
## Summary
- <bullet points describing changes>

## Test plan
- [ ] <verification steps>
EOF
)" \
  --source-branch <branch-name> \
  --target-branch main \
  --organization "https://dev.azure.com/$ORG" \
  --project "$PROJECT" \
  --repository "$REPO"
```

The command returns JSON. Extract `pullRequestId` to form the PR URL:
`https://dev.azure.com/$ORG/$PROJECT/_git/$REPO/pullrequest/<pullRequestId>`

### 5. Link work items (if provided)

If the user provides work item IDs — either explicitly with `--work-items` or just as bare numbers (e.g., `12345`, `#12345`) — add the `--work-items` flag to the `az repos pr create` command:

```bash
az repos pr create \
  ... \
  --work-items <id1> <id2>
```

**Detection rules**:
- `--work-items 12345 67890` — explicit flag, use as-is
- `12345` or `#12345` — bare number in user input, treat as work item ID
- Strip any `#` prefix before passing to `--work-items`
- Multiple IDs are space-separated: `--work-items 12345 67890`

## Notes

- Keep PR title under 70 characters. Use the description for details.
- Use HEREDOC (`cat <<'EOF'`) for commit messages and PR descriptions to handle special characters.
- Default target branch is `main`. Override with `--target-branch` if different.
