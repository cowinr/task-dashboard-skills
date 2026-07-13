---
name: task-create
description: Use whenever the user wants to capture a new task, action, idea, or follow-up into the personal task dashboard, from any project on the machine. Triggers on phrases like "add a task", "new task", "capture this as a task", "log this", "stick this on my list", "remind me to chase X", "idea for later", "I need to follow up with Y about Z", "put this on the dashboard", or `/task-dashboard:task-create`. Writes a correctly-formatted markdown file with frontmatter into the task-data tasks directory, allocating the next pool-wide sequential T-id. Works from any current working directory; the dashboard lives outside the current project.
---

# Task Create

Ad-hoc, in-session task capture. Sibling to the end-of-session `task-dashboard:task-capture` skill, but for a single task surfaced mid-conversation, anywhere on the machine.

## Path resolution

All paths below refer to the **task-data repository**. Resolve its location in this order:

1. The `TASK_DATA_DIR` environment variable, if set.
2. Otherwise `~/projects/task-data` on macOS/Linux, or `%USERPROFILE%\projects\task-data` on Windows 11.

Examples:
- macOS/Linux: `/Users/alice/projects/task-data`
- Windows 11: `C:\Users\alice\projects\task-data`

In all bash commands below, substitute the resolved path for `<task-data>`.

## What you produce

One file at `<task-data>/tasks/t<n>-<slug>.md` with frontmatter and a short body. Schema:

```yaml
---
id: T<n>                              # required; pool-wide sequential, no padding
title: "<short title, ≤ 80 chars>"    # required
type: action                          # required: action | idea | issue
state: open                           # required (default: open)
created: YYYY-MM-DD                   # required; today's date
last-updated: YYYY-MM-DD              # required; same as created at creation time
project: <encoded id or null>         # optional; absolute path of cwd, / → - (POSIX) or \ → - (Windows)
person: <name>                        # optional; only when there's a named human counterparty
waiting-on: <name or thing>           # optional; who or what this task is already blocked on — rare at creation, usually added later via task-update
waiting-since: YYYY-MM-DD             # optional; only set alongside waiting-on — defaults to today if waiting-on is given without a date
blocking: <name>                      # optional; who is already waiting on the task owner for this — rare at creation, usually added later via task-update
blocking-since: YYYY-MM-DD            # optional; only set alongside blocking — defaults to today if blocking is given without a date
tags: [tag1, tag2]                    # optional; ONLY names from tags-master-list.json (see step 5); omit if none fit
---

<one or two short paragraphs of context: why this is on the list, what's
blocking it, what the next concrete step is. Plain markdown. No heading.>
```

**Type:**
- `action` — the user (or a named counterparty) needs to do a concrete thing.
- `idea` — parked for later; nothing actionable yet.
- `issue` — something is wrong and needs fixing (a deficiency, bug, or broken behaviour).

Note: there is no `follow-up` type. A task waiting on someone is still an `action`; set the `person` field to the human involved.

**Project encoding:** take the absolute path of the current working directory and replace every path separator with `-`. On POSIX, replace every `/`; on Windows, replace every `\` (and the drive colon). Examples:
- POSIX: `/home/alice/projects/myapp` → `-home-alice-projects-myapp`
- Windows: `C:\Users\alice\projects\myapp` → `C-Users-alice-projects-myapp`

If the cwd is outside any meaningful project (e.g. the user's home directory), use `null`.

## Workflow

### 1. Pin down title, type, person

Read what the user just said. Distil it into:
- A short, content-rich title. Skip filler ("need to", "should", "remember to"). Drop trailing punctuation.
- A type (`action`, `idea`, or `issue`). Default to `action` if the verb is concrete and current; `idea` if the framing is "could be neat", "worth exploring later", "thought for the future"; `issue` if something is wrong and needs fixing.
- A `person`, if and only if a specific human is named in the commitment ("chase Helen", "ask Ben about X"). Use the first name as written. Don't infer a person where none is mentioned. `person` names another interested party connected to the task — a counterparty or stakeholder. It's deliberately vague, so don't read it as ownership: the task isn't necessarily "theirs".

If the title or type genuinely isn't obvious (rare), ask one short question. Otherwise proceed.

### 2. Allocate the next ID

Glob both the active and archive directories, parse every `id: T<n>` line, take the max, increment by one. Numbers are never reused.

```bash
grep -hE '^id: T[0-9]+' \
  <task-data>/tasks/*.md \
  <task-data>/tasks/archive/*.md 2>/dev/null \
  | sed -E 's/^id: T([0-9]+).*/\1/' \
  | sort -n | tail -1
```

On Windows (PowerShell), equivalent:
```powershell
Select-String -Path "<task-data>\tasks\*.md","<task-data>\tasks\archive\*.md" `
  -Pattern '^id: T(\d+)' | ForEach-Object { $_.Matches[0].Groups[1].Value } |
  Measure-Object -Maximum | Select-Object -ExpandProperty Maximum
```

