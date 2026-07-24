---
name: task-update
description: Use whenever the user wants to change the state, type, or optional fields of an existing task, or append a one-line context note to it, by ID. Triggers on phrases like "update T12", "T19 is in progress", "T7 is blocked on …", "add a note to T15", "T22 — Ben pushed back", "T8 — moving forward", "T19 is actually an issue", "reclassify T7 as an action", "promote T9 to an action", "park T12", "T12 on hold", "T12 branch is foo", "set the branch on T15 to bar", "T9 — assign to Helen", "T7 — add tag infra", "T12 is waiting on Ed", "T9 waiting on architecture forum sign-off", "Ed's unblocked T12", "clear the wait on T7", "T15 is blocking Jason", "clear blocking on T9", "T12 parent is T50", "set the parent on T15 to T9", "T12 is a sub-task of T50", "clear the parent on T7", "T12 is committed for 25 July", "T9 — committed to the board pack, due Friday", "T7 is core", "mark T15 as core", "T12 isn't core any more", "T9 is a focus today", "clear focus on T12", "/task-dashboard:task-update T<n>", or any sentence that names a `T<digits>` id and either announces progress, a block, a parking, a reclassification, a branch assignment, a parent assignment, a person change, a tag change, a waiting-on/blocking change, a committed-date or committed-to change, a core/focus flag change, or a new fact. Edits the frontmatter `state`, `type`, `branch`, `parent`, `person`, `tags`, `waiting-on`, `waiting-since`, `blocking`, `blocking-since`, `committed`, `committed-to`, `core`, and/or `focus`, and/or appends a single body line. Does NOT mark tasks done — that's `task-dashboard:task-complete`. Does NOT create new tasks — that's `task-dashboard:task-create`.
---

# Task Update

In-session edit of a known task. Four things this skill does, and only these:

1. **Change `state`** in frontmatter — `open`, `in-progress`, or `on-hold` (parking a started task somewhere kept warm but out of the Active view). (For `done` or `dropped`, hand off to `task-dashboard:task-complete`, which also runs the archive sweep.)
2. **Change `type`** in frontmatter — `action`, `idea`, or `issue`, and nothing outside that enum. The common case is promoting an `idea` to an `action` when the user decides to act on it; reframing to `issue` is equally valid.
3. **Set or update optional frontmatter fields** — `branch`, `parent`, `person`, `tags`, `waiting-on`, `waiting-since`, `blocking`, `blocking-since`, `committed`, `committed-to`, `core`, and `focus`. Set them when first provided; update them when a new value is given; clear them by setting to `null` or `[]` (for tags) when the user explicitly removes one. These fields may not exist in older task files and should be added when first set. `waiting-on`/`waiting-since` and `blocking`/`blocking-since` are orthogonal to `state` — never infer or change `state` from them, and never derive them from `state`; a task can be `in-progress` and `waiting-on` someone at the same time, and that is the common case, not a contradiction. `waiting-on` is free text naming who **or what** is blocking the task (a person, "architecture forum sign-off", anything) — it is distinct from `person` (a neutral counterparty) and the two are never conflated or auto-populated from each other. `blocking` names who is waiting on the task owner and is set/cleared independently of `waiting-on`. Both pairs share the same dating rule, which mirrors what the dashboard's own API now enforces on every write (see step 3): a field set to a **new or changed** value with no explicit date stamps its `-since` to today; the **exact same value re-written is a no-op and must leave `-since` untouched** — a routine re-confirmation must never silently reset the clock; an explicit date always overrides the default; clearing a field always clears its `-since` too. `committed`/`committed-to` follow the same pairing *shape* as `waiting-on`/`waiting-since`, but with no date logic at all: `committed` is a plain date the user names explicitly (never auto-stamped), `committed-to` is free text valid only alongside it, and setting `committed-to` with no `committed` on record — and none being set in the same edit — is rejected outright. `core` and `focus` are booleans that exist only when true — set with `core: true` / `focus: true`, cleared by **removing the line**, never by writing `false`; `focus` is normally the morning digest's to set, not this skill's.
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
- **Person set/change:** "T9 — assign to Helen", "T12 is now on Ben", "T7 — person is Karl". Use the first name as given. `person` names another interested party (deliberately vague) — don't read it as ownership; the task isn't necessarily "theirs".
- **Tags set/change:** "T7 — tag it infra", "T12 add tags auth, api", "clear tags on T9".
- **Waiting-on set/change:** "T12 is waiting on Ed", "T9 — waiting on architecture forum sign-off", "chase Ed on T12, put that on the record". Free text — capture who or what the task is blocked on, verbatim. Not the same as `person`: don't set `person` just because `waiting-on` was set, or vice versa. This bucket includes a plain re-confirmation of the same value ("T12 is still waiting on Ed") — the dating rules in step 3 decide whether that resets the clock, not this step.
- **Waiting-on clear:** "Ed's unblocked T12", "T9 isn't waiting on anything now", "clear the wait on T7".
- **Blocking set/change:** "T15 is blocking Jason", "Jason's waiting on T15 now", "T7 — Karl needs this from me". Free text — who is owed the task. Also includes a plain re-confirmation of the same value; see step 3.
- **Blocking clear:** "T15 doesn't block Jason any more", "clear blocking on T9".
- **Append context only, no other change:** anything that adds a fact without announcing progress, a reclassification, or a field assignment. "T12 — Helen confirmed she'll come back Friday", "T19 — Ben's going to push back on the schema", "T7 needs the ADR signed off first".

