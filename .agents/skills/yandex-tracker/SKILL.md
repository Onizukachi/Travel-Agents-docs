---
name: yandex-tracker
description: Work with Yandex Tracker tasks through the REST API. Use when Codex needs to read, create, or update Tracker issues, inspect issue title/description/properties, search issues, list or summarize current tasks on a board, inspect backlog-like issue sets, analyze statuses, story points, assignees, sprints, board columns, add issue comments, list possible status transitions, move issues through statuses, or analyze LevelTravel Scrum/EpicFlow project coverage, Scrum/EpicFlow alignment, and team activity/productivity by sprint, team, project, and assignee.
---

# Yandex Tracker

## Quick Start

Use the bundled `tracker.py` helper from this repository for API calls instead of hand-writing `curl`.
Resolve it through the current git checkout so the skill works from any directory inside the project:

```bash
TRACKER_HELPER="$(git rev-parse --show-toplevel)/.agents/skills/yandex-tracker/scripts/tracker.py"
```

Configure credentials once:

```bash
python3 "$TRACKER_HELPER" setup \
  --token '<oauth-or-iam-token>' \
  --org-id '<organization-id>' \
  --org-header X-Org-ID
```

The script stores credentials in `~/.yandex-tracker.env` with mode `0600`.
Use this env-file format:

```dotenv
YANDEX_TRACKER_ORG_ID=<organization-id>
YANDEX_TRACKER_TOKEN=<oauth-or-iam-token>
YANDEX_TRACKER_AUTH_TYPE=OAuth
YANDEX_TRACKER_ORG_HEADER=X-Org-ID
YANDEX_TRACKER_API_BASE=https://api.tracker.yandex.net/v3
YANDEX_TRACKER_DEFAULT_USER_NAME=<default-person-name>
YANDEX_TRACKER_DEFAULT_USER_EMAIL=<default-person-email>
YANDEX_TRACKER_DEFAULT_BOARDS=2:Scrum,214:EpicFlow
```

Use `--auth-type Bearer --org-header X-Cloud-Org-ID` for IAM tokens in Yandex Cloud organizations.
When the user asks for "my tasks", "current tasks", "the board", or analysis without naming a person or board, use `YANDEX_TRACKER_DEFAULT_USER_*` and `YANDEX_TRACKER_DEFAULT_BOARDS`.

Check non-secret effective config:

```bash
python3 "$TRACKER_HELPER" config
```

Read `references/api.md` when the task needs exact endpoint behavior, pagination, board grouping, or Tracker query details.
Read `references/leveltravel.md` for LevelTravel board IDs, queues, team fields, and default analysis commands.
Read `references/epicflow-manifest.md` when analyzing whether Scrum tasks, development projects, and EpicFlow issues follow the initiative workflow.

## Workflows

### Read an Issue

1. Run:

```bash
python3 "$TRACKER_HELPER" issue QUEUE-123
```

2. Summarize key, summary, description, status, assignee, queue, priority, type, sprint, story points, and updated time.
3. If the user asks for raw properties, rerun with `--raw`.

### Board Status

1. If the user does not name a board, get default board summaries:

```bash
python3 "$TRACKER_HELPER" default-board-status --limit 200
```

2. If the user names a specific board, get that board status summary:

```bash
python3 "$TRACKER_HELPER" board-status 123 --limit 200
```

3. Report board name, columns/statuses, task counts, and representative task keys.
4. If the board has a broad filter and the result is truncated by `--limit`, say so and narrow with `--query` or `--filter-json`.

### Backlog Status

Yandex Tracker backlogs are board-specific product areas rather than a single universal API resource. Prefer one of:

- If the user gives a board ID, run `board-status` and inspect board filter, columns, sprint fields, and task statuses.
- If the user gives no board ID, run `default-board-status` and inspect configured boards.
- If the user gives a queue or saved filter criteria, use issue search:

```bash
python3 "$TRACKER_HELPER" search \
  --filter-json '{"queue":"QUEUE","status":"open"}' \
  --order +updated \
  --fields key,summary,status,assignee,updatedAt
```

- If the organization has a local "backlog" field or saved query, ask for that field/query name only if it cannot be inferred from board/filter data.

### LevelTravel Scrum/EpicFlow Coverage

Use this workflow for questions like "which Scrum sprint tasks have epics", "what tasks need EpicFlow", "which projects violate the EpicFlow manifest", or "cluster tasks without projects".

1. Check the current Scrum sprint:

```bash
python3 "$TRACKER_HELPER" current-sprint
```

2. Analyze all current sprint tasks:

```bash
python3 "$TRACKER_HELPER" epicflow-coverage --sprint current
```

3. For one team, pass the Tracker team value:

```bash
python3 "$TRACKER_HELPER" epicflow-coverage --sprint current --team WebFront
```

4. Use `--details` when the user needs task/project lists rather than counts:

```bash
python3 "$TRACKER_HELPER" epicflow-coverage --sprint current --team WebBack --details
```

5. For tasks without development projects, cluster likely project candidates:

```bash
python3 "$TRACKER_HELPER" no-project-clusters --sprint current
```

6. To compare current Scrum sprint work against active EpicFlow board projects:

```bash
python3 "$TRACKER_HELPER" scrum-epicflow-alignment --sprint current
```

Use `--details` when the user asks for the actual projects/tasks behind the counts.

Interpret results using `references/epicflow-manifest.md`: task without project means "assign/create a project first"; project without EpicFlow is a candidate for one EpicFlow issue; project with multiple EpicFlow is cleanup.

### LevelTravel Activity And Productivity

Use this workflow when the user asks what teams did recently, asks for summaries by project or person, or asks about productivity.

1. Summarize LT tasks updated in a date window:

```bash
python3 "$TRACKER_HELPER" activity-summary --since 2026-06-08 --until 2026-06-22
```

2. For exact productivity, include changelog transitions into done statuses:

```bash
python3 "$TRACKER_HELPER" activity-summary --since 2026-06-08 --until 2026-06-22 --with-changelog
```

Interpret `updatedTasks` as activity breadth, `doneTransitions.doneTasks` as throughput, and `activeTasks` as current WIP. Story points are counted only where the field is filled.

### Create an Issue

Only create tasks when the user explicitly asks to create a Tracker task and provides enough required fields: queue and summary. If the queue is missing and no safe default is implied by the prompt, ask for it.

Basic task:

```bash
python3 "$TRACKER_HELPER" create \
  --queue LT \
  --summary 'Краткое название задачи' \
  --description-file /path/to/description.md \
  --markup-type md
```

Common optional fields:

```bash
python3 "$TRACKER_HELPER" create \
  --queue LT \
  --summary 'Краткое название задачи' \
  --type task \
  --priority normal \
  --assignee userlogin \
  --parent LT-123 \
  --tags tag1,tag2 \
  --unique external-id-123
```

For local queue fields or fields not exposed as first-class flags, pass a JSON object:

```bash
python3 "$TRACKER_HELPER" create \
  --queue LT \
  --summary 'Краткое название задачи' \
  --field-json '{"storyPoints": 3, "components": ["dynamic"]}'
```

After creation, report the created issue key and a concise field summary. Use `--raw` if the user needs the full API response.

### Update an Issue

Only update tasks when the user explicitly asks to change a specific issue key. Read the issue first if the requested change depends on current values or could overwrite existing text:

```bash
python3 "$TRACKER_HELPER" issue LT-123 --raw
```

Patch common fields:

```bash
python3 "$TRACKER_HELPER" update LT-123 \
  --summary 'Новое название' \
  --description-file /path/to/description.md \
  --markup-type md \
  --assignee userlogin
```

Patch additive/removal fields:

```bash
python3 "$TRACKER_HELPER" update LT-123 \
  --add-tags tag1,tag2 \
  --remove-tags obsolete \
  --add-followers userlogin
```

Use `--field-json` for queue-specific fields and patch operators:

```bash
python3 "$TRACKER_HELPER" update LT-123 \
  --field-json '{"storyPoints": 5, "followers": {"remove": ["old-login"]}}'
```

Do not update status with `update` or `--field-json`; use `transitions`, `move`, or `transition`.

### Add a Comment

Only add comments when the user explicitly asks for it and gives the target issue and comment text.

```bash
python3 "$TRACKER_HELPER" comment LT-123 \
  --text 'Комментарий'
```

For multiline comments:

```bash
python3 "$TRACKER_HELPER" comment LT-123 \
  --text-file /path/to/comment.txt \
  --markup-type md
```

### Move an Issue Through Statuses

1. Inspect available transitions when the target transition is not obvious:

```bash
python3 "$TRACKER_HELPER" transitions LT-123
```

2. Move by exact transition id/display or target status key/display:

```bash
python3 "$TRACKER_HELPER" move LT-123 'Review' \
  --comment 'Moved to review'
```

3. If Tracker requires transition fields, pass them as a JSON object:

```bash
python3 "$TRACKER_HELPER" transition LT-123 review \
  --field-json '{"resolution":"fixed"}' \
  --comment 'Done'
```

The `move` command refuses to execute if the target does not match exactly one available transition.

## Safety Rules

- Never print or paste stored tokens.
- Use write commands only for explicit user requests. Creation requires a queue and summary; updates/comments/transitions require an exact issue key.
- Read the current issue before updating when the user requests partial edits to description-like fields, asks to preserve existing content, or references current values.
- For create/update queue-specific fields, prefer `--field-json` with a small JSON object; do not invent local field keys.
- Before moving a task, list transitions first unless the user gave a precise status/transition and the command can resolve it unambiguously.
- For required transition fields, use `--field-json` with a JSON object; do not guess business-specific values.
- Status cannot be changed through `update`; use status transitions only.
- Use the same permissions model as the Tracker UI; `403` means the current token user lacks rights or required API scope.
- For IAM tokens, expect expiry and `401`; ask the user to refresh the token rather than debugging unrelated code.
