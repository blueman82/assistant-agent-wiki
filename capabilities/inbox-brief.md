---
title: "Inbox Brief"
type: capability
created: 2026-07-14
last_updated: 2026-07-15
sources: ["tasks/inbox-brief.md", "bridge/notify.ts", "tasks/inbox-brief-launchd.plist", "prompts/system.md", "CLAUDE.md"]
tags: [capability, gmail, telegram, notify, launchd, recommend-only, dashboard-modal, brief-file-sentinel]
---

## What it does

A recommend-only Gmail sweep. Rachel reads recent mail, classifies each thread (needs-action / new / noise), recommends one action per thread (reply / archive / unsubscribe / ignore), and proactively pushes a concise brief to Gary's Telegram — without him asking first. She never sends, unsubscribes, deletes, or archives on her own initiative; at most she may create a draft or apply a label.

## Tool names

`mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__get_thread` for the sweep. Delivery goes through `bridge/notify.ts` (a Bash invocation), not a Gmail or Telegram MCP tool directly.

## How to invoke

- Gary: "run the inbox brief" / "check my inbox" — Rachel reads `tasks/inbox-brief.md` and follows it immediately.
- Scheduled: `tasks/inbox-brief-launchd.plist`, installed as `com.rachel.inbox-brief`, fires 08:00/11:00/14:00/17:00 Ireland time (local launchd job, not a cloud routine — a cloud runtime can't reach the local Gmail MCP connector or Telegram bridge).
- The coderails dashboard's Inbox Brief button (per `prompts/system.md`; not independently verified live at ingest time).

All three run the same headless one-shot: `bin/rachel "Read tasks/inbox-brief.md and follow it." < /dev/null`. Closed stdin means `rl.question` hits EOF and the process exits cleanly on its own once the turn completes — no `KeepAlive` needed.

## Delivery mechanism

A headless one-shot run has no bridge in front of it (`bridge/telegram-bridge.ts`'s inbound loop never starts), so ordinary reply text only reaches stdout/a log. Rachel instead: writes the assembled brief to a scratch file, then runs `./node_modules/.bin/tsx bridge/notify.ts <path>` via Bash. `notify.ts` reuses `bridge/api.ts`'s `sendChunked` — the same sender the Telegram bridge itself uses — addressed only to Gary's own configured chat (no destination argument, so it cannot message anyone else). See [[capabilities/telegram-frontend]] for the bridge's own (distinct) reply path, and [[sources/2026-07-14-inbox-brief]] for the full design rationale.

Success must be confirmed, not assumed: exit code 0 **and** a printed `[notify] sent.` line. Either missing means the brief did not reach Gary, and the sweep must report failure in its own turn output rather than claim success.

## `BRIEF FILE:` sentinel — makes the dashboard modal show the clean brief (PR #27, 2026-07-15)

Once delivery is confirmed (exit 0 + `[notify] sent.`), Rachel's turn must end with a single
final line, exactly: `BRIEF FILE: <absolute path to the scratch file she wrote and sent>` — no
code fence, no bullet, no leading whitespace, so the line is machine-findable by its literal `B`
prefix. On a delivery failure, or the skipped-sweep case above, she must NOT print this line —
state the failure/skip plainly instead, so a stale file is never advertised as current.

This exists for the coderails dashboard's INBOX BRIEF button, not for Gary directly: pressing
that button previously showed the run-output modal Rachel's raw execution trace (tool calls,
reasoning), not the brief itself, because the modal renders the **outer** spawned `claude -p`
agent's `result` field — a separate LLM layer from Rachel, who runs inside it via
`bin/rachel "Read tasks/inbox-brief.md and follow it."`. Editing this file alone could not reach
the modal; the button's own `command` (in coderails' `examples/dashboard-config.json`) had to be
rewritten too, so the outer agent reads the file named on this sentinel line and makes its
**entire final message** exactly that file's verbatim contents. See coderails-wiki's
[[pr_181_183_dashboard-run-output-wrap-and-inbox-brief-clean-modal]] for the full two-repo
mechanism and live verification (run `5cfc5ee0d5a32a82`, modal output byte-matched the sent
brief file, whitespace-normalised).

## Recommend-only — scope of the guarantee

The rule is an instruction constraint, reinforced but not made unbypassable by tool shape. Gary's Gmail MCP connector has no send or hard-delete tool — but it does expose `unlabel_thread`/`unlabel_message`, and removing the `INBOX` label is functionally archiving. So "no send/delete tool" is a partial backstop, not a guarantee; the recommend-only behaviour depends on `tasks/inbox-brief.md`'s instruction being followed.

## Constraints / gotchas

- Skip (no write, no send) only when there's nothing new since the last sweep at all. An all-noise sweep still sends a one-line brief — silence from Rachel should mean "hasn't run yet," never "ran and found nothing."
- Keep well under Telegram's 4096-char single-message limit — trim detail rather than let a brief split across messages.
- No proactive heartbeat/failure alert for the scheduled path (v1, accepted limitation) — launchd logs to `.rachel/inbox-brief.log`; the button path is observable via the dashboard's run route instead.

## Relationships

- [[sources/2026-07-14-inbox-brief]] — PR #23 source page, full design rationale
- [[capabilities/telegram-frontend]] — the bridge's own reply path; `notify.ts` is a separate, standalone outbound path for headless runs
- [[capabilities/send-gate]] — why `notify.ts` and its Bash invocation are deliberately ungated
- [[architecture/overview]] — headless one-shot execution mode
