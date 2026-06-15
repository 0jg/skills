---
name: github-projects
description: >
  Manages GitHub Projects v2 Kanban boards for research repositories within the
  QuantumSensing organisation. Use this skill whenever starting a new coding session,
  planning work, creating tasks, moving issues between columns, or linking PRs to
  issues. Triggers on: "start a session", "plan the work", "what are we doing today",
  "create issues", "update the board", "move to done", or any task breakdown request.
  Always use this skill at session start to plan and at session end to close out.
  All API calls use GraphQL against the GitHub Projects v2 API.
---

# GitHub Projects Skill

## Overview

Every repository in the `QuantumSensing` organisation has exactly one associated
GitHub Project. The project name matches the repository name. The Kanban board has
five columns driven by the `Status` single-select field:

```
Backlog → Todo → In Progress → In Review → Done
```

Issues are the unit of work. PRs are linked to issues. Claude moves issues through
the board automatically as work progresses. The human approves the task breakdown
before coding starts.

---

## Fork mode

A forked repository within `QuantumSensing` is treated as an independent project.
It has its own issues, its own project board, and its own labels. PRs target the
fork's own `main` — not the upstream. There is no assumption of eventual upstream
contribution.

### Detecting fork mode

At session start (or project init), check whether the repo is a fork:

```bash
curl -s \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/QuantumSensing/{repo}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('fork', False), d.get('parent', {}).get('full_name', ''))"
```

If the first token is `True`, this is a fork. Store:

```
IS_FORK = true
UPSTREAM_REPO = <parent full_name>   # e.g. QuantumSensing/original-repo
```

### Fork mode differences

| Behaviour | Greenfield | Fork |
|---|---|---|
| Issues | Created on this repo | Created on this repo |
| Project board | Created on this repo | Created on this repo |
| Labels | Created fresh | Created fresh (replace defaults) |
| PR target | `main` on this repo | `main` on this repo |
| Upstream awareness | N/A | Shown at session start |

### Upstream divergence notice

At every session start in fork mode, surface upstream status:

```bash
git fetch upstream 2>/dev/null || git remote add upstream \
  https://github.com/{UPSTREAM_REPO}.git && git fetch upstream

BEHIND=$(git rev-list --count HEAD..upstream/main 2>/dev/null || echo "unknown")
```

Include in the session start message:

```
[Fork mode] Upstream: QuantumSensing/<original>
This fork is <N> commits behind upstream/main.
PRs target this fork's main — not upstream.
```

If `BEHIND` is large (> 20), flag it to the human but do not take action.

### Project init for a fork

When initialising a forked repo, run the full project initialisation procedure
(labels, project board, column configuration, repo link) on the fork exactly as
for a greenfield repo. Do not interact with the upstream project board.

Add the upstream remote if not already present:

```bash
git remote add upstream https://github.com/{UPSTREAM_REPO}.git
```

---

## Token requirements

`GITHUB_TOKEN` must have:
- **Classic PAT**: `repo` + `project` scopes
- **Fine-grained PAT**: Repository permissions: Issues (read/write), Pull requests
  (read/write); Organisation permissions: Projects (read/write)

The `project` scope is required for all GraphQL project mutations. If missing, all
project API calls will return 401 or silently fail.

---

## Organisation constants

```
OWNER_TYPE = organisation
ORG        = QuantumSensing
```

When resolving owner for REST calls, always use `QuantumSensing`. When resolving
project ownership in GraphQL, query via `organization(login: "QuantumSensing")`.

---

## GraphQL helper: fetch IDs

All mutations require node IDs that cannot be hardcoded. Always fetch them fresh at
the start of any operation. Store them in variables for reuse within the session.

### Fetch project ID and Status field option IDs

```graphql
query GetProject($org: String!, $repo: String!) {
  organization(login: $org) {
    projectsV2(first: 20) {
      nodes {
        id
        title
        number
        fields(first: 20) {
          nodes {
            ... on ProjectV2SingleSelectField {
              id
              name
              options {
                id
                name
              }
            }
          }
        }
      }
    }
  }
}
```

Execute via:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  https://api.github.com/graphql \
  -d '{"query": "..."}'
```

From the response:
- Find the project whose `title` matches the repository name.
- Extract `project_id` (the `id` field).
- Extract `status_field_id` (the `id` of the field named `Status`).
- Extract option IDs for each column name. Store as:

```
STATUS_OPTIONS = {
  "Backlog":     "<option_id>",
  "Todo":        "<option_id>",
  "In Progress": "<option_id>",
  "In Review":   "<option_id>",
  "Done":        "<option_id>"
}
```

### Fetch issue node ID

```bash
curl -s \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/QuantumSensing/{repo}/issues/{issue_number}"
```

Extract `node_id` from the response.

---

## Workflow A — Session start (task breakdown)

Run this at the beginning of every coding session.

### Step 1 — Fetch current board state

Query all open issues and their current Status:

```graphql
query GetProjectItems($projectId: ID!) {
  node(id: $projectId) {
    ... on ProjectV2 {
      items(first: 50) {
        nodes {
          id
          content {
            ... on Issue {
              number
              title
              state
              labels(first: 5) {
                nodes { name }
              }
            }
          }
          fieldValues(first: 10) {
            nodes {
              ... on ProjectV2ItemFieldSingleSelectValue {
                name
                field { ... on ProjectV2SingleSelectField { name } }
              }
            }
          }
        }
      }
    }
  }
}
```

Display a summary to the human:

```
Current board state for {repo}:

