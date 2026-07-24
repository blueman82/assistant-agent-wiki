---
title: "Wake channel consumer: detached-job completion signalling"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - bridge/telegram-bridge.ts
tags:
  - wake-channel
  - file-drop
  - detached-work
  - completion-signalling
  - latency
---

## Problem

Detached work (background jobs spawned via `claude -p --disallowedTools ...`) has no way to start a Rachel turn today. `runTurn`'s only caller is `drainFifo`, which only wakes on an inbound Telegram message. A background job that finishes has nowhere to report back — it can write to its own task file, but Gary never knows the work completed unless he proactively asks Rachel.

## Solution: file-drop wake channel

A producer writes `~/.rachel/wake/<id>.json`:

```json
{
  "id": "adhoc-tunnel",
  "source": "adhoc:tunnel",
  "mode": "narrate",
  "severity": "normal",
  "message": "Tunnel task finished: 3 files changed.",
  "created_at": "2026-07-24T09:00:00Z"
}
```

`bridge/telegram-bridge.ts`'s new `checkWakeFiles()` scans that directory **once per `getUpdates` poll iteration** — immediately after the existing `checkWatchdogs()` call, whose shape and error posture it mirrors. Deliberately **not** the 30-minute sweep: sweep latency would defeat the whole point of Rachel coming back to you. Cost: zero Telegram API calls, under 30s latency.

## Routing

- **`mode: "narrate"`** — synthetic FIFO message `[wake: <source>] <message>`, then a `void drainFifo()` kick. `drainFifo` is single-flight, so if a turn is already running the kick no-ops — but the message is already in `fifo`, and the running drain loop's `while (fifo.length > 0)` picks it up when the current turn ends. No re-entrancy hazard, no lost message.

- **`mode: "fyi"`** — routed through the existing `pushAlert()` wrapper (family `wake`, event id `wake:<id>`, state `fired`), so it inherits quiet hours, dedup and the daily interrupt budget from `proactive/push.ts` like every other proactive ping.

## SDK-spend guard (security property)

A missing mode — **or any mode value that is not exactly `narrate` or `fyi`** — routes down the FYI path with the message prefixed `[untagged wake: <source>]`. `mode: "explode"` is treated the same as no mode at all.

**This is the security property, not a nicety.** Only an explicitly tagged `narrate` can start a billed Rachel turn. An unknown or buggy producer dropping a file into that directory can reach Gary (via the push chokepoint), but can never trigger SDK spend. Two of the eight unit tests exist purely to pin this (one for missing mode, one for an invalid value).

`severity` is producer-supplied and equally untrusted: anything outside push.ts's own `{urgent, normal, digest}` union — including the spec schema's own example value `"info"` — is coerced to `normal` before it reaches `push()`.

## At-most-once semantics

The consumed file is renamed `<id>.json` to `<id>.done` **before** anything is dispatched, never after. The parsed object is held in memory across the rename, so the dispatch reads from memory rather than from a file that no longer exists.

This deliberately chooses **at-most-once over at-least-once**: a crash in the window between the rename and the dispatch loses exactly one wake, rather than replaying it on every subsequent poll forever. A replay storm on the narrate path would be a billed turn per poll iteration.

**Malformed JSON** is renamed to `<file>.bad`, logged, and skipped — quarantined rather than retried forever, and never allowed to propagate out of the poll loop.

**Flood guard** caps work at `WAKE_ITERATION_CAP = 5` files per iteration; the remainder wait for the next cycle.

## Seam

`wakeDir` on `CreateBridgeOptions`, defaulting to `join(homedir(), ".rachel", "wake")` — an expanded absolute path, never a `~` string (Node's fs does not expand `~`), matching the existing `watchdogDir`/`heartbeatPath`/`pushBaseDir` idiom.

## Tests

8 added, none removed. Real `mkdtempSync` temp dirs rather than the stub fs used elsewhere — the `.done`/`.bad` rename semantics are real filesystem behaviour and a stub would be testing itself.

Covered:
- narrate routes to turn (and side-effects verified)
- fyi delivered with no turn
- missing mode → prefixed fyi with no turn
- invalid mode → prefixed fyi with no turn
- malformed JSON → `.bad` with no crash
- rename-before-dispatch ordering (asserted on the on-disk state read from inside the `runTurn` stub, proving the `.json` is already gone when the turn begins)
- the 5-per-iteration cap
- absent directory as a silent no-op

## Verification

- `npm run typecheck` — clean, exit 0
- `npm test` — 558 pass, 0 fail (baseline 550 + 8 new)
- Frozen PR-2 eval harness — exit 0, `WAKE EVAL PASS`
- Frozen negative control — exit 1 on content (empty wake dir), as required

Diff: exactly `bridge/telegram-bridge.ts` and `bridge/telegram-bridge.test.ts`, 407 insertions, 0 deletions.

## Status

PR 2 of the streaming-relay/wake-channel spec. **This lands inert:** the consumer is ready, but no producer writes wake files yet. PR 3 (not yet merged) will add the first producers — ad-hoc backgrounding task completion and loop-launcher task completion hooks.

## Relationships

- [[sources/2026-07-24-streaming-relay-wake-channel]] — the full spec (approved, design rationale, alternatives considered)
- [[sources/2026-07-23-rejection-rca-and-fix-list]] — RCA finding 2: nothing but inbound Telegram can start a turn (why a wake channel is needed)
- [[capabilities/proactive-layer]] — the `push.ts` chokepoint that fyi-mode wake files route through
- [[capabilities/tasks]] — ad-hoc backgrounding, which will be the first wake-channel producer
