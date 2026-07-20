---
title: "Coderails: loop scheduling to nightly"
slug: loop-scheduling-nightly
status: backlog
priority: medium
repo: "[coderails]"
due: null
---

## Overview

Change loop schedules from current cadence to nightly execution. Identify all scheduled loops and update their timing.

## Scope

- Locate all loop schedule definitions (launchd plists, RemoteTrigger routines, cron configs)
- Current schedules: identify and document
- Target: nightly (e.g., 00:00 or 02:00 Europe/Dublin)
- Affected loops: inbox-brief, proactive-calendar, loop-retro, comments-vs-code, etc.

## Acceptance Criteria

- All loop schedules identified (source of truth: launchd/cron/RemoteTrigger)
- Updated to nightly (00:00 or specified time in Dublin timezone)
- Tested: at least one loop executes at new time
- Documentation updated

## Dependencies

- None (but coordinates with task 6: comments-vs-code nightly routine)

## Notes

Original task (line 17 of tasks_to_do.md): "change loop schedules to nightly"