If the intent is mixed (any combination of field changes + new context), do all of it in one pass.

### 3. Edit frontmatter (if any field changed)

Use the Edit tool. Apply all of the following that are relevant:

- **`state`:** replace `state: <current>` with `state: <new>`, where `<new>` is one of `open`, `in-progress`, or `on-hold` (`done`/`dropped` hand off to `task-dashboard:task-complete`).
- **`type`:** replace `type: <current>` with `type: <new>`, where `<new>` is exactly one of `action`, `idea`, or `issue` — reject anything outside that enum and stop rather than write a junk value.
- **`branch`:** set or replace the `branch:` line with the new value. If the field doesn't exist, insert it after `last-updated`. To clear, set `branch: null`.
- **`parent`:** set or replace the `parent:` line with the new value — a single task id, e.g. `parent: T193`. No `-since` companion; it's a plain scalar, same shape as `branch`. If the field doesn't exist, insert it after `branch` (or after `last-updated` if `branch` is absent). To clear, set `parent: null`. Before writing a new or changed value, run these checks in order and reject — stop, report, write nothing — on the first one that fails:
  1. The value matches `^T\d+$` (case-insensitive); anything else is malformed.
  2. The value isn't this task's own id — no self-parent.
  3. Read the candidate parent's file and check its `parent:` line; if it already points back at this task, that's a direct two-task cycle — reject it.
  Deeper cycles (three or more tasks pointing round in a ring) are deliberately out of scope here — the dashboard API backstops those on any write that goes through it. This skill only guards the two checks above; it does not walk the full parent graph.
