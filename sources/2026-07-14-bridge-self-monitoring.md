---
title: "Bridge self-monitoring (PRs #21 + #22)"
type: source
origin: "assistant-agent PR #21 (merge 0598e4e) + PR #22 (merge 097d6c3)"
created: 2026-07-14
last_updated: 2026-07-14
sources: ["bridge/telegram-bridge.ts", "bridge/api.ts", "bridge/launchd.plist"]
tags: [source, telegram, bridge, self-monitoring, health, resilience, pr-21, pr-22]
---

Two related PRs, both merged to `main` on 2026-07-14, that close the Telegram bridge's self-monitoring gaps. Together they make a bridge failure surface on Telegram instead of only being noticed when Rachel goes quiet. See [[capabilities/telegram-frontend]] (the "Self-monitoring" section) for the reference write-up.

## PR #21 — health state machine (merge 0598e4e)

Replaced `bridge/telegram-bridge.ts`'s crash-loop-inducing `process.exit(1)`-on-first-409 with a three-state health machine.

- **Three states** — `BridgeHealth = "healthy" | "conflict" | "failed"`, starting `healthy`.
- **409 backoff + threshold exit** — a `409 Conflict` from `getUpdates` enters `conflict`, backs off `CONFLICT_BACKOFF_MS` (65s), and retries. Only after `CONFLICT_EXIT_THRESHOLD` (5) consecutive 409s (~5 min) does it `process.exit(1)` for launchd to restart. The FATAL exit alert is `await`ed before the exit — fire-and-forget would be killed by `process.exit` before the HTTP request completes.
- **`failed` state** — any non-409 poll error. A non-conflict error resets the 409 streak.
- **Transition-only alerts** — one alert entering `conflict`/`failed`, one on recovery. Never per-error, so an outage doesn't spam.
- **`/status` enrichment** — reports `health: <state>` and, when unhealthy, `last error: <msg> (<ts>, recovered|ongoing)`.
- **launchd `ThrottleInterval: 60`** — belt-and-suspenders in `bridge/launchd.plist` so a future regression can't recreate the fast-restart loop.
- **Poll-loop yield fix** — a `setTimeout(pollIntervalMs)` between successful polls keeps `stop()` observable (prevents an immediate-resolve transport from starving the timer queue).
- **Injectable `conflictBackoffMs`** — via `CreateBridgeOptions`, so tests exercise backoff/threshold-exit without real 65s waits.

**Why.** The old handler exited on the first 409, and launchd's `KeepAlive` restarted in ~1s — faster than Telegram releases the single-consumer `getUpdates` lock (~30-60s). That produced an infinite self-conflict crash loop, invisible until someone noticed Rachel wasn't responding. Superseded the partial, untested branch `fix/bridge-409-backoff`. Tests: 113 pass (was 107), 6 new bridge tests.

## PR #22 — fetch timeout + startup alert (merge 097d6c3)

Stacks on #21, closing two observer gaps in its design where the bridge can be dead without alerting. Both changes are additive; nothing retired.

**Gap 1 — a hung getUpdates fetch never throws.** `tg()` called the transport with no timeout. A wedged long-poll (network drop without RST, sleep/wake) never returns and never throws, so #21's health machine — which assumes failures throw — never fires and the bridge looks `healthy` while dead. Fix: `bridge/api.ts`'s `tg()` now passes `signal: AbortSignal.timeout(config.requestTimeoutMs ?? 45_000)`. 45s is longer than the 30s server-side long-poll (`timeout=30`) so a valid slow poll is never aborted, short enough to catch a genuine hang. The abort surfaces as a thrown error the `failed`-state path handles — an invisible hang becomes an observable alert.

**Gap 2 — non-409 deaths can't alert themselves.** The FATAL 409 exit was the only exit that alerted. Any other death (OOM, reboot, uncaught exception, `launchctl bootout`, the gap-1 timeout crash) sends nothing, and the restarted process boots silently `healthy` with `lastError = null`. Fix: a one-time best-effort `sendChunked(config, "Rachel bridge started.").catch(() => {})` in `run()` after `setMyCommands`, before the poll loop. It does NOT `await` (the process continues into the poll loop, not exiting) and its `.catch` means a failed startup alert never blocks or crashes boot. The next boot announcing itself is the only signal a non-409 crash-restart loop leaves.

Tests (TDD): `api.test.ts` proves a 20ms `requestTimeoutMs` makes `tg()` reject rather than hang; `telegram-bridge.test.ts` proves `run()` sends exactly one "started" alert on boot. `npm run typecheck` confirms `AbortSignal.timeout` under the repo's ES2022 target. Full suite: 117 pass / 0 fail.

## Post-merge manual step

The installed plist must be reloaded to pick up `ThrottleInterval`:
`launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.rachel.telegram-bridge.plist && launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.rachel.telegram-bridge.plist`

## Relationships

- [[capabilities/telegram-frontend]] — the capability page whose "Self-monitoring" section this backs
- [[capabilities/send-gate]] — the approval surface the bridge's callback delivery serves
