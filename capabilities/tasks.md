---
title: "Tasks Capability"
type: capability
created: 2026-06-07
last_updated: 2026-07-15
sources: ["prompts/system.md", "tasks/"]
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

## How to invoke

- "What's on my task list?" → Rachel globs `tasks/` and reads each file
- "Add a task for X" → Rachel writes a new `YYYY-MM-DD-slug.md`
- "Mark X done" → Rachel edits frontmatter `status: done`

## Constraints / gotchas

- Rachel won't surface tasks unprompted — no proactive reminder on session start (as of 2026-06-07)
- No task inbox: tasks must be explicitly requested or dropped as `.md` files manually
- The `tasks/` folder holds a tracked `.gitkeep` plus any live task files (e.g. `launch-model-routing-loop.md`, `salience-cue-experiment-readout.md` as of 2026-07-14). Note: those live files don't follow the documented `YYYY-MM-DD-slug.md` naming — the convention above is the intended format, not an enforced one
- Since PRs #23–#28 (2026-07-14/15) the folder is dual-purpose: it also holds Rachel's standing one-shot prompt files (`inbox-brief.md`, `proactive-calendar.md`) and their launchd plists (`inbox-brief-launchd.plist`, `proactive-calendar-launchd.plist`, `proactive-sweep-launchd.plist`). Those are job instructions for scheduled runs, not Gary-task files in this page's sense — they have no `status`/`due`/`priority` frontmatter and shouldn't be surfaced as tasks. See [[capabilities/inbox-brief]] and [[capabilities/proactive-layer]]
