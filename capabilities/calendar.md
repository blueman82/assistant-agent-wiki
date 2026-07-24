---
title: "Calendar (Google Calendar via MCP)"
type: capability
created: 2026-07-24
last_updated: 2026-07-24
sources: ["prompts/system.md", "rachel.ts", "gate/sendGate.ts", "tasks/proactive-calendar.md", "tasks/proactive-calendar-launchd.plist", "proactive/sweep.ts", "AGENTS.md"]
tags: [capability, calendar, google-calendar, mcp, tool-routing, send-gate, one-shot, conflicts, escalation]
---

## What it does

"Calendar" routes to **Google Calendar** via the `mcp__claude_ai_Google_Calendar__*` MCP tools.
This is the **default and only** wired calendar surface: `prompts/system.md:12` states it as the
tool-routing rule, and `mcp__claude_ai_Google_Calendar__*` is in `DEFAULT_ALLOWED_TOOLS` in
`rachel.ts`.

> **Historical note.** Older `CLAUDE.md` prose describes calendar defaulting to Outlook via
> claude-in-chrome, with Google Calendar as the "personal calendar" opt-in. That is **not** what the
> brain says today — `prompts/system.md`'s routing rules are the operative contract and name Google
> Calendar as the default, no Outlook path. Same divergence as [[capabilities/email]]; see
> [[investigations/2026-07-21-repo-depersonalisation]].

## Tool names

| Purpose | Tool | Gated? |
|---|---|---|
| Read a window | `list_events` | no |
| Read one event | `get_event` | no |
| Search | `search_events` | no |
| List calendars | `list_calendars` | no |
| Find a slot | `suggest_time` | no |
| Create | `create_event` | **yes** |
| Change | `update_event` | **yes** |
| Delete | `delete_event` | **yes** |
| RSVP | `respond_to_event` | **yes** |

## This is where the send gate actually bites

Unlike [[capabilities/email]] (which has no send tool to gate), **four of the five entries in
`GATED_TOOL_NAMES` are Calendar mutators** — verified against `gate/sendGate.ts:32-38`:

```
mcp__claude_ai_Google_Calendar__create_event
mcp__claude_ai_Google_Calendar__update_event
mcp__claude_ai_Google_Calendar__delete_event
mcp__claude_ai_Google_Calendar__respond_to_event
```

(The fifth is `mcp__claude_ai_Slack__slack_send_message`.) Any of these blocks in a `PreToolUse`
hook until Gary approves **that exact request** — approval is bound to a hash of the canonicalised
tool input, is consumed on first use, and cannot be replayed. There is no talking Rachel around it:
even a call made without asking first still blocks and waits. Full mechanics on
[[capabilities/send-gate]].

Reads are ungated, so "what's on my calendar" never prompts.

## The proactive calendar producer

`tasks/proactive-calendar.md` runs as a headless one-shot 4x/day (08/11/14/17) under
`com.rachel.proactive-calendar`. Verified from `tasks/proactive-calendar-launchd.plist`, it is
narrowed to `RACHEL_ALLOWED_TOOLS=Read,Write,Bash,mcp__claude_ai_Google_Calendar__*` — no Chrome, no
WebFetch, no `mcp-exec`.

Each run fetches 48h of events, writes a conflict cache to `$HOME/.rachel/calendar-cache.json`
(**every run, even with zero conflicts** — the cache's freshness is itself a health signal), and
pushes each conflict at severity `normal` under the `[cal]` tag.

**The division of labour with the deterministic sweep matters:**

| Producer | Event-id | Severity |
|---|---|---|
| One-shot (this task) | `cal:<idA>+<idB>` (ids sorted lexicographically) | `normal` only |
| Sweep escalation (`proactive/sweep.ts`) | `cal:<idA>+<idB>:2h` | `urgent` |

State for both is a hash16 of the two events' start+end timestamps, so a rescheduled conflict
re-arms rather than dedup-ing away. **The one-shot must never push calendar urgents itself** — the
30-minute deterministic sweep owns the `<2h` escalation under its own distinct event-id. Two
producers pushing the same urgent under different ids would double-alert.

**Silent-producer detection**: three consecutive sweep ticks with the cache missing or stale beyond
26 hours fire one `normal` `[cal] calendar producer silent` alert. Honest residual, recorded on
[[capabilities/proactive-layer]]: a producer that still writes a cache but pushes *wrongly* is not
detected at all.

## The persistent index

[[calendar-log]] is a wiki page in this vault, not a code artifact — a persistent index of upcoming
events (7–14 days) and recent completions (30 days), auto-refreshed by the same one-shot. It exists
because session memory doesn't survive, so cross-session calendar awareness needs a durable file.
Recurring supplement reminders are filtered out of its tables for readability. See
[[sources/2026-07-20-persistent-calendar-index]].

**It is Rachel's own runtime-maintained file.** Working-tree changes to `calendar-log.md` are
routine auto-maintenance, not ingest work — standing lint precedent is to leave them alone.

## Constraints / gotchas

- **Confirm before acting, not after.** `prompts/system.md:237` — ask upfront; don't proceed
  assuming approval will come later. The gate enforces this on the four mutators regardless.
- **Bash cannot substitute.** A `POST` to the Calendar events endpoint from Bash is denied outright
  by `gate/bashPatterns.ts`. **Known hole** (RCA fix-list item 14): that detector is gated behind a
  `-X POST` / `--request POST` match, so a `curl --data` POST — which is a POST implicitly — evades
  it. See [[sources/2026-07-23-rejection-rca-and-fix-list]].
- **Detached agents bypass all of it.** A spawned `claude -p` never loads the send gate, so ad-hoc
  backgrounding names all four Calendar mutators in its `--disallowedTools` list instead — the only
  enforced restriction there ([[sources/2026-07-22-adhoc-background-escalation]]).
- **All proactive time arithmetic is Intl-in-Dublin**, never UTC arithmetic and never a tz library
  ([[capabilities/proactive-layer]]).

## Relationships

- [[capabilities/send-gate]] — four of its five gated tools live on this surface
- [[capabilities/proactive-layer]] — the `calendar` families, the one-shot/sweep split, the
  silent-producer check
- [[calendar-log]] — the persistent index this surface auto-maintains
- [[sources/2026-07-20-persistent-calendar-index]] — PR #48: why the index exists
- [[capabilities/email]] — the sibling default-routed personal surface, which has *no* gated tool
- [[architecture/mcp-integrations]] — how the Google Calendar connector is wired
