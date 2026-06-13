---
name: task-update
description: Use whenever the user wants to change the state, type, or optional fields of an existing task, or append a one-line context note to it, by ID. Triggers on phrases like "update T12", "T19 is in progress", "T7 is blocked on …", "add a note to T15", "T22 — Ben pushed back", "T8 — moving forward", "T19 is actually an issue", "reclassify T7 as an action", "promote T9 to an action", "park T12", "T12 on hold", "T12 branch is foo", "set the branch on T15 to bar", "T9 — assign to Helen", "T7 — add tag infra", "/task-dashboard:task-update T<n>", or any sentence that names a `T<digits>` id and either announces progress, a block, a parking, a reclassification, a branch assignment, a person change, a tag change, or a new fact. Edits the frontmatter `state`, `type`, `branch`, `person`, and/or `tags`, and/or appends a single body line. Does NOT mark tasks done — that's `task-dashboard:task-complete`. Does NOT create new tasks — that's `task-dashboard:task-create`.
---

# Task Update

In-session edit of a known task. Four things this skill does, and only these:

1. **Change `state`** in frontmatter — `open`, `in-progress`, or `on-hold` (parking a started task somewhere kept warm but out of the Active view). (For `done` or `dropped`, hand off to `task-dashboard:task-complete`, which also runs the archive sweep.)
2. **Change `type`** in frontmatter — `action`, `idea`, or `issue`, and nothing outside that enum. The common case is promoting an `idea` to an `action` when the user decides to act on it; reframing to `issue` is equally valid.
3. **Set or update optional frontmatter fields** — `branch`, `person`, and `tags`. Set them when first provided; update them when a new value is given; clear them by setting to `null` or `[]` (for tags) when the user explicitly removes one. These fields may not exist in older task files and should be added when first set.
4. **Append a single short line** to the body, capturing new context — a date, a name, a blocker, a small decision.

Any one of these, or a combination. If a request would do more than this (rewriting the title, restructuring the body, moving the file), say so and offer to open the file in the editor instead.

## Path resolution

All paths below refer to the **task-data repository**. Resolve its location in this order:

1. The `TASK_DATA_DIR` environment variable, if set.
2. Otherwise `~/projects/task-data` on macOS/Linux, or `%USERPROFILE%\projects\task-data` on Windows 11.

In all commands below, substitute the resolved path for `<task-data>`.

## Workflow

### 1. Find the file by ID

Active tasks live in `<task-data>/tasks/`; archived ones in `<task-data>/tasks/archive/`. Look at active first; archived files are by definition closed and shouldn't be reopened here.

```bash
grep -l "^id: T<n>$" <task-data>/tasks/*.md 2>/dev/null
```

