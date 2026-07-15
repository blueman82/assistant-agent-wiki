---
title: "Inbox Brief (PR #23)"
type: source
created: 2026-07-14
last_updated: 2026-07-15
sources: ["tasks/inbox-brief.md", "bridge/notify.ts", "bridge/notify.test.ts", "tasks/inbox-brief-launchd.plist", "prompts/system.md", "CLAUDE.md"]
tags: [gmail, telegram, notify, launchd, scheduled, recommend-only]
---

## What changed

Rachel gained a standing Gmail sweep capability: read `tasks/inbox-brief.md` and follow it to classify recent mail and push a concise brief to Gary's Telegram. Merged as PR #23 (`feature/inbox-brief` → `main`, merge commit `fb6f675`).

New/changed files:
- `tasks/inbox-brief.md` — the sweep prompt (recommend-only)
- `bridge/notify.ts` (+ `bridge/notify.test.ts`) — new standalone proactive Telegram sender for headless one-shot runs
- `tasks/inbox-brief-launchd.plist` — scheduled trigger, 4x/day
- `prompts/system.md`, `CLAUDE.md` — routing/trigger documentation

## Why `notify.ts` exists

A one-shot `bin/rachel "..." < /dev/null` invocation (launchd, or a dashboard button) has no inbound bridge loop in front of it — that loop only lives in `bridge/telegram-bridge.ts`. So the model's ordinary text-reply path has nowhere to go but stdout/a log file. This was a gap PR review caught: an early version of the scheduled sweep silently printed to a log instead of reaching Telegram.

`notify.ts` fixes this by reusing `bridge/api.ts`'s `sendChunked` directly, addressed to whatever chat `loadTelegramConfig()` resolves — Gary's own configured chat, from `gate/surfaces/telegram.ts`.

## Design decisions

**No destination argument.** `notify(filePath)` takes only a file path — there's no way to name a different recipient. This is what keeps it out of the send-gate's threat model: the gate (`gate/sendGate.ts`, see [[capabilities/send-gate]]) guards sends *to others* (Slack channels, Calendar invitees); a notification to the operator's own chat is the same trust class as the bridge's own reply/alert messages and the approval surface's own sends, both already ungated.

**Message read from a FILE, not argv.** Two reasons: (1) argv hits shell-quoting limits on a multi-line brief; (2) more importantly, a swept email whose body happens to contain a string like `api.telegram.org/.../sendMessage` would otherwise land inside the Bash `tool_use` command Rachel issues to invoke the script, tripping `gate/bashPatterns.ts`'s own send-pattern block on Rachel's *own* call to `notify.ts` (see the gate's "Bash defense-in-depth" bullet in [[capabilities/send-gate]]). Reading the text from a file sidesteps both failure modes.

**Testability.** `configFn` is injectable (default `loadTelegramConfig`) — same seam idiom as `rachel.ts`'s `queryFn` param on `runTurn`. `notify.test.ts` stubs the transport, never touches the real network, and includes a grep-guard test asserting no test in the file calls the real `api.telegram.org` endpoint.

## Recommend-only — precise scope (not tool-absence alone)

`tasks/inbox-brief.md` instructs: never send, unsubscribe, delete, or archive without being asked; at most create a draft or apply a label. This is an **instruction constraint**, not a hard tool-absence guarantee: Gary's Gmail MCP connector has no send or hard-delete tool, but it *does* expose `unlabel_thread`/`unlabel_message`, and removing the `INBOX` label is functionally archiving — so the connector's shape does not literally make the rule unbypassable. The task file states this explicitly as a "belt-and-braces" caveat, not a guarantee.

## Delivery contract (headless runs)

`inbox-brief.md`'s step 5: write the assembled plain-text brief to a scratch file, then run `./node_modules/.bin/tsx bridge/notify.ts <path>` via Bash — never a raw `curl`/HTTP call. Success is confirmed by exit code 0 **and** a printed `[notify] sent.` line; either signal missing means delivery failed and must be reported as failure in turn output (visible via the dashboard button's run-output route even without Telegram).

Skip entirely (no write, no send) only when there is nothing new since the last sweep at all. An all-noise sweep still sends a one-line "nothing needs attention" brief — this is what lets Gary trust silence from the schedule as "sweep hasn't run yet," not "sweep ran and found nothing."

## Triggers (per `prompts/system.md`)

1. Gary asking directly ("run the inbox brief" / "check my inbox").
2. `tasks/inbox-brief-launchd.plist` — a **local** launchd job, not a cloud routine (a cloud runtime can't reach the local Gmail MCP connector or the Telegram bridge). Installed and loaded as `com.rachel.inbox-brief` (verified via `launchctl list` at ingest time). Fires 08:00/11:00/14:00/17:00 Ireland time.
3. The coderails dashboard's Inbox Brief button, per `prompts/system.md` — runs the same headless invocation with `cwd=/Users/harrison/Github/assistant-agent`. Documented as stated in `prompts/system.md`; not independently verified live in this ingest pass.

All three converge on the same headless one-shot execution mode: `bin/rachel` with stdin closed, so `rl.question` hits EOF and the REPL exits cleanly on its own — no `KeepAlive` needed, unlike the long-lived bridge process.

## Accepted v1 limitation

No proactive heartbeat or failure alert for the scheduled path specifically. launchd's `StandardOutPath`/`StandardErrorPath` (`.rachel/inbox-brief.log`) captures a log for the schedule-triggered case; the button-triggered case is observable via the dashboard's run route. Neither path pages Gary if a run silently fails to even start.

## Relationships

- [[capabilities/inbox-brief]] — the capability page this source page backs
- [[capabilities/telegram-frontend]] — the bridge's own reply path, distinct from `notify.ts`'s standalone one
- [[capabilities/send-gate]] — why `notify.ts` and its Bash invocation are both deliberately ungated
- [[architecture/overview]] — headless one-shot execution mode alongside the long-lived REPL/bridge
