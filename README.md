# task-dashboard plugin

A Claude Code plugin that bundles six personal task-management skills into one installable unit. Tasks are plain markdown files stored in a `task-data` git repository; the skills read and write them directly without any server or database.

Built to work on both macOS/Linux and Windows 11.

## The six skills

| Skill | What it does |
|-------|-------------|
| `task-dashboard:task-create` | Capture a single new task mid-conversation (action, idea, or issue). Allocates the next sequential T-id and writes one markdown file. |
| `task-dashboard:task-update` | Edit an existing task by ID: change state, type, branch, person, or tags; append a one-line context note. Does not close tasks. |
| `task-dashboard:task-complete` | Close a task as `done` or `dropped`, append a dated resolution note, then run the archive sweep to move the file to the archive directory. |
| `task-dashboard:task-list` | Read-only summary of active tasks in the terminal — filter by person, project, state, type, or tag. Quick alternative to opening the dashboard. |
| `task-dashboard:task-bind` | Bind a task to the current session. Surfaces the task's full state and establishes a standing contract to route material progress back into the file. Delegates all writes to `task-update` or `task-complete`. |
| `task-dashboard:task-capture` | End-of-session sweep. Scans the conversation, writes new task files for commitments surfaced, and updates existing ones. Pairs with `/retro`. |

## Prerequisites

**For all skills:**

- A `task-data` repository cloned locally. This is the directory where task markdown files live. The default path is `~/projects/task-data` (macOS/Linux) or `%USERPROFILE%\projects\task-data` (Windows 11). You can override it with the `TASK_DATA_DIR` environment variable.
- The repository should contain a `tasks/` subdirectory and a `tasks/archive/` subdirectory.

**For `task-dashboard:task-complete` only:**

- The `task-dashboard` repository (the Node web app) must also be cloned locally, because closing a task runs `npm run archive` from that repository. The default path is `~/projects/task-dashboard` (macOS/Linux) or `%USERPROFILE%\projects\task-dashboard` (Windows 11). You can override it with the `TASK_DASHBOARD_DIR` environment variable.

**Sharing this plugin with a colleague means sharing this setup.** Your colleague will need their own clones of both repositories and their own GitHub remote or fork. The skills themselves contain no personal data — but the task files do, so each user should maintain separate `task-data` repositories.

## Configuration

The skills resolve task-data and task-dashboard locations from environment variables, falling back to the default paths if the variables are not set.

**`TASK_DATA_DIR`** — path to your task-data repository.

macOS/Linux (shell profile, e.g. `~/.zshrc` or `~/.bashrc`):
```bash
export TASK_DATA_DIR="/Users/alice/projects/task-data"
```

Windows 11 (PowerShell profile, or System environment variables):
```powershell
$env:TASK_DATA_DIR = "C:\Users\alice\projects\task-data"
```

**`TASK_DASHBOARD_DIR`** — path to your task-dashboard repository (only needed for `task-complete`).

macOS/Linux:
```bash
export TASK_DASHBOARD_DIR="/Users/alice/projects/task-dashboard"
```

Windows 11:
```powershell
$env:TASK_DASHBOARD_DIR = "C:\Users\alice\projects\task-dashboard"
```

If neither variable is set, the skills fall back to:
- macOS/Linux: `~/projects/task-data` and `~/projects/task-dashboard`
- Windows 11: `%USERPROFILE%\projects\task-data` and `%USERPROFILE%\projects\task-dashboard`

## Installation

**From a Git repository (once this repo has a remote):**

```
/plugin marketplace add <owner>/<repo>
/plugin install task-dashboard@<repo>
```

**Local development / testing before pushing:**

```bash
claude --plugin-dir ~/projects/task-dashboard-skills
```

On Windows 11:
```powershell
claude --plugin-dir "$env:USERPROFILE\projects\task-dashboard-skills"
```

Confirm the six skills appear with their namespaced names:

```
/task-dashboard:task-create
/task-dashboard:task-update
/task-dashboard:task-complete
/task-dashboard:task-list
/task-dashboard:task-bind
/task-dashboard:task-capture
```

## Namespacing

If you previously used the personal (non-plugin) versions of these skills, note that all invocation names have changed:

| Before (personal skill) | After (plugin) |
|-------------------------|----------------|
| `/task-create` | `/task-dashboard:task-create` |
| `/task-update` | `/task-dashboard:task-update` |
| `/task-complete` | `/task-dashboard:task-complete` |
| `/task-list` | `/task-dashboard:task-list` |
| `/task-bind` | `/task-dashboard:task-bind` |
| `/task-capture` | `/task-dashboard:task-capture` |

Internal cross-skill references (for example, `task-bind` delegating to `task-update`) also use the namespaced form, so they resolve correctly within the plugin.

## Retiring personal copies (manual step)

After verifying the plugin is installed and the six skills work correctly, remove the personal copies from `~/.claude/skills/`:

```bash
rm -rf ~/.claude/skills/task-create
rm -rf ~/.claude/skills/task-update
rm -rf ~/.claude/skills/task-complete
rm -rf ~/.claude/skills/task-list
rm -rf ~/.claude/skills/task-bind
rm -rf ~/.claude/skills/task-capture
```

Do this yourself after verification — do not automate it. Keeping both around temporarily is fine while you confirm nothing is missing.

## Windows 11 notes

- Path separators in project encoding use `\` → `-` (and the drive colon `C:` → `C`) rather than `/` → `-`. The skills document both conventions.
- The `npm run archive` command in `task-complete` is the same on both platforms; Node handles the path differences.
- Git for Windows respects the POSIX shebang in `scripts/pre-commit`, so the pre-commit hook installs and runs the same way.
- Set `TASK_DATA_DIR` and `TASK_DASHBOARD_DIR` as user environment variables (System Properties > Environment Variables) rather than in a shell profile, so they are visible to Claude Code regardless of which terminal it is launched from.