If that returns nothing (empty repo), start at 1.

### 3. Build the filename

`t<n>-<slug>.md`:
- `t` is lowercase, `<n>` is the integer with no padding.
- `<slug>` is the title: lowercased, non-alphanumerics replaced with `-`, collapsed runs of `-`, trimmed, ≤ 40 chars (cut at a word boundary if possible).
- If a file with that name already exists in either `<task-data>/tasks/` or `<task-data>/tasks/archive/`, append `-2`, `-3`, etc. (Collisions are rare since IDs are unique; the suffix is belt-and-braces.)

### 4. Resolve the project field

POSIX:
```bash
pwd | sed 's|/|-|g'
```

Windows (PowerShell):
```powershell
(Get-Location).Path -replace '[\\:]', '-'
```

If the resulting string doesn't look like a real project (e.g. just the home directory), set `project: null`.

### 5. Select tags from the controlled vocabulary

Tags are a **controlled vocabulary**. The only tags you may put on a new task are those already catalogued in `<task-data>/tags-master-list.json`. Never invent a tag, and never carry a project name, a `personal`/`work` label, or any free-text descriptor onto a task just because it seems apt — if it is not in the master list, it does not go on.

1. Read `<task-data>/tags-master-list.json`. Its shape is:

   ```json
   { "categories": { "<category>": [ { "name": "azure", "description": "…" }, … ] } }
   ```

   The allowed tag set is every `name` across every category.

2. From what the user said, pick only the catalogued tags that clearly apply. Match case-insensitively, but write each tag using the **canonical casing from the file**. Be conservative: a tag must be an obvious fit, not a stretch. Most tasks get zero or one tag.

3. If nothing in the vocabulary clearly applies, **omit the `tags` line entirely**. An absent `tags` is always correct; an off-list tag never is.

4. If `tags-master-list.json` does not exist (older setups), omit `tags` — there is no vocabulary to draw from, and this skill must not create one.

Growing the vocabulary is the dashboard's Tag Manager's job, never this skill's. If the user explicitly wants a brand-new tag, capture the task without it and tell them to add the tag via the dashboard's **Tags** manager first, then re-tag the task.

### 6. Write the file

Use the Write tool. Frontmatter first, then a blank line, then 1-2 short paragraphs of context. Do **not** include a Markdown heading — the dashboard renders the title from frontmatter, and a duplicate `#` heading creates visual noise.

The dashboard renders GitHub-style alert callouts: a `> [!type]` blockquote (lowercase `note`, `info`, `tip`, `important`, `warning`, `caution`) shows as a tinted, icon-led block. Reach for one when the body has a genuine warning, tip, or piece of important context — don't force it on routine tasks.

Set `last-updated` to the same value as `created` (both are today's date on a new task).

Quote the title only if it contains a colon or other YAML-significant character. Otherwise leave it unquoted, matching the existing files.

### 7. Report

One short block, no per-step chatter:

```
📋 Captured T<n>: <title>
   type:    action
   project: -home-alice-projects-myapp   (or "null")
   person:  Helen                         (omit line if no person)
   file:    <task-data>/tasks/t<n>-<slug>.md
```

If the user mentioned the wrong wording, they'll edit the file directly. Don't ask for confirmation up front — the friction defeats the purpose of an ad-hoc capture.

## Examples

**Example 1 — straight action**

User: *"add a task: I need to chase Helen about the Q2 numbers"*

Output:
- type: action
- title: Chase Helen about Q2 numbers
- person: Helen
- body: One line of context naming what's outstanding.

**Example 2 — idea**

User: *"throw this on the list as an idea — would be neat to auto-archive tasks that have been done > 30 days"*

Output:
- type: idea
- title: Auto-archive tasks older than 30 days
- no person
- body: One sentence on the motivation.

**Example 3 — no project context**

User running from their home directory says: *"capture this: book MOT for the car"*

Output:
- type: action
- project: null
- title: Book MOT for the car

## Boundaries

- One task per invocation. If the user rattles off three things, ask which they meant or do them as three writes — but don't merge unrelated commitments into one file.
- Don't update existing tasks here. That's `task-dashboard:task-update`'s job. If the user says "T12 is done" while invoking this skill, redirect to `task-dashboard:task-complete` / `task-dashboard:task-update`.
- Tags are a controlled vocabulary (see step 5): only ever apply tags already in `tags-master-list.json`, matched case-insensitively against the canonical casing. Never invent tags, never use a project name or "personal"/"work" as a tag, and omit `tags` whenever nothing in the master list clearly fits. New vocabulary entries are added through the dashboard's Tag Manager, not here.
- Never log or echo the body of other tasks. This skill writes one new file; it does not browse.
