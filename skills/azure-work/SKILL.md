---
name: azure-work
description: >
  Use when starting work from Azure DevOps board items, picking up backlog items,
  or when given a specific work item number to implement.
  Also use when the user says "pick up a work item", "what's on the board",
  or "work on item 12345".
---

# Azure DevOps Work Item Workflow

Pick up and implement Azure DevOps work items. Browse open items or work directly on a specific item by number.

## Invocation

| Mode | Command | Behavior |
|------|---------|----------|
| Browse | `/azure-work` | List open work items, user picks one |
| Direct | `/azure-work <number>` | Go straight to that work item |

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

Then ensure the Azure DevOps extension is installed:

```bash
az extension add --name azure-devops
```

---

## Setup (First Run Only)

On first invocation, the skill needs Azure CLI authentication and a backlog URL.

### 1. Verify Azure CLI Authentication

```bash
az account show
```

If not authenticated, run `az login --allow-no-subscriptions` which opens the browser for Azure AD OAuth. No PAT tokens or stored credentials are needed.

### 2. Check for Configuration

Check if the `ADO_BACKLOG_URL` environment variable is set. If it IS set, `ADO_ORG` and `ADO_PROJECT` should also be available (and `ADO_TEAM` if the URL contained a team) — skip to the Workflow section.

If `ADO_BACKLOG_URL` is NOT set, prompt the user:

> "What's your Azure DevOps backlog URL?"
> (e.g., `https://dev.azure.com/org/project/_backlogs/backlog/team/Backlog%20items`)

**Note:** The URL may or may not include a team segment. Both formats are valid:
- With team: `https://dev.azure.com/org/project/_backlogs/backlog/team/Backlog%20items`
- Without team: `https://dev.azure.com/org/project/_backlogs/backlog/Backlog%20items`
- Board URL also accepted: `https://dev.azure.com/org/project/_boards/board/t/team/Backlog%20items`

### 3. Extract Org, Project, and Team from URL

Works with both backlogs and boards URLs:

```
https://dev.azure.com/{org}/{project}/_backlogs/backlog/{team}/Backlog%20items
https://dev.azure.com/{org}/{project}/_backlogs/backlog/Backlog%20items
https://dev.azure.com/{org}/{project}/_boards/board/t/{team}/Backlog%20items
```

```bash
ORG=$(echo "$URL" | sed -n 's|.*dev.azure.com/\([^/]*\)/.*|\1|p')
PROJECT=$(echo "$URL" | sed -n 's|.*dev.azure.com/[^/]*/\([^/]*\)/.*|\1|p')
```

**Extract team (if present).** The team is the segment after `_backlogs/backlog/` that is NOT a backlog level name. Backlog level names include: `Backlog%20items`, `Stories`, `Features`, `Epics`, `Bugs`. If the segment after `_backlogs/backlog/` is none of those, it is a team name. For board URLs, the team is the segment after `_boards/board/t/`.

```bash
# For backlog URLs: segment after _backlogs/backlog/ that isn't a backlog level
TEAM=$(echo "$URL" | sed -n 's|.*_backlogs/backlog/\([^/]*\)/.*|\1|p' | python3 -c "import sys,urllib.parse; t=urllib.parse.unquote(sys.stdin.read().strip()); print(t if t and t not in ['Backlog items','Stories','Features','Epics','Bugs'] else '')")

# For board URLs: segment after _boards/board/t/
if [ -z "$TEAM" ]; then
  TEAM=$(echo "$URL" | sed -n 's|.*_boards/board/t/\([^/]*\)/.*|\1|p' | python3 -c "import sys,urllib.parse; print(urllib.parse.unquote(sys.stdin.read().strip()))")
fi
```

If `TEAM` is empty, that's fine — the skill runs project-wide. If `TEAM` has a value, save it as `ADO_TEAM`.

### 4. Store Configuration

Read the existing `.claude/settings.local.json` file (or start with `{}`), and add/update the `env` key with the backlog URL and extracted org/project. Use a read-modify-write pattern to preserve any existing settings:

```json
{
  "env": {
    "ADO_BACKLOG_URL": "https://dev.azure.com/org/project/_backlogs/backlog/team/Backlog%20items",
    "ADO_ORG": "org",
    "ADO_PROJECT": "project",
    "ADO_TEAM": "team"
  }
}
```

Only include `ADO_TEAM` if a team was extracted from the URL. Omit it entirely when no team is present — the skill will query project-wide.

