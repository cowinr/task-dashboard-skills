---
name: task-list
description: Use whenever the user wants a quick read of their current task list without opening the dashboard. Triggers on phrases like "what's on my list", "show open tasks", "tasks for Helen", "what do I owe Gav", "tasks for this project", "open follow-ups", "what's in progress", "what tasks tagged X", "/task-dashboard:task-list", or "list tasks". Reads frontmatter from the task-data tasks directory, applies any filter mentioned (person, project, state, type, tag), and prints a compact table. Read-only; never writes.
---

# Task List

Read-only summary of active tasks, filtered to what the user asked for. The dashboard at `http://127.0.0.1:4731` is the rich view; this skill is the CLI quick read when they don't want to switch context.

Only reads from `<task-data>/tasks/*.md`. Never reads `<task-data>/tasks/archive/`. Active means active.

## Path resolution

All paths below refer to the **task-data repository**. Resolve its location in this order:

1. The `TASK_DATA_DIR` environment variable, if set.
2. Otherwise `~/projects/task-data` on macOS/Linux, or `%USERPROFILE%\projects\task-data` on Windows 11.

In all commands below, substitute the resolved path for `<task-data>`.

## Workflow

### 1. Parse the filter from the user's sentence

Pick out any of:

- **Person:** "tasks for Helen", "what do I owe Gav", "anything outstanding with Ben" → filter by frontmatter `person:` (case-insensitive match on the bare first name).
- **Project:** "for this project" / "tasks here" → use the encoded cwd (see path encoding in `task-dashboard:task-create`). "Tasks for claude-dashboard" → look for `project:` ending in `-claude-dashboard`.
- **State:** "open tasks" (state: open), "in progress" (state: in-progress), or "any state" if the user didn't constrain. Default: `open` and `in-progress` (i.e. anything not done/dropped).
- **Type:** "actions" (type: action), "ideas" (type: idea), or both. Default: both.
- **Tag:** "tagged X", "for the task-dashboard work" → match against `tags: [...]`.

If the user's sentence is ambiguous, pick the most likely interpretation and state the filter at the top of the output. Don't ping-pong with clarifying questions for a read-only command.

### 2. Read frontmatter from every active task

For each `.md` file in `<task-data>/tasks/` (NOT recursing into `archive/`), pull these fields:

- `id`
- `title` (strip surrounding quotes if present)
- `type`
- `state`
- `project`
- `person` (may be absent)
- `tags` (may be absent)

Skip files that fail to parse — log a one-line warning under the table.

### 3. Apply filters

Cumulative. Person filter AND state filter AND tag filter, etc. Empty filter on a dimension means "no constraint on that dimension".

### 4. Sort

By default: state ascending (`in-progress` first, then `open`), then ID ascending. If the user asked for a person, sort by ID ascending within that person only — no point grouping by state.

### 5. Render

Compact, monospace-friendly. Don't use a markdown table for small results; use aligned plain text. Example output for "tasks for Helen":

```
Tasks for Helen (2 open, 1 in progress)

T19  in-progress  action  Chase Helen about Q2 numbers
T34  open         idea    Helen-suggested dashboard tweak
T42  open         action  Helen — schedule fortnightly catch-up
```

For an unfiltered "what's open":

```
12 open / 3 in progress

T7   in-progress  action  Draft ADR for new auth middleware
T12  in-progress  action  Sweep todo from Gav's review
T19  in-progress  action  Chase Helen about Q2 numbers
T22  open         action  Ben — confirm schema decision
T34  open         idea    Auto-archive tasks > 30 days old
…
```

Cap output at ~25 rows. If more match the filter, print the first 25 and add `… and N more — open the dashboard for the full list.` at the bottom.

### 6. State the filter

At the very top, one short line, so the user can sanity-check what was filtered. E.g.:

> Filtering: person=Helen, state=open|in-progress, type=any, project=any

If no filter was applied, just say `All active tasks.`

## Boundaries

- **Never read `<task-data>/tasks/archive/`** under any phrasing. Archived means closed; if the user asks "what have I done lately" suggest the archive directory or the dashboard's archive view, then stop.
- **Never modify files.** This skill is read-only. If the user chains "list my tasks and mark T19 done", do the list and hand off to `task-dashboard:task-complete` for the mark — don't bundle.
- **Never log full task bodies.** Title and metadata only. The body is on disk; the user reads there.
- **Never paginate beyond the first 25.** If the result is enormous, the filter was too loose or the dashboard is the right tool.