- **`committed`:** a plain scalar date, `YYYY-MM-DD` — validate against `^\d{4}-\d{2}-\d{2}$` before writing; reject and stop on anything malformed rather than write a junk value. Set or replace the `committed:` line. If the field doesn't exist, insert it after `parent` (or after `branch`/`last-updated` if `parent` is absent). No `-since` companion and no auto-stamping — unlike `waiting-on`/`blocking`, this is a date the user names explicitly, not a clock reading. To clear, remove the line — clearing `committed` also clears `committed-to` (see below), even if the user only mentioned `committed`. **Soft-warn gate:** before writing a new or changed `committed` on a task that isn't `core: true`, tell the user committed dates are for core, externally-owed commitments and this task isn't marked core, then proceed once they confirm — a soft warning, not a hard block.
- **`committed-to`:** free text, meaningful only alongside `committed` — mirrors the *structure* of the `waiting-on`/`waiting-since` pairing but carries no date logic of its own. Set or replace the `committed-to:` line, inserted immediately after `committed`. Before writing a new or changed `committed-to`, check the task already has `committed` set, or that `committed` is being set in this same edit — if neither is true, reject and stop rather than write an orphaned value.
- **`person`:** set or replace the `person:` line. If the field doesn't exist, insert it after `last-updated` (or after `branch` if present). To clear, remove the line entirely.
- **`tags`:** set or replace the `tags:` line, e.g. `tags: [infra, auth]`. If the field doesn't exist, insert it after `last-updated`. To clear, set `tags: []`.
- **`waiting-on`:** before editing, read the CURRENT value already in the file (absent counts as no current value). Set or replace the `waiting-on:` line with the new value (free text — a person or a thing, e.g. `waiting-on: Ed` or `waiting-on: architecture forum sign-off`). If the field doesn't exist, insert it after `person` (or after `last-updated`/`project` if `person` is absent). Then, trimmed and compared exactly (case-sensitive) against the current value:
  - **New or changed** (including "was absent, now set"): also set `waiting-since` to **today's real date** (`YYYY-MM-DD`, read the clock, never copied from an example) — insert it immediately after `waiting-on`, or replace the existing line.
  - **The exact same value re-written — a no-op:** leave `waiting-since` completely untouched; do not touch that line at all. This is the single most important rule in this section — a routine re-confirmation ("still waiting on Ed") must never silently reset the clock.
  - **Explicit date given** ("waiting on Ed since 9 June"): use that date instead of today's, whether the value is new, changed, or unchanged — an explicit date always wins over the default.
  - To clear, remove both the `waiting-on:` and `waiting-since:` lines; clearing one always clears the other, even if the user only mentioned one.
- **`waiting-since`:** only ever set alongside `waiting-on`, per the rules above. Format `YYYY-MM-DD`. Never set `waiting-since` on a task with no `waiting-on`.
- **`blocking`:** identical rules to `waiting-on`, mirrored exactly. Before editing, read the CURRENT value already in the file. Set or replace the `blocking:` line with the new value (who is waiting on the task owner). If the field doesn't exist, insert it after `waiting-since` (or after `person`/`last-updated` if `waiting-on` is absent). Then, trimmed and compared exactly against the current value:
  - **New or changed:** also set `blocking-since` to today's real date — insert it immediately after `blocking`, or replace the existing line.
  - **The exact same value re-written — a no-op:** leave `blocking-since` completely untouched.
  - **Explicit date given:** use that date instead of today's — always wins over the default.
  - To clear, remove both the `blocking:` and `blocking-since:` lines; clearing one always clears the other.
  - `blocking`/`blocking-since` is independent of `waiting-on`/`waiting-since` — setting, changing, or clearing one pair never touches the other.
- **`blocking-since`:** only ever set alongside `blocking`, per the rules above. Format `YYYY-MM-DD`. Never set `blocking-since` on a task with no `blocking`.
- **`core` and `focus`:** boolean flags, both present-when-true and absent-when-off — there is no `core: false` or `focus: false` in this schema. To turn one on, write `core: true` / `focus: true` (insert after `tags`, or after `last-updated` if no other optional fields exist). To turn one off, **remove the line entirely.** Never write `false` — the repo auto-syncs across machines, and a `false` line is pure diff churn for a value the schema already treats as absent. `focus` is normally set by the morning digest rather than by hand; this skill can still set or clear it on an explicit request, but that's the less common path.

Always update `last-updated` to today's date in YYYY-MM-DD format. If `last-updated` is not yet in the frontmatter, add it immediately after `created`. Never touch `id`, `title`, `created`, or `project` — those are immutable.

