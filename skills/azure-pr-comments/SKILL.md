---
name: azure-pr-comments
description: >
  Manage pull request comments and review threads in Azure DevOps.
  Use when the user wants to list, add, reply to, resolve, or delete
  PR comments. Also use when the user says "check PR comments",
  "respond to review feedback", "resolve comment thread",
  "add a comment to the PR", or "what feedback is on my PR".
---

# Azure DevOps PR Comments

List, add, reply to, resolve, and delete comment threads on Azure DevOps pull requests.

## Invocation

| Mode | Command | Behavior |
|------|---------|----------|
| List | `/azure-pr-comments` | List threads on current branch's PR |
| List | `/azure-pr-comments <pr-id>` | List threads on specified PR |
| Add | `/azure-pr-comments add` | Add a comment to current PR |
| Reply | `/azure-pr-comments reply` | Reply to a thread |
| Resolve | `/azure-pr-comments resolve` | Resolve/close a thread |

The LLM determines the PR ID, thread ID, and content from context. Natural language also works:

- "What comments are on my PR?"
- "Add a comment to PR 123: looks good"
- "Reply to thread 5 on PR 123: fixed in latest commit"
- "Resolve all threads on my PR"
- "Mark thread 5 as won't fix"

---

## Prerequisites

Verify `az` CLI is available before proceeding. Run:

```bash
az --version
```

If the command fails, inform the user how to install it:

- **macOS**: `brew install azure-cli`
- **Windows**: `winget install --exact --id Microsoft.AzureCLI`
- **Linux**: `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`

**Windows shell requirement:** All commands in this skill use bash syntax (`sed`, `mktemp`, `jq`, etc.). On Windows, run in **Git Bash** or **WSL** — PowerShell and cmd are not supported.

Then ensure the Azure DevOps extension is installed:

```bash
az extension add --name azure-devops
```

---

## Detect Repository Info

Extract organization, project, and repository from the git remote:

```bash
REMOTE_URL=$(git remote get-url origin)
ORG=$(echo "$REMOTE_URL" | sed -n 's|.*dev.azure.com/\([^/]*\)/.*|\1|p')
PROJECT=$(echo "$REMOTE_URL" | sed -n 's|.*dev.azure.com/[^/]*/\([^/]*\)/.*|\1|p')
REPO=$(echo "$REMOTE_URL" | sed -n 's|.*/_git/\(.*\)|\1|p')
```

Then get the repository ID (required by the threads API):

```bash
REPO_ID=$(az repos show --repository "$REPO" --org "https://dev.azure.com/$ORG" --project "$PROJECT" --query "id" -o tsv)
```

---

## Resolve PR ID

If the user provides a PR ID, use it directly. Otherwise, detect the PR for the current branch:

```bash
BRANCH=$(git branch --show-current)
PR_ID=$(az repos pr list \
  --source-branch "$BRANCH" \
  --org "https://dev.azure.com/$ORG" \
  --project "$PROJECT" \
  --repository "$REPO" \
  --status active \
  -o json | jq '.[0].pullRequestId')
```

If no active PR is found for the current branch, inform the user and ask for a PR ID.

---

## API Notes

The `az repos pr` CLI has **no native subcommands for comments or threads**. All operations use `az devops invoke` which wraps the Azure DevOps REST API.

**Common parameters for all calls:**

```bash
az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource <resource> \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
  --api-version 7.1
```

**Thread status values (numeric):**

| Value | Status | Meaning |
|-------|--------|---------|
| 1 | Active | Open, needs attention |
| 2 | Fixed | Resolved as fixed |
| 3 | Won't Fix | Resolved as won't fix |
| 4 | Closed | Closed |
| 5 | By Design | Resolved as by design |
| 6 | Pending | Pending resolution |

---

## Workflow

### Mode Routing

After detecting repo info and resolving the PR ID, follow the path matching the user's intent:

