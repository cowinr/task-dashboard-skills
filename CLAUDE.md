# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

The source for the `task-dashboard` Claude Code plugin: six skills (`task-bind`, `task-create`, `task-update`, `task-list`, `task-complete`, `task-capture`) for managing personal tasks stored as markdown in a separate task-data repository. Installed via the marketplace defined in `.claude-plugin/marketplace.json`.

## Versioning rule (non-negotiable)

Any change that results in a push to GitHub MUST bump the version number in the same change. The plugin updater only fetches when it sees a higher version, so an un-bumped push is invisible to every installed copy.

Bump all three version fields together, keeping them in lockstep:

- `.claude-plugin/plugin.json` → `version`
- `.claude-plugin/marketplace.json` → `metadata.version`
- `.claude-plugin/marketplace.json` → `plugins[0].version`

Use semver: patch for fixes and skill-text tweaks, minor for new skills or new capabilities, major for breaking changes to skill behaviour or the task-file contract. Do the bump as part of the same commit as the change, not a follow-up, so history never shows a content change without a version change.

## Skills

Each skill lives in `skills/<name>/SKILL.md`. Edit the source here, never the installed cache copy under `~/.claude/plugins/cache/` (that is overwritten on update). After pushing a bump, run `/plugin` → update (or reinstall) to pick the change up locally.
