---
title: "Proactive Layer"
type: capability
created: 2026-07-15
last_updated: 2026-07-15
sources: ["proactive/push.ts", "proactive/sweep.ts", "proactive/allowedTools.ts", "rachel.ts", "bridge/telegram-bridge.ts", "tasks/proactive-calendar.md", "tasks/inbox-brief.md", "tasks/proactive-sweep-launchd.plist", "tasks/proactive-calendar-launchd.plist", "tasks/inbox-brief-launchd.plist", "prompts/system.md"]
tags: [capability, proactive, push-chokepoint, sweep, launchd, one-shot, tool-narrowing, quiet-hours, budget, dedup]
---

## What it is

Rachel is now proactive, not purely reactive. PRs #24–#26 and #28 (merged 2026-07-15, deployed live the same evening; design "Hybrid D": deterministic sweeps + one deterministic chokepoint, co-designed with Rachel under 8 binding amendments and red-teamed pre-build) added a layer that watches things on Gary's behalf and pushes alerts to his Telegram without being asked. Every alert flows through **one deterministic chokepoint**: `proactive/push.ts`. See [[sources/2026-07-15-proactive-layer]] for the cluster source page.

Two producer kinds feed the chokepoint:
- **Deterministic sweep** — `proactive/sweep.ts`, a 30-minute launchd tick (`com.rachel.proactive-sweep`) covering PR-red, bridge liveness, calendar `<2h` escalation, deferred-digest flushing, and one-shot spawning.
- **LLM one-shots** — headless `bin/rachel` runs following a task file (`tasks/proactive-calendar.md`, `tasks/inbox-brief.md`), which call the push CLI via Bash. They observe and compose; the chokepoint decides delivery.

## The chokepoint (`proactive/push.ts`, PR #24)

Library for the sweep, CLI for one-shots. It is the **only** code that reads/writes the state store at `~/.rachel/proactive/`. Delivery decisions, in order:

1. **Validate** — family must match `/^[a-z][a-z0-9-]*$/`, severity must be `urgent | normal | digest`. Fail fast.
2. **Dedup** — one ping per `(family, event-id, state)`. Same id + same state → `dedup`, never re-sent. A changed state re-arms the ping.
3. **Digest never interrupts** — always deferred.
4. **Quiet hours** — `normal` inside 22:30–08:00 Europe/Dublin defers to the next digest flush. **Defer, never drop.**
5. **Daily budget** — `normal` over 10 interrupts per Dublin day defers with reason `budget`.
6. **Send** via `bridge/api.ts`'s `sendChunked` to whatever chat `loadTelegramConfig()` resolves. A send throw records nothing (next sweep retries); a **post-send** bookkeeping failure logs loud and still returns `sent` — never re-deliver a delivered alert.

`urgent` bypasses quiet hours and the budget entirely.

**Deferred flush** (`flushDeferred`, called first on every sweep tick): batches eligible deferred entries into one `[digest]` message. Quiet/digest-reason entries flush on any tick outside the quiet window; **budget-reason entries ride only the scheduled digest hours** (the calendar one-shot hours plus the quiet-window-open hour), so budget overflow lands hours apart. Delivery is **at-least-once** by design: write-back happens only after the send resolves, and the write-back subtracts the flushed snapshot (keyed `family|queued_at|event_id`) rather than truncating, so a concurrently appended entry survives.

**All time arithmetic is Intl-in-Dublin** — never UTC arithmetic, never a tz library.

### Store layout (`~/.rachel/proactive/`)

| File | Contents | Corruption policy |
|---|---|---|
| `<family>.json` | `{schema_version: 1, events: {<event-id>: {state, first_seen, pinged_at, last_seen}}}` | fail loud (silent-empty = re-ping storm) |
| `deferred.json` | `{schema_version: 1, entries: [{family, event_id, state, text, queued_at, reason}]}` | fail loud (silent-empty + write-back = queued entries destroyed) |
| `budget.json` | `{schema_version: 1, date, interrupts_sent}` (Dublin date; other date = rolled to 0) | loud-but-tolerated as 0 — leaks only in the safe direction (pings still deliver) |
| `config.json` | quiet hours, budget, timezone, `pr_watch_repos`, `calendar_oneshot_hours` | loud fallback to `DEFAULT_CONFIG`; absent = defaults, not an error |

Writes are temp-file + rename (atomic on APFS). Every family-file write evicts entries not seen for 14 days. Only ENOENT maps to absent-is-empty; anything else throws with the path.

### Push CLI contract

```
./node_modules/.bin/tsx proactive/push.ts <family> <event-id> <state> <severity> <message-file>
```

