---
title: "PR #60: Bridge log timestamps"
type: source
created: 2026-07-23
last_updated: 2026-07-23
sources: ["bridge/telegram-bridge.ts", "bridge/telegram-bridge.test.ts"]
tags: [source, telegram, bridge, logging, observability, rca]
---

## What changed

PR #60 (merged `15a84e2`, work commit `f778ba2`) added an ISO-8601 timestamp prefix to every log line the Telegram bridge writes about itself. See [[capabilities/telegram-frontend]]'s "Log line timestamps (PR #60)" section for the full implementation detail — this page records the motivating incident and the decision context.

## Why

Same-day, a SIGTERM RCA session hit a wall: `.rachel/telegram-bridge.log` had no timestamps at all, so a `(killed) signal=SIGTERM` line after a voice-synthesis sequence could not be placed in time. Two log-reading mistakes compounded the difficulty before reproduction settled it:

1. **Misread a single multi-line log entry as two separate events.** `execFile`'s error object folds the child's captured stderr into `err.message` as `Command failed: <cmd + args>\n<stderr>` — so a signal-killed subprocess's stderr appears twice in one entry (once raw, once re-embedded), with the full command line (including a long text argument) sandwiched between. Read superficially, this looks like "an exit-1 record, then later a separate SIGTERM record" — it's one execFile error. This is the same trap recorded as a standing lesson: reading a fragment of a multi-line record inverts conclusions.
2. **The SIGTERM was wrongly linked to an adjacent, unrelated `turn exceeded 600000ms` line** — structurally impossible, since the turn watchdog is cleared before synthesis ever runs.

Old-vs-new (was this SIGTERM the already-fixed PR #55 20-second-timeout incident, or a new failure at the new ~127s-scaled budget?) could not be resolved from the log text — there were no timestamps to anchor it. It took reconstructing the actual failing text, reproducing the real synthesis locally with `/usr/bin/time`, and cross-referencing the reply-wav filename's embedded `Date.now()` stamp against the bridge process's actual start time to prove the SIGTERM predated the PR #55 deploy by hours. That reproduction was the right call methodologically, but it was expensive time to spend on something a timestamp would have answered in one glance.

## The decision

Scoped narrowly on purpose (confirmed via `AskUserQuestion`, "bridge lines only" chosen over three broader alternatives — tagging embedded subprocess blobs too, extending to all four launchd services, or not building it at all):
- Only the bridge's own `console.log`/`console.error` emissions get the prefix.
- Embedded subprocess stderr (synthesis/ffmpeg child output folded into thrown errors) stays raw and unprefixed — solving *that* is a different, larger problem (structurally marking multi-line embedded blocks), and conflating the two risked either scope creep or a half-solution that looked more complete than it was.
- The other three launchd one-shot services (inbox-brief, proactive-sweep, proactive-calendar) were explicitly left untouched — their logs are far lower-signal and rarely RCA'd, so the fix doesn't generalize there without separate justification.

## Relationships

- [[capabilities/telegram-frontend]] — the capability page this change lives under
- [[sources/2026-07-22-pr55-voice-synthesis-timeout]] — the earlier incident this RCA was mistaken for, and whose fix (the scaled synthesis timeout) was already correctly deployed the whole time
- [[sources/2026-07-22-adhoc-background-escalation]] — PR #56, the turn-timeout duration logging this PR sits alongside in the same file