### 4. Append context line (if there's new info)

One line, plain prose, ≤ 140 chars. **Append it with the `task-note` CLI**, which stamps the author, reads the wall-clock time, refreshes `last-updated`, formats the line so the dashboard renders it, stacks it with the right blank-line separation, and refuses to write to an archived task:

```bash
task-note T19 "Helen confirmed she'll come back by Friday on the Q2 numbers."
```

Pass the task id and the note text only — nothing else. `task-note` decides the author itself: `argus` in an Argus session (set by the `argus` repo's environment), `claude` in a general session. So you never write the author or the timestamp by hand, and Argus-authored notes stay distinguishable from a conversation Richard was actually in — the dashboard gives `argus` an eye glyph, `claude` a sparkles glyph. Don't add a heading like `## Updates`; one note per inflection; keep it tight.

**Fallback — only if `task-note` is not on PATH** (e.g. a machine without it): hand-edit instead. Append `claude YYYY-MM-DD HH:mm — text` to the end of the body, preceded by exactly one blank line so notes stack as blank-line-separated paragraphs (never a tight block), and refresh `last-updated` to today. Use the real wall-clock time; do not guess or round. Lead with `argus` instead of `claude` only when you are operating as the Argus system; never write `user` from a skill (that prefix is the dashboard UI's). Don't reformat existing content.

For a genuine blocker or warning that deserves visual emphasis, the one fact may instead be appended as a `> [!warning]` callout (two physical lines: `> [!warning]` then `> <text>`), which the dashboard renders as a tinted block. Use sparingly — one callout per genuinely important inflection, never for routine notes.

Whether or not a context line is appended, always update `last-updated` in the frontmatter to today's date.

### 5. Report

```
✏️  T12 → type: action · state: in-progress     <task-data>/tasks/t12-foo.md
     + "claude 2026-05-11 14:32 — Helen confirmed she'll come back by Friday."
```

Include only the parts that changed on the `→` line: `type:`, `state:`, `branch:`, `parent:`, `person:`, `tags:`, `waiting-on:`, `blocking:`, `committed:`, `committed-to:`, `core:`, and/or `focus:`, joined by ` · ` when more than one changed. (`waiting-since` and `blocking-since` are never shown separately — each always travels with its parent field.) If `waiting-on` or `blocking` was "changed" only in the sense of being re-confirmed with the exact same value, nothing on disk actually changed — omit it from the `→` line entirely, same as any other no-op field. `core`/`focus` show as `core: true` / `focus: true` when the line is set, and `core: (cleared)` / `focus: (cleared)` when the line is removed — never `core: false`.
If no frontmatter changed (line appended only): omit the `→` line.
If only a line was appended (no frontmatter change): omit the `→` line; if no line was appended, omit the `+ "…"` line.

## Examples

**Example 1 — state only**

User: *"T19 — starting on it now"*

- Find T19, set `state: in-progress`. No body line.

**Example 2 — context only**

User: *"T22 — Ben pushed back on the schema; want to keep the date string instead of converting to ISO"*

- Find T22, no state change. Append a body line summarising the push-back: "claude 2026-05-11 14:30 — Ben prefers keeping the date string; doesn't want ISO conversion."

**Example 3 — both**

User: *"T7 in flight — got Charlotte's sign-off, just need to write the diff"*

- Find T7, set `state: in-progress`. Append: "claude 2026-05-11 14:30 — Charlotte signed off; remaining work is the diff."

**Example 4 — type promotion (idea → action), with state**

User: *"right, let's actually do T9 — it's an action now, starting it"*

- Find T9, set `type: action` and `state: in-progress`. Append: "claude 2026-05-11 14:30 — Promoted from idea to action; starting implementation." Report line: `T9 → type: action · state: in-progress`.

**Example 5 — type only**

User: *"T15 isn't an idea, it's a bug"*

- Find T15, set `type: issue`. No state change. Append a one-line note only if the user added a fact; a bare reclassification needs no body line. Report line: `T15 → type: issue`.

**Example 6 — parking to on-hold**

User: *"park T12 for now — waiting on the vendor to confirm the format"*

- Find T12, set `state: on-hold` (parking language targets on-hold, never `open`). Append: "claude 2026-06-09 10:15 — Parked pending vendor confirmation of the format." Report line: `T12 → state: on-hold`.

**Example 7 — branch set**

User: *"T19 — branch is t19-auth-refactor"*

- Find T19, set (or add) `branch: t19-auth-refactor`. No state/type change, no body line needed unless the user added a fact. Report line: `T19 → branch: t19-auth-refactor`.

**Example 8 — person change + context**

User: *"T7 — moving to Karl, he's picking this up now"*

- Find T7, set `person: Karl`. Append: "claude 2026-06-11 14:30 — Karl taking this over." Report line: `T7 → person: Karl`.

**Example 9 — waiting-on set, then cleared**

User: *"T12 is waiting on Ed"*

- Find T12, set `waiting-on: Ed` and `waiting-since:` to **today's real date** in `YYYY-MM-DD` (no date was given, so the wait starts now). Read the clock; never copy a date from an example. No state change — `state` stays whatever it already was. Report line: `T12 → waiting-on: Ed`.

Later, user: *"Ed's sorted T12"*

- Find T12, remove both `waiting-on:` and `waiting-since:`. Report line: `T12 → waiting-on: (cleared)`.

**Example 10 — blocking set**

User: *"T15 is blocking Jason — he's waiting on my part"*

- Find T15, set `blocking: Jason` and `blocking-since` to today's real date (no date was given). No change to `waiting-on` — the two are independent. Report line: `T15 → blocking: Jason`.

**Example 11 — waiting-on re-confirmed with the same value (the trap)**

T12 already has `waiting-on: Ed` and `waiting-since: 2026-05-11`. User: *"T12 is still waiting on Ed"*

- Find T12. The new value ("Ed") is identical, trimmed, to what's already on disk — a no-op. Leave `waiting-since` exactly as it was, `2026-05-11`; do **not** stamp today. No state change. Since nothing on disk actually changed, omit `waiting-on:` from the `→` line entirely — append a context line only if the user volunteered new information beyond the re-confirmation itself.

**Example 12 — blocking re-confirmed, then genuinely changed**

T15 already has `blocking: Jason` and `blocking-since: 2026-06-01`. User: *"Jason's still waiting on T15"*

- Find T15. Same value — a no-op. Leave `blocking-since` untouched at `2026-06-01`. Omit the `→` line.

Later, user: *"actually it's Karl chasing T15 now, not Jason"*

- Find T15. `blocking` changes from `Jason` to `Karl` — a genuine change, no date given, so `blocking-since` re-stamps to today's real date. Report line: `T15 → blocking: Karl`.

**Example 13 — parent set**

User: *"T50 sits under T12 now"*

- Find T50, check T12 exists and isn't T50 itself, and read T12's file to confirm its `parent` isn't already `T50` (no direct cycle). Set `parent: T12`. Report line: `T50 → parent: T12`.

**Example 14 — parent rejected: self and two-cycle**

User: *"T50's parent is T50"*

- Reject: a task can't be its own parent. Stop and report; write nothing.

Later, with T50's `parent: T12` already on disk, user: *"T12 — set its parent to T50"*

- T12's candidate parent (T50) already has `parent: T12` on disk — a direct two-task cycle. Reject, stop, and report; write nothing.

**Example 15 — committed + committed-to, soft-warn gate**

User: *"T31 — we've committed to the board for 25 July on this"*

- Find T31; T31 isn't `core: true`. Warn: committed dates are for core, externally-owed commitments, and T31 isn't marked core — ask whether to proceed anyway. User confirms.
- Validate the date against `^\d{4}-\d{2}-\d{2}$` — `2026-07-25` passes. Set `committed: 2026-07-25` and `committed-to: the board`. Report line: `T31 → committed: 2026-07-25 · committed-to: the board`.

**Example 16 — committed-to rejected: no committed on record**

User: *"T44 — committed-to is the audit committee"* (T44 has no `committed` field, and none is being set in this edit)

- Reject: `committed-to` is only meaningful alongside `committed`, and T44 has neither an existing value nor one being set now. Stop and report; write nothing.

**Example 17 — core and focus, on then off**

User: *"T9 is core"*

- Find T9, add the line `core: true`. Report line: `T9 → core: true`.

Later, user: *"T9 isn't core any more"*

- Find T9, **remove the `core: true` line entirely** — never rewrite it as `core: false`. Report line: `T9 → core: (cleared)`. The same handling applies to `focus`: `focus: true` to set, the line removed (never `focus: false`) to clear.

## Boundaries

- **Never edit `id`, `title`, `created`, or `project`.** Those changes warrant opening the file in the editor; surface that and stop.
- **Mutable frontmatter fields this skill may edit:** `state`, `type`, `branch`, `parent`, `person`, `tags`, `waiting-on`, `waiting-since`, `blocking`, `blocking-since`, `committed`, `committed-to`, `core`, `focus`, and `last-updated`. Nothing else.
- **`core` and `focus` are present-when-true, absent-when-off — there is no `core: false` or `focus: false` in this schema.** To turn one on, write `core: true` / `focus: true`; to turn one off, remove the line entirely. Writing `false` is never correct: the repo auto-syncs across machines, and a `false` line is pure diff churn for a value the schema treats as absent anyway.
- **`committed-to` is only ever written alongside `committed`.** Setting `committed-to` on a task with no `committed` on record, and none being set in the same edit, is rejected — stop and write nothing. Clearing `committed` always clears `committed-to` too, even if the user only mentioned `committed`. Before writing a new or changed `committed` on a task that isn't `core: true`, soft-warn — committed dates are for core, externally-owed commitments — and proceed once the user confirms; it's a nudge, not a block.
- **`waiting-on`/`waiting-since`/`blocking`/`blocking-since` never drive `state`, and `state` never drives them.** Do not set `state: on-hold` because `waiting-on` was set, and do not infer `waiting-on` or `blocking` from a state change. They are orthogonal facts tracked side by side.
- **`waiting-on` is not `person`.** Never copy one field's value into the other or assume they should match.
- **A same-value re-write of `waiting-on` or `blocking` must never reset its `-since`.** Always compare against the value currently on disk before deciding whether to stamp a new date — see step 3. Resetting the clock on a routine re-confirmation is the one mistake that makes the whole age-tracking feature worthless.
- **A `type` change must land inside the `action | idea | issue` enum.** Anything else is rejected; stop and report rather than write a junk value.
- **A `state` change must land inside the `open | in-progress | on-hold` enum.** `done`/`dropped` hand off to `task-dashboard:task-complete`.
- **A `parent` set or change must pass format, self-parent, and direct two-cycle checks before it's written.** Malformed values, self-parent, and a direct A↔B cycle are all rejected — stop and write nothing. Deeper (3+ node) cycles are not checked here; the dashboard API backstops those on API-driven writes.
- **Always refresh `last-updated` to today's date** on every invocation, even if only a body line was appended.
- **Never edit a file in `<task-data>/tasks/archive/`.** Archived means closed.
- **Don't add more than one body line per invocation.** If the user volunteers multiple updates in one breath, condense to one line, or ask which line they want. The risk of long appended monologues is that the file body bloats and the dashboard preview becomes useless.
- **Don't fall through to `task-dashboard:task-create` if the ID isn't found.** Stop and report. A missing ID is more likely a typo than a missing task.
