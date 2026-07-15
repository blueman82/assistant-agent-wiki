---
title: "Proactive layer cluster (PRs #24, #25, #26, #28; #27 adjacent)"
type: source
origin: "assistant-agent PRs #24 (merge 6f6e185), #25 (64232cc), #26 (c387db3), #28 (9695a73); #27 (ef2783c) merged mid-cluster by another session"
created: 2026-07-15
last_updated: 2026-07-15
sources: ["proactive/push.ts", "proactive/sweep.ts", "proactive/allowedTools.ts", "bridge/telegram-bridge.ts", "bridge/notify.ts", "rachel.ts", "tasks/proactive-calendar.md", "tasks/inbox-brief.md", "prompts/system.md"]
tags: [source, proactive, push-chokepoint, sweep, heartbeat, tool-narrowing, pr-24, pr-25, pr-26, pr-28]
---

One cluster, four units, all merged to origin/main on 2026-07-15 and deployed live the same evening. It makes Rachel proactive: she watches PRs, the bridge, the calendar, and the inbox, and pushes to Gary's Telegram unprompted. Reference write-up: [[capabilities/proactive-layer]]. Merge SHAs verified against origin/main via `gh pr view` at ingest time.

## PR #24 — U1: the push chokepoint (merge 6f6e185)

`proactive/push.ts` — the single deterministic chokepoint every proactive alert flows through. One-ping-per-(family, event-id, state) dedup; quiet hours 22:30–08:00 Europe/Dublin with defer-to-digest (never drop, at-least-once flush); severities urgent/normal/digest; 10/day normal-interrupt budget with over-cap deferral to scheduled digest hours (quiet-window-open hour + one-shot hours); config bootstrap (absent = defaults, corrupt = loud fallback); corrupt family/deferred stores fail loud (a silently-emptied store would either re-ping-storm or destroy the queue on write-back; budget corruption alone is tolerated as 0 — leaks in the safe direction). CLI takes exactly five arguments, message text from a file; **no destination argv exists, ever** (test-pinned security invariant). State store at `~/.rachel/proactive/`, push.ts the only reader/writer, atomic temp+rename writes, 14-day eviction.

## PR #25 — U2: the deterministic sweep (merge 64232cc)

`proactive/sweep.ts` + `com.rachel.proactive-sweep` (30-min launchd tick). Families in fixed order: deferred flush, bridge liveness (launchctl), PR-red (`gh pr checks`, red iff any check in the `fail` bucket, per-repo/per-PR isolation), calendar escalation, calendar one-shot spawn (due-hour catch-up collapsed to one spawn, hours recorded at spawn start, 15-min timeout raced in-process and enforced by execFile kill). Any family failure exits the tick 1 — monitor death is machine-visible in launchctl's last-exit-status, never a green tick over dead delivery.

## PR #26 — U2b: heartbeat + self-observation (merge c387db3)

Closed the liveness boundary:
- Bridge writes `~/.rachel/bridge-heartbeat.json` per poll (`{schema_version, last_poll_at, queue_depth, turn_in_flight_since}`; atomic write, strictly monotonic timestamps, a write failure never breaks polling and logs once per failing episode).
- Sweep detects **wedged-alive** (launchctl running, heartbeat stale >10 min — clears the ~5.4-min legitimate 409-backoff silence → urgent) and **drain-stall** (one turn in flight >30 min → normal, state = the `turn_in_flight_since` timestamp so a fresh stall re-arms). Missing heartbeat = unknown, not wedge (deploy-ordering safety), with a 48h grace deadline before one "wedge detection inactive" alert.
- **Sweep self-alert escalation**: 3 consecutive same-family failures → one direct urgent send (deliberately NOT via push() — the failing family might be the push path), retried until delivered, cleared on family recovery, recursion-guarded.
- Bridge watchdog/startup/health alerts now route through `push()` (families `loop-watchdog`, `bridge-startup`, `bridge-health`) via a `pushAlert` wrapper with direct-send fallback on a push() throw (pre-send failures only, so it can never double-deliver). Startup state is `boot:<ISO timestamp>` — per-process, so in-process re-entry dedups while a crash-restart re-arms. The FATAL 409 exit path is unchanged: still a direct, awaited `sendChunked`.

