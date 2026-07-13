---
name: task-list
description: Use whenever the user wants a quick read of their current task list without opening the dashboard. Triggers on phrases like "what's on my list", "show open tasks", "tasks for Helen", "what do I owe Gav", "tasks for this project", "open follow-ups", "what's in progress", "what tasks tagged X", "who am I waiting on", "what am I waiting on and for how long", "who's waiting on me", "what am I blocking", "what's outstanding from Ed", "/task-dashboard:task-list", or "list tasks". Reads frontmatter from the task-data tasks directory, applies any filter mentioned (person, project, state, type, tag, waiting-on, blocking), and prints a compact table, or a dedicated waiting-on/blocking view for "who am I waiting on" and "who is waiting on me" style questions. Read-only; never writes.
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
- **Waiting-on:** "who am I waiting on", "what's outstanding from Ed" → filter to tasks with a non-empty `waiting-on:` (optionally narrowed to a specific value, case-insensitive substring match). Triggers the dedicated waiting-on render (step 5).
- **Blocking:** "who is waiting on me", "what am I blocking", "what does Jason need from me" → filter to tasks with a non-empty `blocking:` (optionally narrowed to a specific value). Triggers the dedicated blocking render (step 5).

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
- `waiting-on`, `waiting-since` (may be absent; `waiting-since` is only meaningful alongside `waiting-on`)
- `blocking`, `blocking-since` (may be absent; `blocking-since` is only meaningful alongside `blocking`)

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

### 5a. Waiting-on / blocking views

Two dedicated renders, used instead of the default table when the question is specifically about waiting-on or blocking (step 1).

**"Who am I waiting on"** — filter to tasks with a non-empty `waiting-on`. Sort by `waiting-since` ascending (oldest first — the longest-outstanding wait is the most overdue). Age is computed from `waiting-since` to today, in whole days — `waiting-since` is the reason this is honest: `last-updated` moves every time a note is appended, so it is never used for age here. A task with `waiting-on` set but no `waiting-since` (a hand-edited file, or data from before this field existed) shows age as `?` and sorts last. Example:

```
Waiting on (3)

T12  47 days  Ed                            Chase Ed about the Q2 numbers
T34  12 days  architecture forum sign-off   Finish the ADR
T9    3 days  Gav                           Confirm the schema decision
```

**"Who is waiting on me"** — filter to tasks with a non-empty `blocking`. Sort by `blocking-since` ascending (oldest first — the longest-outstanding ask is the most overdue), exactly mirroring the waiting-on view above. Age is computed from `blocking-since` to today, in whole days, for the same reason: `last-updated` moves on every note append, so it is never used for age here. A task with `blocking` set but no `blocking-since` (a hand-edited file, or data from before this field existed) shows age as `?` and sorts last. "Jason has been waiting on you for 20 days" is the sentence that changes behaviour; "Jason is waiting on you" is a fact you skim past — the age column is the point. Example:

```
Blocking (3)

T15  20 days  Jason  Finish the migration script
T22   6 days  Karl   Send over the design doc
T31      ?    Karl   Confirm the API contract
```

Both views respect the ~25-row cap and the same "… and N more" overflow line as the default render.

### 6. State the filter

At the very top, one short line, so the user can sanity-check what was filtered. E.g.:

> Filtering: person=Helen, state=open|in-progress, type=any, project=any

If no filter was applied, just say `All active tasks.`

## Boundaries

- **Never read `<task-data>/tasks/archive/`** under any phrasing. Archived means closed; if the user asks "what have I done lately" suggest the archive directory or the dashboard's archive view, then stop.
- **Never modify files.** This skill is read-only. If the user chains "list my tasks and mark T19 done", do the list and hand off to `task-dashboard:task-complete` for the mark — don't bundle.
- **Never log full task bodies.** Title and metadata only. The body is on disk; the user reads there.
- **Never paginate beyond the first 25.** If the result is enormous, the filter was too loose or the dashboard is the right tool.