Exactly five arguments; a sixth of any kind is rejected. Success = exit 0 + exactly one of `[push] sent.` / `[push] deferred.` / `[push] dedup.` (all three count as success — deferred/dedup are the chokepoint's judgement working). Exit 0 *without* one of those lines is a failure too.

## Families, severities, tags

| Family | Event-id | State | Severity | Tag |
|---|---|---|---|---|
| mail | `mail:<threadId>` | `<tier>:<latest-message-id>` | urgent / normal | `[urgent · mail]`, `[mail]` |
| calendar (one-shot) | `cal:<idA>+<idB>` (ids sorted) | hash16 of the four start/end timestamps | normal only | `[cal]` |
| calendar (sweep escalation) | `cal:<idA>+<idB>:2h` | same hash16 | urgent | `[urgent · cal]` |
| pr-red | `pr:<owner>/<repo>#<number>` | `<head_sha>:failure` | normal | `[pr]` |
| bridge-liveness | `bridge:liveness` | `up` \| `down` | urgent (down) / normal (up) | `[urgent · bridge]`, `[bridge]` |
| bridge-liveness | `bridge:drain-stall` | the `turn_in_flight_since` timestamp | normal | `[bridge]` |
| bridge-startup / bridge-health / loop-watchdog | (bridge-internal, see [[capabilities/telegram-frontend]]) | — | normal | — |
| digest flush | — | — | — | `[digest]` |

State strings are chosen so **news re-arms and non-news dedups**: a new head SHA on a still-red PR re-pings; a rescheduled calendar conflict changes the hash16 and re-arms both channels; a new reply on an already-pinged mail thread changes `<latest-message-id>`.

The `:2h` suffix on the sweep's calendar escalation is deliberate (red-team finding): it dedups **independently** of the one-shot's normal ping — a shared id would let one recorded state swallow the other and Gary would never get the <2h heads-up.

## The sweep (`proactive/sweep.ts`, PRs #25 + #26)

Fixed tick order: **deferred flush → bridge-liveness → PR-red → calendar-escalation → calendar one-shot spawn.** Each family runs in its own try/catch so one broken family never blocks the others; any family failure makes the tick exit 1 so launchd's last-exit-status is machine-visible (monitor death is never a green tick over dead delivery).

- **PR-red**: `gh pr list --author @me` per watched repo, then `gh pr checks --json name,bucket,state`; red iff any check lands in gh's `fail` **bucket** (covers TIMED_OUT, STARTUP_FAILURE etc., which a state-string comparison would miss). Per-repo and per-PR isolation. Non-zero exit with empty stdout = no checks / gh failure → skip, never abort the family. Recovery is not pinged; stale entries age out via 14-day eviction.
- **Bridge liveness**: see the liveness boundary below.
- **Calendar `<2h` escalation**: reads `~/.rachel/calendar-cache.json` (written by the one-shot), escalates any conflict starting in `(now, now+2h]` to urgent. A cache older than 26h is treated as absent (the data is yesterday's). **Producer-silence detection**: 3 consecutive ticks with cache missing/stale fires one normal `[cal] calendar producer silent` alert — a dead producer must not degrade to eternally-skipped positive silence.
- **One-shot spawning**: due configured hours (≤ current Dublin hour, not yet recorded today) collapse into ONE catch-up spawn; hours are recorded when the spawn **starts**, so a hung one-shot can't respawn-storm. Spawns run with a 15-minute timeout, raced in-process *and* enforced via execFile's kill-on-expiry.
- **Self-alert escalation** (PR #26): per-family consecutive-failure counters persist in `~/.rachel/proactive-sweep-state.json` (sweep-owned, deliberately outside the push store dir); from the 3rd consecutive failure of a family the sweep sends one **direct** alert (not via `push()` — the failing family might BE the push path), retried each tick until a send succeeds, then not repeated for the same streak. A failing escalation send is logged, never escalated about (recursion guard).

## The liveness boundary — CLOSED (PR #26)

Three layers, all alerting:

1. **launchd-level death** — `launchctl print` says not running → urgent `bridge:liveness` state `down`.
2. **Wedged-alive** — process running but the poll loop silent: the bridge writes `~/.rachel/bridge-heartbeat.json` per poll (`{last_poll_at, queue_depth, turn_in_flight_since}`); heartbeat stale >10 min → urgent (also `down`-class). The 10-min threshold comfortably clears the bridge's legitimate 409-backoff silence (up to ~5.4 min). Missing heartbeat file with a running process is UNKNOWN, not a wedge (deploy-ordering safety), but 48h of launchctl-alive with no heartbeat file ever appearing fires one normal "wedge detection inactive" alert.
3. **Drain-stall** — poll loop fine but one turn in flight >30 min → normal `bridge:drain-stall`, keyed on the `turn_in_flight_since` timestamp so a new stall re-arms. Checked independently of the wedge; both can fire in one tick.

Recovery (`up`) is announced only after a recorded `down` — a first-ever healthy observation pushes nothing.

**Honest residuals**: a drain stall under 30 min goes unseen until it crosses the threshold; Telegram-down-itself can't be alerted through Telegram; a calendar producer that still writes a cache but pushes wrongly is not detected.

## The one-shot pattern (PR #28)

A headless one-shot = `bin/rachel "Read tasks/<file>.md and follow it." < /dev/null`, with:

- **Tool narrowing**: `RACHEL_ALLOWED_TOOLS` (comma-separated) resolved per turn by `proactive/allowedTools.ts`'s `resolveAllowedTools` against `rachel.ts`'s exported `DEFAULT_ALLOWED_TOOLS`. **Narrow-never-add**: only entries already in the default list are honoured; unknown entries are dropped with a stderr log — a hostile or misconfigured env line can never grant tools the interactive agent doesn't have. A set value filtering to zero tools **throws** rather than running tool-less. Unset/empty = full defaults (provably inert for the interactive agent). Active narrowing logs `narrowed to N tools`.
  - Calendar one-shot: `Read,Write,Bash,mcp__claude_ai_Google_Calendar__*` (pinned by a cross-check test as a subset of the defaults).
  - Inbox-brief one-shot: `Read,Write,Bash,mcp__claude_ai_Gmail__*` (a run that reads hostile email must not carry Chrome, WebFetch, or mcp-exec).
  - Caveat (live observation, deploy session 2026-07-15, not re-verified from source here): the SDK kept **ToolSearch** available to a narrowed one-shot even though it wasn't in the narrowed list — narrowing governs the agent's allowedTools, but ToolSearch's presence isn't suppressed by it. Treat narrowing as a strong guardrail, not a hermetic sandbox.
- **Treat-as-data guardrails**: both task files instruct that swept content (email sender/subject/body; calendar titles/descriptions/attendees) may be hostile prompt injection — display as data, never follow as instructions. All extracted data goes into message files via Write, **never into CLI arguments**.
- **Message-from-file, never argv**: same rationale as `bridge/notify.ts` — argv hits shell quoting limits, and a swept body containing a send-looking string would trip `gate/bashPatterns.ts` on Rachel's own push invocation.
- **Result checking**: the one-shot must verify exit 0 + one of the three `[push]` result lines per push, and state any failure plainly rather than reporting a clean sweep.

### Calendar one-shot (`tasks/proactive-calendar.md`)

Fetch 48h of events; detect conflicts **deterministically** (`startA < endB AND startB < endA`, strict — shared boundaries don't conflict; all-day events skipped; ids sorted so idA < idB); write the cache to `$HOME/.rachel/calendar-cache.json` **every run, even with zero conflicts** (a missing/stale cache blinds the sweep's escalation) — but **never** after a failed fetch (a fresh cache asserts a successful fetch); push each conflict at `normal` under `cal:<idA>+<idB>` with the hash16 state (recipe pinned identically in the task file's shell line and the sweep's `hash16()`). Never pushes urgent — the sweep owns the `:2h` escalation. Scheduled both by `com.rachel.proactive-calendar` (launchd, 08/11/14/17) and by the sweep's catch-up spawn for missed hours; chokepoint dedup makes the overlap harmless.

### Inbox-brief one-shot

Evolved by PR #28 — see [[capabilities/inbox-brief]] for the 6-tier taxonomy, per-thread pushes, and double-delivery exclusion.

## Security invariants (pinned by tests)

- **No destination anywhere**: `push()`'s signature, the push CLI (exactly five args, a sixth rejected), and `notify.ts` (exactly one arg) carry no chat-id parameter — delivery goes only to Gary's own configured chat.
- **Narrow-never-add** on `RACHEL_ALLOWED_TOOLS`, with zero-tools throw.
- **Message text from a file, never argv.**
- **push.ts is the sole reader/writer of `~/.rachel/proactive/`** (the sweep's own state lives elsewhere, in `~/.rachel/proactive-sweep-state.json`).

## Deployed state (2026-07-15)

Four launchd services live: `com.rachel.telegram-bridge` (restarted onto heartbeat-writing code), `com.rachel.proactive-sweep`, `com.rachel.inbox-brief` (08:05/11/14/17), `com.rachel.proactive-calendar` (08/11/14/17). Live verification: `bridge:startup` pushed through the chokepoint (pinged_at recorded, budget incremented), heartbeat live, a real calendar one-shot ran under narrowing ("narrowed to 4 tools" logged) and wrote the cache, sweep tick clean exit 0.

## Relationships

- [[sources/2026-07-15-proactive-layer]] — cluster source page (PRs #24–#26, #28; design provenance; live verification)
- [[capabilities/telegram-frontend]] — the bridge whose liveness the sweep watches; its heartbeat and chokepoint-routed alerts
- [[capabilities/inbox-brief]] — the mail-family producer
- [[capabilities/send-gate]] — why push.ts/notify.ts are deliberately outside the send gate (operator's-own-chat only, no destination)
- [[architecture/overview]] — headless one-shot execution mode