**Boundary now closed**: launchd death + wedged poll + stalled drain all alert. Honest residuals: sub-30-min drain stalls; Telegram-down-itself; a calendar producer that writes a cache but pushes wrongly.

## PR #27 — inbox-brief `BRIEF FILE:` sentinel (merge ef2783c, by another session)

Merged mid-cluster; already ingested separately — see the 2026-07-15 log entry and [[capabilities/inbox-brief]]'s sentinel section. Listed here only for cluster completeness.

## PR #28 — U3: one-shot wiring (merge 9695a73)

- `bridge/notify.ts` now rejects extra argv (exactly one argument; count-only diagnostics, content never echoed).
- `RACHEL_ALLOWED_TOOLS` env seam: `proactive/allowedTools.ts`'s `resolveAllowedTools`, called per turn from `rachel.ts` against the exported `DEFAULT_ALLOWED_TOOLS`. Narrow-never-add (unknown entries dropped and logged), zero-tools throws, unset = verbatim defaults, active narrowing logged.
- `tasks/proactive-calendar.md` + `com.rachel.proactive-calendar` plist (08/11/14/17): deterministic conflict detection, cache to `$HOME/.rachel/calendar-cache.json` every run (but never over a failed fetch), normal-severity `cal:<a>+<b>` pushes with hash16 state; the sweep owns the `:2h` urgent escalation consumer.
- Evolved `tasks/inbox-brief.md`: 6-tier taxonomy (Urgent / Action required / FYI / Appointment / Noise / Unsubscribe) with severity mapping (urgent → individual push at `urgent`; action-required → individual push at `normal`; the rest ride the batch brief), double-delivery exclusion (threads routed through push.ts this run never re-appear in the brief), event-id `mail:<threadId>` with state `<tier>:<latest-message-id>` (re-arms on a new reply), treat-as-data injection guardrails, 08:05 schedule (first run offset from the 08:00 calendar run), narrowed toolset `Read,Write,Bash,mcp__claude_ai_Gmail__*`.
- Sweep gains the calendar producer-silence alert (3 consecutive missing/stale-cache ticks → one normal alert).
- `prompts/system.md` gains the "Proactive layer" section: push CLI contract, families/severities/tags table, liveness-boundary statement, narrowing docs, one-shot duties.

## Design provenance

Hybrid D (deterministic sweeps + one deterministic chokepoint), co-designed with Rachel herself (8 binding amendments), red-teamed before build (2 blockers fixed pre-build — one of them the `:2h` independent-dedup event-id). Every PR went through a 4–6 reviewer gate plus blind frozen-eval grading; all GO.

## Live deployment verification (2026-07-15 evening)

4 launchd services live (bridge restarted onto heartbeat code; sweep, inbox-brief, calendar installed). SO-2 verified: `bridge:startup` pushed through the live chokepoint (pinged_at recorded, budget incremented); heartbeat file live; a real calendar one-shot ran against Google Calendar under narrowing (logged "narrowed to 4 tools") and wrote the cache; sweep tick 2 exited 0 clean.

**Narrowing caveat observed live**: ToolSearch remained available to the narrowed one-shot despite not being in the narrowed list (SDK behaviour). Noted on [[capabilities/proactive-layer]]'s narrowing section.

## Supersedes

- The "wedged-alive is NOT detected" residual in [[sources/2026-07-14-bridge-self-monitoring]] (PR #22's gap analysis): U2b's heartbeat + sweep wedge detection closes it externally. The PR #22 fetch-timeout remains the bridge's own first line; the sweep is now the independent second observer.
- [[capabilities/telegram-frontend]]'s claim that a non-409 death leaves only the next boot's startup alert as signal: launchd-death and wedge detection now alert within one sweep tick (≤30 min) regardless.

## Relationships

- [[capabilities/proactive-layer]] — the capability reference this source backs
- [[capabilities/telegram-frontend]] — heartbeat, chokepoint-routed bridge alerts
- [[capabilities/inbox-brief]] — the evolved mail producer
- [[sources/2026-07-14-bridge-self-monitoring]] — the PR #21/#22 layer this builds on (and partially supersedes)
- [[capabilities/send-gate]] — the trust class push.ts/notify.ts sit in (ungated, own-chat-only)