Tell the user: "Configuration saved to `.claude/settings.local.json`. These values will be loaded automatically on future runs. Please restart Claude Code for the environment variables to take effect."

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ADO_BACKLOG_URL` | Yes (prompted on first run) | — | Full Azure DevOps backlog URL |
| `ADO_ORG` | Auto-extracted | from URL | Azure DevOps organization name |
| `ADO_PROJECT` | Auto-extracted | from URL | Azure DevOps project name |
| `ADO_TEAM` | Auto-extracted | from URL (if team is in URL) | Azure DevOps team name (omitted for project-wide queries) |
| `ADO_WORK_ITEM_FILTER` | No | *(none — lists all)* | Tag to filter work items by (e.g. `"claude"`) |

- When `ADO_TEAM` is set → fetches the team's area paths and scopes the WIQL query (see Step 2)
- When `ADO_TEAM` is unset → queries project-wide (all teams)
- When `ADO_WORK_ITEM_FILTER` is unset → lists ALL open work items (within team scope if set)
- When `ADO_WORK_ITEM_FILTER` is set → adds `AND [System.Tags] CONTAINS '<value>'` to the WIQL query
- Configuration is always per-project via `.claude/settings.local.json`

---

## Workflow

### Step 1: Verify Configuration

Check that `ADO_ORG` and `ADO_PROJECT` environment variables are set. If missing, run Setup.

**Auto-populate `ADO_TEAM` (one-time migration):** If `ADO_BACKLOG_URL` is set but `ADO_TEAM` is NOT set as an environment variable at all, extract the team from the URL using the logic from Setup Step 3. Save the result to `.claude/settings.local.json` (read-modify-write to preserve existing settings):
- If a team is found: save `"ADO_TEAM": "teamname"` and inform the user: "Detected team '{team}' from your backlog URL. Saved to configuration."
- If no team is in the URL: save `"ADO_TEAM": ""` so this check does not run again.

Use the extracted team value (if any) for the rest of this session even before restart.

Verify authentication:

```bash
az account show
```

### Step 2: List or Fetch Work Items

**Browse mode** (`/azure-work` with no args):

Query open work items using WIQL.

#### 2a. Build area path filter (if `ADO_TEAM` is set)

If `ADO_TEAM` is set, fetch the team's configured area paths:

```bash
az boards area team list --team "$ADO_TEAM" --project "$ADO_PROJECT" --org "https://dev.azure.com/$ADO_ORG" -o json
```

This returns an array of area path objects, each with:
- `value` — the area path string (e.g., `"MyProject\\TeamArea"`)
- `includeChildren` — boolean indicating whether child area paths are included

Build an area path WHERE clause from the result:
- If `includeChildren` is `true`: use `[System.AreaPath] UNDER 'value'`
- If `includeChildren` is `false`: use `[System.AreaPath] = 'value'`
- Combine multiple entries with `OR`, wrapped in parentheses

**Example:** If the team has two area paths:
```json
[
  {"value": "MyProject\\Web", "includeChildren": true},
  {"value": "MyProject\\Shared", "includeChildren": false}
]
```

The clause becomes:
```
AND ([System.AreaPath] UNDER 'MyProject\Web' OR [System.AreaPath] = 'MyProject\Shared')
```

If `ADO_TEAM` is NOT set, skip this — no area path clause is added.

#### 2b. Execute the WIQL query

Construct the query by combining the base conditions with optional area path and tag filters:

**Base query (no team, no filter):**

```bash
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType] FROM WorkItems WHERE [System.State] <> 'Done' AND [System.State] <> 'Closed' AND [System.State] <> 'Removed' ORDER BY [Microsoft.VSTS.Common.Priority] ASC, [System.Id] ASC" --org "https://dev.azure.com/$ADO_ORG" --project "$ADO_PROJECT"
```

**With team scope:** append the area path clause from Step 2a before `ORDER BY`.

**With tag filter (when `ADO_WORK_ITEM_FILTER` is set):** append `AND [System.Tags] CONTAINS '$ADO_WORK_ITEM_FILTER'` before `ORDER BY`.

**Both filters can be combined.** Build the WHERE clause additively:

```
WHERE [System.State] <> 'Done'
  AND [System.State] <> 'Closed'
  AND [System.State] <> 'Removed'
  AND (<area path clause>)           -- only if ADO_TEAM is set
  AND [System.Tags] CONTAINS '...'   -- only if ADO_WORK_ITEM_FILTER is set
