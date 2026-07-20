---
title: "Tasks Capability"
type: capability
created: 2026-06-07
last_updated: 2026-07-18
sources: ["prompts/system.md", "tasks/", "tasks/EXAMPLE-task.md", ".gitignore"]
tags: [capability, tasks]
---

## What it does

Manages Gary's task list as flat markdown files. Rachel reads, creates, and updates task files on request. It does not proactively surface tasks at session start — it's reactive.

## Tool names

`Read`, `Write`, `Edit`, `Glob` — standard file tools, no MCP needed.

## File format

- **Location**: `/Users/harrison/Github/assistant-agent/tasks/`
- **Naming**: `YYYY-MM-DD-slug.md`
- **Frontmatter**: `title`, `status`, `due`, `priority`

Example:
```markdown
---
title: "Thing to do"
status: pending
due: 2026-06-10T09:00:00+01:00
priority: high
---

Body text here.
```

**Two frontmatter shapes.** The above covers Gary's own tasks. Loop-launcher task files (`tasks/launch-*.md` — see the Loop launcher section of `prompts/system.md`) use a separate shape instead: `title`, `slug`, `repo`, `permission_mode`, `status` (`launchable`/`launched`/`done`) — no `due`/`priority`. `tasks/EXAMPLE-task.md` (added PR #43, 2026-07-18, `chore/untrack-personal-tasks`) documents both shapes side by side and is the only genuine-task file left tracked in git.

## How to invoke

- "What's on my task list?" → Rachel globs `tasks/` and reads each file
- "Add a task for X" → Rachel writes a new `YYYY-MM-DD-slug.md`
- "Mark X done" → Rachel edits frontmatter `status: done`

## Constraints / gotchas

- Rachel won't surface tasks unprompted — no proactive reminder on session start (as of 2026-06-07)
- No task inbox: tasks must be explicitly requested or dropped as `.md` files manually
- **`tasks/*.md` is gitignored as of PR #43 (2026-07-18, `chore/untrack-personal-tasks`, merged `e27b7cf`).** Personal task files (e.g. `launch-model-routing-loop.md`, `salience-cue-experiment-readout.md`) stay on disk but are no longer tracked in the shared repo — a fresh clone won't have them. Only `tasks/EXAMPLE-task.md` is tracked, specifically to document the format for both frontmatter shapes (see above). Those untracked live files still don't follow the documented `YYYY-MM-DD-slug.md` naming.
- Five files stay tracked despite being under `tasks/*.md` in spirit, because untracking them would break a fresh clone: the three launchd plists (read by `scripts/install.sh:72-74`), `inbox-brief.md` (read by `com.rachel.inbox-brief`, the dashboard button, and `prompts/system.md`), and `proactive-calendar.md` (read by `proactive/sweep.ts` and asserted in `proactive/sweep.test.ts`). `.gitignore` carries an explicit exception per file rather than untracking the whole directory.
- Since PRs #23–#28 (2026-07-14/15) the folder is dual-purpose: it also holds Rachel's standing one-shot prompt files (`inbox-brief.md`, `proactive-calendar.md`) and their launchd plists (`inbox-brief-launchd.plist`, `proactive-calendar-launchd.plist`, `proactive-sweep-launchd.plist`). Those are job instructions for scheduled runs, not Gary-task files in this page's sense — they have no `status`/`due`/`priority` frontmatter and shouldn't be surfaced as tasks. See [[capabilities/inbox-brief]] and [[capabilities/proactive-layer]]
