---
title: "Tasks Capability"
type: capability
created: 2026-06-07
last_updated: 2026-07-21
sources: ["prompts/system.md", "tasks/", "tasks/EXAMPLE-task.md", ".gitignore", "AGENTS.md"]
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
- **Only three infrastructure files stay tracked, plus `EXAMPLE-task.md` and `.gitkeep`** (`git ls-files tasks/`, verified 2026-07-21): the three launchd plists (listed in the `TEMPLATES` array at `scripts/install.sh:70-75`, verified 2026-07-21). They are tracked because they are `.plist`, not `.md`, so the `tasks/*.md` ignore rule never matched them — `.gitignore` carries exactly one exception line, `!tasks/EXAMPLE-task.md`, not a per-file exception.
- **Superseded (was: "five files stay tracked").** `inbox-brief.md` and `proactive-calendar.md` were tracked under PR #43's fresh-clone-safety rationale, but commit `a505042` (2026-07-21, "Upgrade agent SDK to 0.3.216; untrack personal task files") untracked both. `.gitignore` now states the consequence plainly: they "are runtime dependencies (read by the launchd jobs and `proactive/sweep.ts`) — they must exist locally on any machine running those jobs, but they are no longer distributed here." **Live gap**: `scripts/install.sh` still installs `com.rachel.inbox-brief` and `com.rachel.proactive-calendar`, whose task files a fresh clone no longer has. Not yet reconciled in code.
- Since PRs #23–#28 (2026-07-14/15) the folder is dual-purpose: it also holds Rachel's standing one-shot prompt files (`inbox-brief.md`, `proactive-calendar.md`) and their launchd plists (`inbox-brief-launchd.plist`, `proactive-calendar-launchd.plist`, `proactive-sweep-launchd.plist`). Those are job instructions for scheduled runs, not Gary-task files in this page's sense — they have no `status`/`due`/`priority` frontmatter and shouldn't be surfaced as tasks. See [[capabilities/inbox-brief]] and [[capabilities/proactive-layer]]
- **This folder is also the single destination for actionable backlog ingested from `raw/`, across all repos — not just `assistant-agent`.** `AGENTS.md`'s Ingest workflow (Step 0, added 2026-07-20, commit `49c478b`) classifies each source file before touching the wiki: reference/knowledge material still becomes wiki pages as usual, but an actionable backlog/TODO list is written here instead, one file per item, using the standard `title/status/due/priority` frontmatter. Items that target a repo other than `assistant-agent` (e.g. `coderails`) get a `Repo: <path>` line in the body rather than a separate per-repo task folder — there is no per-repo task storage, everything lands in this one flat directory. This rule exists because of a same-day incident: a backlog dump was nearly double-handled, once correctly here and once as an abandoned, never-committed write into `assistant-agent-wiki/tasks/` (a page type never part of this wiki's schema — cleaned up in commit `51bbeb9`, that directory is empty and unused).
