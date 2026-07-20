---
title: Persistent Calendar Index
type: source
created: 2026-07-20
last_updated: 2026-07-20
sources: [tasks/proactive-calendar.md]
tags: [calendar, persistence, cross-session-memory, proactive-layer]
---

## Summary

Added persistent calendar logging to the proactive-calendar task: upcoming events (7–14 days ahead) and recent events (past 30 days) are now written to a wiki page (`calendar-log.md`) and refreshed 4x/day. Solves the gap between Gary's calendar and my (Rachel's) cross-session awareness.

## Problem

Gary's calendar events were invisible to me across sessions. When he mentioned "I cancelled the bike insurance" (logged in calendar), I had no persistent memory of it on the next session. I could query the calendar on-demand via MCP, but that doesn't help me stay contextually aware of his commitments without asking.

## Solution

**Step 4 in `tasks/proactive-calendar.md`** now:
1. Fetches 30 days of calendar data (supports both the 7–14d window and 30d lookback)
2. Extracts two time windows: upcoming (7–14 days) and recent (past 30 days)
3. Writes two markdown tables to `/Users/harrison/Github/assistant-agent-wiki/calendar-log.md`
4. Runs 4x/day as part of the proactive calendar task (same mechanism that detects conflicts)

**Wiki page structure** (`calendar-log.md`):
- Two markdown tables (Date | Event | Time)
- No Status column (Google Calendar doesn't support completion tracking)
- Auto-refreshed — Gary never edits it
- Browseable in Obsidian alongside other context

## Key Decisions

| Decision | Why |
|----------|-----|
| Fetch 30 days (not 48h) | Supports both upcoming and historical windows |
| Wiki page (not task file) | Single source of truth in Obsidian; browseable; persistent |
| Two rolling tables (not log) | Recent past + near future captures most-relevant context |
| Auto-refresh (not manual) | Zero maintenance burden on Gary |
| Date\|Event\|Time only | Simple schema; Google Calendar has no completion field |

## Impact

- Rachel now maintains cross-session awareness of Gary's calendar commitments
- Enables proactive context ("I see you have a meeting at 2pm") without asking
- Foundation for future calendar-aware features (conflict detection, availability checks)
- Demonstrates persistent wiki indexing pattern for other sources (email, tasks, etc.)

## Related Pages

- [[capabilities/calendar]] — the underlying Google Calendar MCP integration
- [[capabilities/proactive-layer]] — the 4x/day refresh mechanism
- [[calendar-log]] — the index page itself

## PR Details

**PR #48**: Add persistent calendar-log wiki index
- Modified: `tasks/proactive-calendar.md` (added step 4, renumbered steps 4-6 to 5-7)
- Created: `calendar-log.md` (skeleton with two empty tables)
- Updated: `index.md` (cross-reference to calendar-log)
- Merged: 2026-07-20 (commit cfc2e61)

**Fixes during review**:
- Expanded fetch window from 48 hours to 30 days (supports both upcoming and historical)
- Corrected wiki path to absolute: `/Users/harrison/Github/assistant-agent-wiki/calendar-log.md`
- Removed undefined "Status" column (Google Calendar has no completion field)