Todo:        #{n} Title, #{n} Title
In Progress: #{n} Title
In Review:   #{n} Title
Backlog:     #{n} Title, #{n} Title
Done:        (not shown — archived)
```

### Step 2 — Propose new tasks

Based on the session goal (provided by the human or inferred from context), propose
a set of new issues. Present them for approval before creating anything:

```
Proposed tasks for this session:

1. [Coding] Short imperative title — one-sentence description
2. [Research] Short imperative title — one-sentence description
3. [Coding] Short imperative title — one-sentence description

Shall I create these as issues and add them to the board?
```

Do not create any issues until the human confirms.

### Step 3 — Create issues

For each approved task, create a GitHub Issue via REST:

```bash
curl -s -X POST \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/QuantumSensing/{repo}/issues" \
  -d '{
    "title": "<imperative title>",
    "body": "<issue body — see template below>",
    "labels": ["<label>"]
  }'
```

Capture the `number` and `node_id` of each created issue.

#### Issue body template

Ensure all maths is in LaTeX formatting for Markdown.

```markdown
## Why
<!-- What problem does this task solve? -->

## What
<!-- Concrete definition of done. What will exist or work when this is complete? -->

## Acceptance criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Notes
<!-- Physics/numerical considerations, relevant references, constraints. Omit if none. -->
```

Label must be exactly one of: `Writing`, `Coding`, `Experiments`, `Research`, `Won't Fix`.
Select the single most appropriate label per issue.

### Step 4 — Add issues to project and set status

For each new issue, add it to the project:

```graphql
mutation AddItem($projectId: ID!, $contentId: ID!) {
  addProjectV2ItemById(input: {
    projectId: $projectId
    contentId: $contentId
  }) {
    item { id }
  }
}
```

Capture the returned item `id` (this is the project item ID, distinct from the issue node ID).

Then set Status to `Todo`:

```graphql
mutation SetStatus($projectId: ID!, $itemId: ID!, $fieldId: ID!, $optionId: String!) {
  updateProjectV2ItemFieldValue(input: {
    projectId: $projectId
    itemId: $itemId
    fieldId: $fieldId
    value: { singleSelectOptionId: $optionId }
  }) {
    projectV2Item { id }
  }
}
```

Use `STATUS_OPTIONS["Todo"]` as `$optionId`.

### Step 5 — Confirm and begin

Once all issues are created and on the board:

```
Board updated. {n} tasks added to Todo.

Starting with: #{number} — {title}

Let's go.
```

---

## Workflow B — Moving issues during a session

Move an issue to a new column by updating its Status field. Always fetch the project
item ID for the issue before mutating (it may not be cached).

### Fetch project item ID for an issue

```graphql
query GetItemId($projectId: ID!, $issueNumber: Int!) {
  node(id: $projectId) {
    ... on ProjectV2 {
      items(first: 50) {
        nodes {
          id
          content {
            ... on Issue { number }
          }
        }
      }
    }
  }
}
```

Filter by `number` to find the item ID.

### Move mutation

```graphql
mutation MoveItem($projectId: ID!, $itemId: ID!, $fieldId: ID!, $optionId: String!) {
  updateProjectV2ItemFieldValue(input: {
    projectId: $projectId
    itemId: $itemId
    fieldId: $fieldId
    value: { singleSelectOptionId: $optionId }
  }) {
    projectV2Item { id }
  }
}
```

#### Trigger points (automatic — do not ask the human)

| Event | Move to |
|---|---|
| Claude begins work on an issue | `In Progress` |
| PR opened for this issue | `In Review` |
| Human confirms PR merged | `Done` |
| Issue is blocked or deferred | `Backlog` |

#### Linking PR to issue

When opening a PR, include `Closes #{issue_number}` in the PR body. GitHub will
auto-close the issue on merge. Include this in the PR body generated by the
`code-review` skill:

```markdown
Closes #{issue_number}
```

Add this line immediately after the `## What` section of the PR body.

Linking issues to PRs and branches should be done automatically.

---

## Workflow C — Session close

At the end of a session, present a summary:

```
Session summary for {repo}:

Completed:   #{n} Title, #{n} Title
In Review:   #{n} Title (awaiting your merge)
In Progress: #{n} Title (carried over)
Remaining:   #{n} Title, #{n} Title

Review open PRs at: https://github.com/QuantumSensing/{repo}/pulls
Project board:      https://github.com/orgs/QuantumSensing/projects/{project_number}
```

