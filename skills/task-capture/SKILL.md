---
name: task-capture
description: Use when the user says "/task-dashboard:task-capture", "capture tasks", "log tasks", "wrap up tasks", or invokes /task-dashboard:task-capture — scans the current conversation, writes new task files into the task-data tasks directory for any commitments raised, and updates existing task files when the conversation has moved them on. Pairs with /retro at end of session.
---

# Task Capture

End-of-session sweep. Reads the conversation in full, identifies real commitments, writes or updates markdown task files under `<task-data>/tasks/`. Auto-applies; report at the end.

The user is running this deliberately at session close, so don't ask for confirmation per item — write the files and report. Anything wrong, the user deletes or marks `state: dropped` afterwards.

## Path resolution

All paths below refer to the **task-data repository**. Resolve its location in this order:

1. The `TASK_DATA_DIR` environment variable, if set.
2. Otherwise `~/projects/task-data` on macOS/Linux, or `%USERPROFILE%\projects\task-data` on Windows 11.

In all commands below, substitute the resolved path for `<task-data>`.

## Phase 1: Identify candidates

Re-read the conversation from the top. Produce two lists.

### 1a. Explicit ID mentions

Scan for any token matching `\bT\d{1,4}\b` — the literal letter `T` followed by 1-4 digits. Examples the user is likely to dictate: `T1`, `T12`, `T42`, `T123`. The pattern is deliberately tight so that random letter-and-number combos in conversation (`S36`, `ADR099`, `B12`) don't generate spurious mentions. False positives that slip through still get filtered in Phase 2 because they won't match a real id on disk.

For each ID mention, capture the surrounding context (the sentence or two that referenced it) and what the user said about it: "T12 is done", "still waiting on T19", "drop T7", "T22 — Ben pushed back on the schema". These bypass fuzzy matching entirely; they are direct routes to a specific file in Phase 2.

### 1b. Implicit candidates

Build a list of new commitments the conversation surfaced, each tagged as one of:

- **Action** — the user said they will do something concrete. "I'll write that up." "Let me draft the ADR." "Need to bump the SRI hashes." Verb plus the user as the actor.
- **Idea** — the user parked something for later that isn't an immediate action. "Worth thinking about a stale-task nudge later." "Could be neat to add a search box."
- **Issue** — something is wrong and needs fixing. A deficiency, bug, or broken behaviour that surfaced in conversation.

**Skip rules — do NOT capture as an implicit candidate:**

- Generic agreement or acknowledgement ("ok", "sounds good", "let's do that") with no concrete verb.
- Things the user has just done in the same session — the work is already complete; if anything, it might warrant updating an existing task to `done`, not a new capture.
- Decisions or facts (those go in the daily journal or a wiki page).
- Hypothetical "we could" statements that the user immediately drops or moves past.
- Anything the user said is already tracked elsewhere (Linear, Outlook tasks, calendar events, ADRs).
- Commitments about the task-dashboard project itself raised inside a task-dashboard session — they're already on the dev plan; capturing them in their own list is meta-noise.

When in doubt: skip. The cost of a missed capture is one re-mention next session; the cost of a false capture is noise that erodes trust in the whole list.

Within the same session, deduplicate implicit candidates: if the user mentioned the same commitment twice ("I should chase Gav" early on, then "yeah, definitely chasing Gav" later), keep one.

## Phase 2: Reconcile against existing tasks

Two routes, in order. Stop at the first one that resolves a candidate.

### 2a. Direct lookup for explicit ID mentions

For each ID mention from Phase 1a:

1. Find the file. `grep -l "^id: <ID>$" <task-data>/tasks/**/*.md` (active and archive). One match expected; if zero, log a warning ("ID mentioned but not on disk: <ID>") and move on. If more than one, log and pick the most recently modified.
2. Apply the state change implied by the conversation, same rules as 2b below.
3. Mark the ID as handled so Phase 2b doesn't double-process anything that overlaps.