| Intent | Go to |
|--------|-------|
| List / view comments | [List Threads](#list-threads) |
| Add a comment | [Add Comment](#add-comment) |
| Reply to a thread | [Reply to Thread](#reply-to-thread) |
| Resolve / close a thread | [Resolve Thread](#resolve-thread) |
| Delete a comment | [Delete Comment](#delete-comment) |

---

### List Threads

Fetch all comment threads on the PR:

```bash
az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource pullRequestThreads \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
  --http-method GET \
  --api-version 7.1 \
  -o json
```

The response contains `value` -- an array of thread objects. Each thread has:
- `id` -- thread ID
- `status` -- numeric status (see table above)
- `comments[]` -- array of comments, each with `id`, `content`, `author.displayName`
- `threadContext` -- file context if it's an inline comment (`filePath`, line numbers)

**Filter out system threads.** Threads where all comments have `commentType` of `0` (unknown) or `2` (codeChange) or `3` (system) are system-generated. Only show threads with at least one `commentType: 1` (text) comment.

**Display as a table:**

```
PR #<pr-id> Comment Threads:

| Thread | Status | File | Comment | Author |
|--------|--------|------|---------|--------|
| #1 | Active | src/main.ts:15 | "Should we add error handling here?" | Jane Doe |
| #2 | Fixed | — (general) | "LGTM!" | John Smith |
| #3 | Active | src/utils.ts:42 | "This could be simplified" | Jane Doe |
```

For threads with multiple comments, show the first comment and note the reply count (e.g., "+2 replies").

---

### Add Comment

Add a new comment thread to the PR. Determine from context whether this is a general comment or an inline file comment.

#### General Comment (PR-level)

Write the JSON body to a temp file and invoke:

```bash
TMPFILE=$(mktemp "${TMPDIR:-/tmp}/ado-thread-XXXXXX.json")
jq -n --arg content "$COMMENT_TEXT" \
  '{comments: [{parentCommentId: 0, content: $content, commentType: 1}], status: 1}' \
  > "$TMPFILE"

az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource pullRequestThreads \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
  --http-method POST \
  --in-file "$TMPFILE" \
  --api-version 7.1 \
  -o json

rm -f "$TMPFILE"
```

#### Inline Comment (File-level)

When the user specifies a file and line number:

```bash
TMPFILE=$(mktemp "${TMPDIR:-/tmp}/ado-thread-XXXXXX.json")
jq -n --arg content "$COMMENT_TEXT" --arg path "/$FILE_PATH" --argjson line "$LINE_NUMBER" \
  '{comments: [{parentCommentId: 0, content: $content, commentType: 1}], status: 1, threadContext: {filePath: $path, rightFileStart: {line: $line, offset: 1}, rightFileEnd: {line: $line, offset: 1}}}' \
  > "$TMPFILE"

az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource pullRequestThreads \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
  --http-method POST \
  --in-file "$TMPFILE" \
  --api-version 7.1 \
  -o json

rm -f "$TMPFILE"
```

**Note:** The `filePath` must start with `/` and be relative to the repo root.

Display the result:

```
Added comment to PR #<pr-id> (thread #<thread-id>):
  "<comment text>"
```

---

### Reply to Thread

Reply to an existing comment thread. The user must provide or you must determine:
- Thread ID (from listing, or from context)
- Reply content

Find the parent comment ID (usually `1` for the first comment in the thread):

```bash
TMPFILE=$(mktemp "${TMPDIR:-/tmp}/ado-reply-XXXXXX.json")
jq -n --arg content "$REPLY_TEXT" \
  '{content: $content, parentCommentId: 1, commentType: 1}' \
  > "$TMPFILE"

az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource pullRequestThreadComments \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
    threadId="$THREAD_ID" \
  --http-method POST \
  --in-file "$TMPFILE" \
  --api-version 7.1 \
  -o json

rm -f "$TMPFILE"
```

Display the result:

```
Replied to thread #<thread-id> on PR #<pr-id>:
  "<reply text>"
```

---

### Resolve Thread

Change a thread's status. Common actions:

| User says | Status value |
|-----------|-------------|
| "resolve" / "mark as fixed" | 2 (Fixed) |
| "won't fix" / "not an issue" | 3 (Won't Fix) |
| "close" | 4 (Closed) |
| "by design" / "intentional" | 5 (By Design) |
| "reopen" / "reactivate" | 1 (Active) |

```bash
TMPFILE=$(mktemp "${TMPDIR:-/tmp}/ado-status-XXXXXX.json")
jq -n --argjson status "$STATUS_VALUE" \
  '{status: $status}' \
  > "$TMPFILE"

az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource pullRequestThreads \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
    threadId="$THREAD_ID" \
  --http-method PATCH \
  --in-file "$TMPFILE" \
  --api-version 7.1 \
  -o json

rm -f "$TMPFILE"
```

**Resolve all active threads:** If the user says "resolve all threads", first list all threads, filter for status `1` (active) with `commentType: 1` comments, then update each one:

```bash
# Get all active thread IDs
THREAD_IDS=$(az devops invoke ... --http-method GET -o json | jq '[.value[] | select(.status == 1) | select(.comments[] | .commentType == 1) | .id] | unique[]')

# Resolve each
for THREAD_ID in $THREAD_IDS; do
  # update status to 2 (fixed)
done
```

Display the result:

```
Resolved thread #<thread-id> on PR #<pr-id>: Active → Fixed
```

Or for bulk:

```
Resolved 3 threads on PR #<pr-id>: #1, #3, #5
```

---

### Delete Comment

Soft-delete a specific comment. Requires both the thread ID and comment ID:

```bash
az devops invoke \
  --org "https://dev.azure.com/$ORG" \
  --area git \
  --resource pullRequestThreadComments \
  --route-parameters \
    project="$PROJECT" \
    repositoryId="$REPO_ID" \
    pullRequestId="$PR_ID" \
    threadId="$THREAD_ID" \
    commentId="$COMMENT_ID" \
  --http-method DELETE \
  --api-version 7.1
```

**Note:** This is a soft delete -- the comment is marked as deleted but not physically removed.

Display the result:

```
Deleted comment #<comment-id> from thread #<thread-id> on PR #<pr-id>
```

---

## Notes

- All comment operations use `az devops invoke` -- there are no native `az repos pr` subcommands for comments
- Thread status values are numeric (1-6), not strings
- File paths in `threadContext` must start with `/` and be repo-root-relative
- Deleted comments are soft-deleted (marked, not removed)
- Max 500 comments per thread
- `commentType: 1` is a regular text comment; filter out system threads (types 0, 2, 3) when displaying
- The `--in-file` flag is required for POST/PATCH bodies; use temp files for portability
- Repository ID is required (not repository name) for the threads API
- The `azure-pr` skill handles PR creation; this skill manages the comment lifecycle after creation