ORDER BY [Microsoft.VSTS.Common.Priority] ASC, [System.Id] ASC
```

Display results as a table:

```
| ID    | Type       | Priority | State | Title                    |
|-------|------------|----------|-------|--------------------------|
| 12345 | User Story | 1        | New   | Add export feature       |
| 12346 | Bug        | 2        | New   | Fix login timeout        |
```

Then ask: **"Enter the work item number to start working on it."**

The user will reply with just a number (e.g., `12345`). Use that as the work item ID.

**Direct mode** (`/azure-work 12345`):

Skip listing, go straight to Step 3 with the provided ID.

### Step 3: Read Work Item Details

Fetch the full work item with all fields and relations:

```bash
az boards work-item show --id $WORK_ITEM_ID --org "https://dev.azure.com/$ADO_ORG" --expand all
```

Extract and present to the user:

| Field | JSON Path |
|-------|-----------|
| Title | `fields["System.Title"]` |
| Description | `fields["System.Description"]` (HTML → convert to readable text) |
| Acceptance Criteria | `fields["Microsoft.VSTS.Common.AcceptanceCriteria"]` |
| State | `fields["System.State"]` |
| Type | `fields["System.WorkItemType"]` |
| Tags | `fields["System.Tags"]` |
| Assigned To | `fields["System.AssignedTo"]` |
| Priority | `fields["Microsoft.VSTS.Common.Priority"]` |
| Repro Steps (bugs) | `fields["Microsoft.VSTS.TCM.ReproSteps"]` |

Also display any non-empty custom fields that may contain additional requirements.

### Step 4: Download and Read All Attachments

From the work item's `relations` array, find entries where `rel` is `"AttachedFile"`. Download each using an authenticated request:

```bash
TOKEN=$(az account get-access-token --resource "499b84ac-1321-427f-aa17-267ca6975798" --query accessToken -o tsv)
curl -s -H "Authorization: Bearer $TOKEN" -o "/tmp/$FILENAME" "$ATTACHMENT_URL"
```

**Attempt to read ALL attachment types:**

| Type | Approach |
|------|----------|
| Text, Markdown, Code | Read tool directly |
| Images (PNG, JPG, GIF) | Read tool (Claude is multimodal) |
| PDFs | Read tool with `pages` parameter |
| Word docs (.docx) | Invoke `document-skills:docx` if available, otherwise note for user |
| Spreadsheets (.xlsx) | Invoke `document-skills:xlsx` if available, otherwise note for user |
| Other | Note the filename and type, inform the user |

**Read every attachment before proceeding to implementation.** Attachments often contain mockups, specs, or requirements not captured in the description.

### Step 5: Create Feature Branch

```bash
# Ensure we're on the latest default branch
DEFAULT_BRANCH=$(git remote show origin | sed -n 's/.*HEAD branch: //p')
git checkout "$DEFAULT_BRANCH"
git pull
git checkout -b "feature/$WORK_ITEM_ID"
```

### Step 6: Update Work Item State

The valid "in progress" state depends on the work item type:

| Work Item Type | "In Progress" State |
|----------------|---------------------|
| Product Backlog Item | `Approved` |
| Bug | `Approved` |
| Task | `In Progress` |

```bash
# For Product Backlog Items and Bugs:
az boards work-item update --id $WORK_ITEM_ID --state "Approved" --org "https://dev.azure.com/$ADO_ORG"

# For Tasks:
az boards work-item update --id $WORK_ITEM_ID --state "In Progress" --org "https://dev.azure.com/$ADO_ORG"
```

Skip this step if the work item is already in the target state.

### Step 7: Decide Implementation Approach

**LLM Decision Point** — Assess the work item and decide:

**Brainstorm first when:**
- New feature with vague or high-level description
- Multiple valid implementation approaches exist
- Requirements are ambiguous or incomplete
- Significant architectural decisions needed
- Work item type is "Epic" or "Feature"

If `superpowers:brainstorming` is available, invoke it. Otherwise, brainstorm the approach directly: explore the codebase, identify affected areas, consider trade-offs, and present options to the user before implementing.

**Go straight to implementation when:**
- Bug fix with clear reproduction steps
- Detailed acceptance criteria already provided
- Implementation approach is obvious or prescribed
- Small, well-scoped change
- Work item type is "Bug" or "Task" with clear instructions

If `feature-dev:feature-dev` is available, invoke it for guided implementation. Otherwise, proceed directly: explore the codebase, implement the changes, and write tests.

**Pass the work item context** (title, description, acceptance criteria, attachment contents) as input to whichever approach is used.

### Step 8: Verification

After implementation is complete:

1. **Run project standard checks** (e.g., typecheck, build, test — adapt to the project's tooling).

2. **Verify before claiming done.** If `superpowers:verification-before-completion` is available, invoke it. Otherwise, re-run checks and confirm all pass with fresh output. Do not claim work is done without evidence.

3. **Report results to the user.** Include:
   - What was implemented
   - Verification results
   - Branch name (`feature/<id>`)
   - Next step suggestion: "Run `/azure-pr --work-items <id>` to create a PR"

**Do NOT mark the work item as Done or Resolved.** That happens at PR closure via a separate workflow.

---

## Notes

- The `azure-pr` skill handles PR creation and can link the work item via `--work-items`
- The `pr-complete` skill handles post-merge cleanup
- Work item state progression: **New → Active** (this skill) → **Resolved/Done** (PR closure)
- If the user replies with just a number after the listing, treat it as the work item ID to pick up
- Configuration persists across sessions — setup only runs once per project
