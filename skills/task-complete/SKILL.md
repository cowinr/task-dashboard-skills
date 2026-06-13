---
name: task-complete
description: Use whenever the user wants to close out an existing task — either finished or abandoned — and have it swept into the archive. Triggers on phrases like "T12 is done", "mark T19 done", "finished T15", "T7 — drop it", "drop T22", "close T8", "/task-dashboard:task-complete T<n>", "kill T<n>", or any sentence naming a `T<digits>` id alongside completion/abandonment language. Sets the frontmatter `state` to `done` (default) or `dropped`, appends a dated resolution note describing what was actually done or what changed, then runs `npm run archive` in the task-dashboard so the file moves to the archive directory.
---

# Task Complete

Final state transition for a task. Two paths:

- **done** — the user finished it. Default unless they say otherwise.
- **dropped** — the user is abandoning it. Triggered by "drop T<n>", "kill T<n>", "scrap T<n>", "no longer doing T<n>", "not bothering with T<n>".

Both states are archive-worthy; the dashboard's `archive` script handles either.

## Path resolution

Two paths to resolve:

**task-data repository** — Resolve in this order:
1. The `TASK_DATA_DIR` environment variable, if set.
2. Otherwise `~/projects/task-data` on macOS/Linux, or `%USERPROFILE%\projects\task-data` on Windows 11.

**task-dashboard repository** — Resolve in this order:
1. The `TASK_DASHBOARD_DIR` environment variable, if set.
2. Otherwise `~/projects/task-dashboard` on macOS/Linux, or `%USERPROFILE%\projects\task-dashboard` on Windows 11.

In all commands below, substitute the resolved paths for `<task-data>` and `<task-dashboard>`.

## Workflow

### 1. Find the file

```bash
grep -l "^id: T<n>$" <task-data>/tasks/*.md 2>/dev/null
```

If zero matches in active, check archive: a `grep -l` in `<task-data>/tasks/archive/` will find it there. If it's already archived, report and stop — don't try to reopen-and-reclose.

### 2. Decide the resolution state

`done` unless the user's wording clearly says abandonment. Examples:

- "T12 is done" → done
- "finished T19" → done
- "T7 — done, finally" → done
- "drop T22", "scrap T22", "not doing T22 any more", "kill T22" → dropped
- "T15 — moot, the underlying issue went away" → dropped (and append a note saying why)

If the wording is genuinely ambiguous, ask once: "Done or dropped?"

### 3. Edit the frontmatter

Set `state: <done|dropped>` and update `last-updated` to today's date in YYYY-MM-DD format. If `last-updated` is not yet in the frontmatter, add it immediately after `created`. This stamps the close-out date before the archive sweep moves the file. Don't touch any other frontmatter field.

### 4. Append a resolution note describing what was done

Before flipping the state, write a note that captures **what was actually done, or what changed** — not a bare "Done in session." The archived task is the only durable record of the work; the note must let a future reader understand the outcome without reconstructing the session.

Lead with today's date in long British format, then a factual summary. Keep it tight — one line when the work genuinely was trivial, two or three short lines (or a brief `- ` list) when there's substance worth recording. Name the concrete artefacts: files or modules touched, the PR or commit, the decision taken, who confirmed it.

- For `done`: describe the outcome and the change. Examples:
  - `11 May 2026 — Done; added retry/backoff to the sync client (src/sync/client.ts), PR #142 merged.`
  - `11 May 2026 — Done; Helen confirmed the Q2 numbers by email, figures filed in the model.`
  - `11 May 2026 — Done; root cause was a stale cache key — fixed in cacheKey() and covered by a regression test.`
- For `dropped`: describe why it's being abandoned and any consequence. Examples:
  - `11 May 2026 — Dropped; superseded by T57, which takes the schema-first approach instead.`
  - `11 May 2026 — Dropped; problem went away after the Node 25 upgrade, no code change needed.`
  - `11 May 2026 — Dropped; no longer relevant after the team reprioritised.`

Source the detail in this order: the user's own wording if they gave any; otherwise the work done in this session (changed files, commands run, decisions made); otherwise the task body and recent context. If genuinely nothing is known beyond "it's done", say so explicitly (`Done; no detail captured.`) rather than inventing specifics — but treat that as the rare exception, not the default.

Append after a single blank line if the body doesn't already end in one. Don't add a heading.

### 5. Run the archive sweep

On macOS/Linux:
```bash
npm --prefix <task-dashboard> run archive
```

On Windows 11 (PowerShell):
```powershell
npm --prefix "<task-dashboard>" run archive
```

If `npm` isn't on PATH, prepend the Node bin directory. On macOS with nvm:
```bash
export PATH="$HOME/.nvm/versions/node/v22.22.0/bin:$PATH" && \
  npm --prefix <task-dashboard> run archive
```

This moves any task whose state is `done` or `dropped` into `<task-data>/tasks/archive/`. Exit cleanly even if the sweep finds nothing else to move; report only what changed for the task just closed.

### 6. Report

```
✅ T12 → done                         archived
     + "11 May 2026 — Resolved; PR #142 merged."
     archive: <task-data>/tasks/archive/t12-foo.md
```

For dropped:

```
🗑  T22 → dropped                     archived
     + "11 May 2026 — Dropped; superseded by T57."
     archive: <task-data>/tasks/archive/t22-bar.md
```

If the archive sweep didn't actually move the file (e.g. the script errored, or the file was already gone), say so plainly and stop — don't pretend success.

## Examples

**Example 1 — done with explicit outcome**

User: *"T12 is done — Helen replied with the Q2 numbers this morning"*

- Set `state: done`, append `11 May 2026 — Done; Helen replied with Q2 numbers.`, run archive sweep.

**Example 2 — done, detail reconstructed from the session**

User: *"finished T19"*

- The user gave no wording, so summarise the work done this session. If T19 was "add alphabetical ordering to the filter dropdowns", append `11 May 2026 — Done; sorted the project/person filter dropdowns alphabetically and fixed the cold-load select sync (src/public/js/panels/tasks.js).` Only fall back to `Done; no detail captured.` if the session genuinely reveals nothing.

**Example 3 — dropped**

User: *"drop T22 — Ben changed his mind and we're keeping the old schema"*

- Set `state: dropped`, append `11 May 2026 — Dropped; Ben opted to keep the old schema.`, run archive sweep.

## Boundaries

- **One task per invocation.** If the user rattles off "T12 and T19 are both done", handle them sequentially as two separate edits and one combined archive sweep.
- **Never close a task without a resolution note that describes what was done or what changed.** The note is the only audit trail once the file is archived; a bare state flip, or a contentless "Done in session.", loses the work. Reconstruct the detail from the session if the user didn't give it.
- **Always update `last-updated` to today's date when closing.** This records the close-out date and survives the archive move.
- **Never modify `id`, `title`, `type`, `created`, `project`, `person`, or `tags`.** Closing a task does not rewrite its history.
- **Never delete files.** Closing means setting state and letting the archive script move the file. The active tree never loses files by hand from this skill.
- **Never run any command other than `npm run archive`** in the task-dashboard. This skill is not a general-purpose dashboard runner.