### 2b. Fuzzy match for implicit candidates, scoped to the current project

Compute the **current project id**: take the absolute path of the current working directory and replace every path separator with `-`. On POSIX replace `/`; on Windows replace `\` and the drive colon.

Narrow the candidate pool to tasks whose `project` matches:

```bash
grep -l "^project: <current-project-id>$" <task-data>/tasks/*.md \
  <task-data>/tasks/archive/*.md
```

This is the optimisation — even with hundreds of tasks across many projects, only the handful for the current cwd ever get title-and-body inspected. For each implicit candidate from Phase 1b, only this narrowed list is searched.

If the current cwd is outside any encoded project (e.g. a one-off scratch session), skip 2b's fuzzy matching entirely — every implicit candidate becomes a new capture in Phase 3. Better than guessing across the whole pool.

Within the narrowed pool:

1. Look for an existing task with the same person AND a clearly-overlapping subject (substring match on title, plus a sanity check on the body context). Match generously — the goal is not to duplicate.
2. **If a match is found, apply the state rules:**
   - The conversation says it's done → edit frontmatter `state: done`, append a one-line note to the body (e.g. "Resolved 10 May 2026 in session.").
   - The conversation says it's started but not finished → edit `state: in-progress`, append one-line context.
   - The conversation says it's dropped → edit `state: dropped`, append a one-line reason.
   - New context landed (a date, a name, a blocker) but state is unchanged → append one short line to the body, do not touch frontmatter.
   - The same commitment is restated without progress → leave the file alone.
   - Move the candidate from "new captures" to "updates".
3. **If no match:** keep on the new-captures list for Phase 3.

## Phase 3: Write the files

For each remaining new capture, allocate the next sequential id, then write `<task-data>/tasks/t<n>-<slug>.md` with frontmatter:

```yaml
---
id: T<n>                         # required; pool-wide sequential, no padding
title: <short title, ≤ 80 chars> # required
type: action                     # required: action | idea | issue
state: open                      # required (default: open)
created: YYYY-MM-DD              # required
project: <encoded id or null>    # optional; absolute path of cwd with separators replaced by -
person: <name>                   # optional; for related user
tags: [tag1, tag2]               # optional; leave off if nothing meaningful
---

<one or two short paragraphs of context: why this is on the list, what's
blocking it, what the next concrete step is. Plain markdown.>
```

The dashboard renders `> [!type]` alert callouts (lowercase `note`, `info`, `tip`, `important`, `warning`, `caution`) as tinted, icon-led blocks — reach for one when a capture carries a genuine warning or important caveat, but keep routine captures plain.

Slug rules: lowercase, hyphenated, derived from the title, ≤ 40 chars. If a file with that name already exists, suffix `-2`, `-3`, etc.

Sequence number rules: glob existing `<task-data>/tasks/**/*.md` (active and archive), parse out every `id: T<digits>`, take the max integer, increment by one. Numbers are never reused. If emitting more than one new task in this run, allocate them in order — Tn, Tn+1, Tn+2 — so the ids reflect capture order within the session.

Project encoding: take the absolute path of the current working directory and replace every path separator with `-`. If the session has no clear project home, use `null`.

## Phase 4: Report

End with one consolidated block, no per-file chatter during the writes:

```
📋 Captured N new tasks:
- Action  · <title>                     <relative path>
- F-up    · <title> · @<person>         <relative path>
- Idea    · <title>                     <relative path>
- Issue   · <title>                     <relative path>

Updated M existing tasks:
- T12    → state: done                  <relative path>           (direct: ID mention)
- T19    → context appended             <relative path>           (direct: ID mention)
- <title> → state: in-progress          <relative path>           (fuzzy match)

Skipped K candidates (already on file, no change).
Warnings:
- ID mentioned but not on disk: T99
```

If nothing was captured or updated, say "Nothing to capture." and stop. Omit the Warnings line when empty.

Never log full task bodies in the report — titles only. The bodies are on disk and the user reviews there.