If zero matches in active, check archive and report it: "T<n> is in the archive — reopen with `task-dashboard:task-update` after restoring, or leave it." If more than one match in active (shouldn't happen), use the most recently modified and warn.

### 2. Determine the intent

Read the user's sentence carefully:

- **State → in-progress:** "starting T12", "working on T12", "T12's in flight", "kicking off T12".
- **State → on-hold:** "parking T12", "back-burnering T12", "T12 on hold", "put T12 on the back burner". On-hold is the resting place for started-but-parked work: it stays out of the Active view but is never archived. Parking language targets `on-hold`, never `open`.
- **State → open** (a genuine reopen): "reopen T12", "T12 is active again", "back to open on T12" — only when the user wants it back in the Active list, not merely parked.
- **Type change:** "T9 is actually an action now", "promote T9 to an action", "reclassify T7 as an issue", "T12 turned out to be a bug, not an idea". The frequent case is `idea → action` when the user commits to acting on a parked idea, often alongside `state → in-progress` in the same breath.
- **Branch set/change:** "T12 branch is t12-my-feature", "set branch on T15 to releases/foo", "T9 — branch is users/rc/thing". Extract the branch name verbatim.
- **Person set/change:** "T9 — assign to Helen", "T12 is now on Ben", "T7 — person is Karl". Use the first name as given.
- **Tags set/change:** "T7 — tag it infra", "T12 add tags auth, api", "clear tags on T9".
- **Append context only, no other change:** anything that adds a fact without announcing progress, a reclassification, or a field assignment. "T12 — Helen confirmed she'll come back Friday", "T19 — Ben's going to push back on the schema", "T7 needs the ADR signed off first".

If the intent is mixed (any combination of field changes + new context), do all of it in one pass.

### 3. Edit frontmatter (if any field changed)

Use the Edit tool. Apply all of the following that are relevant:

- **`state`:** replace `state: <current>` with `state: <new>`, where `<new>` is one of `open`, `in-progress`, or `on-hold` (`done`/`dropped` hand off to `task-dashboard:task-complete`).
- **`type`:** replace `type: <current>` with `type: <new>`, where `<new>` is exactly one of `action`, `idea`, or `issue` — reject anything outside that enum and stop rather than write a junk value.
- **`branch`:** set or replace the `branch:` line with the new value. If the field doesn't exist, insert it after `last-updated`. To clear, set `branch: null`.
- **`person`:** set or replace the `person:` line. If the field doesn't exist, insert it after `last-updated` (or after `branch` if present). To clear, remove the line entirely.
- **`tags`:** set or replace the `tags:` line, e.g. `tags: [infra, auth]`. If the field doesn't exist, insert it after `last-updated`. To clear, set `tags: []`.

Always update `last-updated` to today's date in YYYY-MM-DD format. If `last-updated` is not yet in the frontmatter, add it immediately after `created`. Never touch `id`, `title`, `created`, or `project` — those are immutable.

### 4. Append context line (if there's new info)

One line, plain prose, ≤ 140 chars. Lead with today's date in long British format, e.g.:

```
11 May 2026 — Helen confirmed she'll come back by Friday on the Q2 numbers.
```

Append it to the end of the body, after a single blank line if the body doesn't already end in one. Don't add a heading like `## Updates`. Don't reformat existing content. The dashboard renders the body as-is; tiny appended lines build up naturally over time.

For a genuine blocker or warning that deserves visual emphasis, the one fact may instead be appended as a `> [!warning]` callout (two physical lines: `> [!warning]` then `> <date> — <text>`), which the dashboard renders as a tinted block. Use sparingly — one callout per genuinely important inflection, never for routine notes.

Whether or not a context line is appended, always update `last-updated` in the frontmatter to today's date.

### 5. Report

```
✏️  T12 → type: action · state: in-progress     <task-data>/tasks/t12-foo.md
     + "11 May 2026 — Helen confirmed she'll come back by Friday."
```

Include only the parts that changed on the `→` line: `type:` and/or `state:`, joined by ` · ` when both changed.
If no frontmatter changed (line appended only): omit the `→` line.
If only a line was appended (no frontmatter change): omit the `→` line; if no line was appended, omit the `+ "…"` line.

## Examples

**Example 1 — state only**

User: *"T19 — starting on it now"*

- Find T19, set `state: in-progress`. No body line.

**Example 2 — context only**

User: *"T22 — Ben pushed back on the schema; want to keep the date string instead of converting to ISO"*

- Find T22, no state change. Append a body line summarising the push-back: "11 May 2026 — Ben prefers keeping the date string; doesn't want ISO conversion."

**Example 3 — both**

User: *"T7 in flight — got Charlotte's sign-off, just need to write the diff"*

- Find T7, set `state: in-progress`. Append: "11 May 2026 — Charlotte signed off; remaining work is the diff."

**Example 4 — type promotion (idea → action), with state**

User: *"right, let's actually do T9 — it's an action now, starting it"*

- Find T9, set `type: action` and `state: in-progress`. Append: "11 May 2026 — Promoted from idea to action; starting implementation." Report line: `T9 → type: action · state: in-progress`.

**Example 5 — type only**

User: *"T15 isn't an idea, it's a bug"*

- Find T15, set `type: issue`. No state change. Append a one-line note only if the user added a fact; a bare reclassification needs no body line. Report line: `T15 → type: issue`.

**Example 6 — parking to on-hold**

User: *"park T12 for now — waiting on the vendor to confirm the format"*

- Find T12, set `state: on-hold` (parking language targets on-hold, never `open`). Append: "9 June 2026 — Parked pending vendor confirmation of the format." Report line: `T12 → state: on-hold`.

**Example 7 — branch set**

User: *"T19 — branch is t19-auth-refactor"*

- Find T19, set (or add) `branch: t19-auth-refactor`. No state/type change, no body line needed unless the user added a fact. Report line: `T19 → branch: t19-auth-refactor`.

**Example 8 — person change + context**

User: *"T7 — moving to Karl, he's picking this up now"*

- Find T7, set `person: Karl`. Append: "11 June 2026 — Karl taking this over." Report line: `T7 → person: Karl`.

## Boundaries

- **Never edit `id`, `title`, `created`, or `project`.** Those changes warrant opening the file in the editor; surface that and stop.
- **Mutable frontmatter fields this skill may edit:** `state`, `type`, `branch`, `person`, `tags`, and `last-updated`. Nothing else.
- **A `type` change must land inside the `action | idea | issue` enum.** Anything else is rejected; stop and report rather than write a junk value.
- **A `state` change must land inside the `open | in-progress | on-hold` enum.** `done`/`dropped` hand off to `task-dashboard:task-complete`.
- **Always refresh `last-updated` to today's date** on every invocation, even if only a body line was appended.
- **Never edit a file in `<task-data>/tasks/archive/`.** Archived means closed.
- **Don't add more than one body line per invocation.** If the user volunteers multiple updates in one breath, condense to one line, or ask which line they want. The risk of long appended monologues is that the file body bloats and the dashboard preview becomes useless.
- **Don't fall through to `task-dashboard:task-create` if the ID isn't found.** Stop and report. A missing ID is more likely a typo than a missing task.