Do not move any issues to Done at session close unless the human has confirmed a
merge. The `code-review` skill handles the Done transition.

---

## Integration with `code-review` skill

When the `code-review` skill opens a PR:
1. This skill moves the linked issue from `In Progress` → `In Review`.
2. The PR body includes `Closes #{issue_number}`.
3. When the human confirms merge, this skill moves the issue to `Done`.

The `code-review` skill should call into this skill's move logic at two points:
- After `curl` to open PR → move to `In Review`
- After human confirms merge → move to `Done`

---

## Project initialisation (called from `research-project-init`)

When a new repository is created, this skill must:

### 1 — Create labels

Labels are defined at the `QuantumSensing` organisation level and propagate
automatically to new repositories on creation. Do not create them manually per repo.

The canonical label set and colours are:

| Label | Hex |
|---|---|
| `Coding` | `#8338ec` |
| `Experiments` | `#fb5607` |
| `Research` | `#ff006e` |
| `Writing` | `#006cf4` |
| `Won't Fix` | `#000000` |

If labels are missing on a repository (e.g. a fork, or a repo created before the
org-level defaults were set), recreate them with the correct colours:

```bash
# Remove GitHub defaults first
for label in "bug" "documentation" "duplicate" "enhancement" \
             "good first issue" "help wanted" "invalid" \
             "question" "wontfix"; do
  curl -s -X DELETE \
    -H "Authorization: Bearer $GITHUB_TOKEN" \
    -H "Accept: application/vnd.github+json" \
    "https://api.github.com/repos/QuantumSensing/{repo}/labels/$(python3 -c "import urllib.parse; print(urllib.parse.quote('$label'))")"
done

# Create canonical labels
declare -A COLOURS=(
  ["Coding"]="8338ec"
  ["Experiments"]="fb5607"
  ["Research"]="ff006e"
  ["Writing"]="006cf4"
  ["Won't Fix"]="000000"
)

for label in "Coding" "Experiments" "Research" "Writing" "Won't Fix"; do
  curl -s -X POST \
    -H "Authorization: Bearer $GITHUB_TOKEN" \
    -H "Accept: application/vnd.github+json" \
    "https://api.github.com/repos/QuantumSensing/{repo}/labels" \
    -d "{\"name\": \"$label\", \"color\": \"${COLOURS[$label]}\"}"
done
```

### 2 — Create the GitHub Project

```graphql
mutation CreateProject($ownerId: ID!, $title: String!) {
  createProjectV2(input: {
    ownerId: $ownerId
    title: $title
  }) {
    projectV2 {
      id
      number
    }
  }
}
```

Use the organisation node ID as `$ownerId`. Fetch it with:

```graphql
query GetOrgId {
  organization(login: "QuantumSensing") {
    id
  }
}
```

Set `$title` to the repository name.

### 3 — Configure Status field columns

The project is created with a default Status field. Fetch its ID, then the column
options are set via the GitHub UI or GraphQL. As of 2025, creating new single-select
options programmatically requires:

```graphql
mutation UpdateStatusField($projectId: ID!, $fieldId: ID!) {
  updateProjectV2Field(input: {
    projectId: $projectId
    fieldId: $fieldId
    singleSelectOptions: [
      { name: "Backlog",     color: GRAY,   description: "" }
      { name: "Todo",        color: BLUE,   description: "" }
      { name: "In Progress", color: YELLOW, description: "" }
      { name: "In Review",   color: ORANGE, description: "" }
      { name: "Done",        color: GREEN,  description: "" }
    ]
  }) {
    projectV2Field { id }
  }
}
```

If this mutation is unavailable (API versions vary), instruct the human to set the
five columns manually in the GitHub UI before the first session, then confirm.

### 4 — Link repository to project

```graphql
mutation LinkRepo($projectId: ID!, $repoId: ID!) {
  linkProjectV2ToRepository(input: {
    projectId: $projectId
    repositoryId: $repoId
  }) {
    repository { name }
  }
}
```

Fetch the repository node ID with:

```bash
curl -s \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/QuantumSensing/{repo}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['node_id'])"
```

### 5 — Confirm

```
Project created: https://github.com/orgs/QuantumSensing/projects/{project_number}

Board configured with columns: Backlog → Todo → In Progress → In Review → Done
Labels created: Writing, Coding, Experiments, Research, Won't Fix

If the Status columns need to be set manually, do so now, then confirm.
```

---

## Common failure modes

| Symptom | Action |
|---|---|
| 401 on project mutation | Token lacks `project` scope; prompt human to regenerate |
| `singleSelectOptionId` not found | Re-fetch STATUS_OPTIONS — IDs may have changed |
| Project not found for repo | Check project title matches repo name exactly |
| `updateProjectV2Field` unavailable | Instruct human to set columns in UI manually |
| Issue not appearing on board | Confirm `addProjectV2ItemById` succeeded; check item ID |
| Labels already exist on create | DELETE existing before POST; or use PATCH to update |
