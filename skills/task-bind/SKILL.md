---
name: task-bind
description: Use whenever the user wants to bind a task to the current session so its file tracks the work as it happens. Triggers on phrases like "bind T112", "work on T112", "let's do T112", "starting on T112", "I'm working on T112 today", "pick up T112", "focus on T112", "/task-dashboard:task-bind T<n>", or any sentence naming a `T<digits>` id alongside starting/focusing language. Loads the task, emits a compact bind block, and establishes a standing session contract that routes material progress back into the file via the other task-* skills. Read-only at load time; all writes delegate to `task-dashboard:task-update` or `task-dashboard:task-complete`.
---

# Task Bind

Bind a task to the current session. Two parts, bolted together:

1. **The loader** — read-only at bind time. Finds the task, surfaces its full state in a compact block, and prints a one-line binding restatement that survives later context.
2. **The session contract** — a standing intent that, for the rest of the session, this is the task being worked, and material progress flows back into its file. A skill runs once and cannot intercept later turns; the contract is a primed instruction and may fade under a long session or compaction. Worst case it degrades to today's behaviour — a `/retro` + `task-dashboard:task-capture` at session end still reconciles. That is an accepted risk, not a failure.

## Path resolution

All paths below refer to the **task-data repository**. Resolve its location in this order:

1. The `TASK_DATA_DIR` environment variable, if set.
2. Otherwise `~/projects/task-data` on macOS/Linux, or `%USERPROFILE%\projects\task-data` on Windows 11.

In all commands below, substitute the resolved path for `<task-data>`.

## Workflow

### 1. Find the task file

Active tasks live in `<task-data>/tasks/`; archived ones in `<task-data>/tasks/archive/`. Check active first.

```bash
grep -l "^id: T<n>$" <task-data>/tasks/*.md 2>/dev/null
```

If zero matches in active, check archive:

```bash
grep -l "^id: T<n>$" <task-data>/tasks/archive/*.md 2>/dev/null
```

If found in the archive, report it: "T\<n\> is in the archive — it is already closed. Reopen by restoring the file first, or bind a different task." Stop. Do not bind an archived task.

If more than one active match (should not happen), use the most recently modified and warn.

If not found anywhere, report: "T\<n\> not found in active tasks or archive." Stop.

### 2. Read the file

Read the full file — frontmatter and body. Extract:

- `id`, `title`, `type`, `state`
- `person`, `project`, `tags` (any that are present)
- `related` — a YAML list in frontmatter (e.g. `related:\n  - T113`); extract all entries. This is the authoritative source.
- `branch`, `pr`, `jira` — frontmatter fields. `branch` is a plain string; `pr` and `jira` are URLs (surface the URL or its human-readable key). These are the authoritative source.
- The next concrete step, if stated explicitly in the body; otherwise summarise from context.

If `related`, `branch`, `pr`, or `jira` are absent from the frontmatter, they are simply absent — do not scan the body as a substitute for these structured fields. A body scan for incidental `T\d+` mentions may be offered as a clearly-secondary "also seen in body" supplement only if it adds something the frontmatter did not already provide, and must be labelled as such.

After reading, open the file in the editor so it is on screen as the bind lands:

```bash
code -r <task-data>/tasks/<the-task-file>.md 2>/dev/null || true
```

Use `-r` (reuse window) so a rebind does not spawn a fresh window each time. This is a read-only side effect and does not violate the read-only-at-bind-time rule — the loader still never writes to the task file. If the `code` CLI is not on PATH the command fails silently and the bind proceeds unaffected; never block or warn on it.

### 3. Emit the compact bind block

```
📋 Bound: T<n> — <title>

   id:      T<n>
   type:    <type>
   state:   <state>
   person:  <person>          ← omit line if absent
   project: <project>         ← omit line if absent
   tags:    [<tags>]          ← omit line if absent / empty
   related: T<x>, T<y>       ← omit line if none found
   branch:  <branch>          ← omit line if none found
   PR:      <pr ref>          ← omit line if none found
   Jira:    <ticket>          ← omit line if none found

   Next step: <one concrete sentence from the task body, or a summary>
```

### 4. Print the binding restatement

One line, immediately after the block, so it survives later scrollback:

> Bound to T\<n\> — \<type\>, \<state\>. Next: \<next step, ≤ 80 chars\>.

### 5. Suggest a matching session name

Claude Code cannot rename a running session programmatically (`/rename` is interactive only). So emit a ready-to-paste suggestion line, immediately after the restatement, so the session name can be synced in one keystroke:

> 💡 Match your session name: `/rename T\<n\> \<title\>`

Use the task's `title` verbatim. Print this once on bind only; never nag about it again.

### 6. Engage the session contract

After the bind block and restatement, state the standing contract exactly once, briefly:

> Session contract active. Material progress on T\<n\> will be reflected back into its file via task-dashboard:task-update or task-dashboard:task-complete. Chatter turns write nothing.

Do not repeat this notice on subsequent turns; it is a one-time establishment.

---

## Session contract — auto-write rules

For the rest of this session, the bound task's file must stay true. Three rules, applied at every turn where T\<n\> is relevant:

### Rule A — Decision to build (idea → action)

When the user makes a clear decision to proceed with implementation of an idea:

- **Delegate the whole transition to `task-dashboard:task-update` in a single call.** Set `type: action` and `state: in-progress` together, with its `last-updated` refresh. Do not make a special-case frontmatter edit here — route everything through `task-dashboard:task-update`.
- This is the only time `type` changes under the session contract.
- Only fires on a clear "we're building this" signal, not on exploratory discussion.

### Rule B — Real blocker or material decision

When a genuine blocker surfaces, or a decision is made that changes the task's trajectory:

- **Delegate to `task-dashboard:task-update`: append one disciplined line.**
- Format: canonical note format, plain prose, ≤ 140 chars. Lead with `claude`, today's date in `YYYY-MM-DD`, and the current wall-clock time in `HH:mm` (24-hour). Example: `claude 2026-05-19 14:32 — Blocked on Ben's sign-off before the schema can be finalised.`
- No `## Updates` heading. No more than one line per turn.
- Only fires when something genuinely changes the task's direction or blocks progress. Meeting notes, side discussion, and chatter do not qualify.

### Rule C — No material change

Any turn that advances the conversation without a clear inflection point for T\<n\> writes nothing to the file. Zero writes.

**The ceiling for auto-write is one line per inflection. Running commentary is the explicit failure mode.**

### Re-read before every write

**Never trust the load snapshot for a write.** At the point of every write — Rule A or Rule B — re-read the file first, then construct the edit on what the file actually contains. A mid-session hand-edit by the user must never be clobbered by a stale in-memory copy.

---

## Report formats

### Bind confirmation (Phase 1 output)

The compact block from step 3 above, followed by the restatement line. Nothing else on a clean bind. Example:

```
📋 Bound: T<n> — <title>

   id:      T<n>
   type:    action
   state:   in-progress
   project: -home-alice-projects-some-project
   related: T<x>, T<y>

   Next step: <the next concrete step from the task body>.

Bound to T<n> — action, in-progress. Next: <next step, ≤ 80 chars>.

💡 Match your session name: /rename T<n> <title>

Session contract active. Material progress on T<n> will be reflected back into its file via task-dashboard:task-update or task-dashboard:task-complete. Chatter turns write nothing.
```

### Auto-update echo (Rule A or Rule B fires)

After a delegated write completes, echo the task-update or task-complete report block exactly as that skill produced it. Do not add a second summary; the report from the delegated skill is the echo.

---

## Rebinding

If `/task-dashboard:task-bind T<m>` is invoked while T\<n\> is already bound:

1. Note the replacement: "Replacing binding T\<n\> → T\<m\>."
2. Run the full loader workflow for T\<m\>.
3. The old binding is gone; no deferred write for T\<n\> is carried over.

There is one active binding per session. Rebind replaces cleanly.

---

## Boundaries

- **One binding per session.** Rebind replaces; two concurrent bindings are not possible.
- **Read-only at bind time.** The loader never writes. A bind with no subsequent write is correct behaviour.
- **Delegate, never reimplement.** Every write goes through `task-dashboard:task-update` (state, type, context) or `task-dashboard:task-complete` (done/dropped + archive). No second copy of the transition logic, ID lookup, or report formats lives here.
- **Completion stays explicit.** This skill may bring a task to `in-progress` and append context lines, but it never decides the task is done or dropped. Closing is always a deliberate `task-dashboard:task-complete`.
- **Checklist behaviour is not in scope.** Ticking `- [ ]` items, tracking checklist progress, and reporting completion percentages are deferred to a later task. Checklists stay ordinary body text. This is a deliberate cut (19 May 2026).
- **Never auto-archive.** The archive sweep is always triggered by `task-dashboard:task-complete`, never by this skill.
- **Never bind an archived task.** If the id is found only in `<task-data>/tasks/archive/`, report and stop.
- **Never write to a file in `<task-data>/tasks/archive/`.** If a delegated write would land there, stop and report the anomaly.
- **`last-updated` is always refreshed on every write.** The delegated skills already do this; the instructions to them must not suppress it.
- **Do not fall through to `task-dashboard:task-create` if the ID is not found.** Stop and report. A missing ID is a typo or a task that was never written, not an invitation to create one.
- **The session contract is a soft commitment, not a hard interceptor.** It can fade under a very long session or context compaction. This is the accepted trade-off. Mitigation: the compact restate on load, re-reading before every write, and `/retro` + `task-dashboard:task-capture` as the end-of-session safety net.
