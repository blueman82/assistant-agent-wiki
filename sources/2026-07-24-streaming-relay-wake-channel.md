---
title: "Spec — Streaming Ticker + Completion→Wake Channel (approved, not built)"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources: ["docs/coderails/specs/spec-streaming-relay-wake-channel.md", "docs/coderails/specs/rca-2026-07-23-fix-list.md", "bridge/telegram-bridge.ts", "bridge/api.ts", "proactive/push.ts", "prompts/system.md", "rachel.ts"]
tags: [spec, approved-not-built, telegram, bridge, streaming, ticker, wake-channel, proactive, decision-record, answer-first]
---

> ⚠️ **NOT BUILT.** This is an approved design, not shipped behaviour. Zero of the three PRs have
> landed and the live drill has not run. Nothing on this page describes what Rachel does today.
> Read [[capabilities/telegram-frontend]] and [[capabilities/proactive-layer]] for current
> behaviour.

## Why this page exists

The spec (`docs/coderails/specs/spec-streaming-relay-wake-channel.md`) lives in a **gitignored**
directory. This wiki page is the only durable home for it — a clean checkout of `assistant-agent`
contains neither this spec nor the RCA it derives from.

Built from items **10, 11 and 17** of the fix list in
[[sources/2026-07-23-rejection-rca-and-fix-list]] (that page carries the canonical numbering).

**It also closes an older task.** `assistant-agent/tasks/2026-07-20-telegram-streaming-responses.md`
(from the 2026-07-20 `raw/task_to_do.md` ingest) asked the same question four days earlier and was
blocked on "Telegram Bot API rate limits for `editMessageText` aren't known yet". That file is now
`status: superseded`, pointing here; both of its open questions are answered below (the rate-limit
design in [[#Ticker behaviour]], and the surface-awareness question made moot by the ticker being
bridge-side and mechanical). One deliberate scope difference is recorded there: that task wanted
streaming that *matches the input mode* (voice-in → voice stream), which Decision A explicitly does
not do — candidate C was the version that would have, and it lost.

## The three problems

1. **Dead air mid-turn.** The bridge buffers *all* of Rachel's output and sends it only when the
   turn ends. Turns run 2–10 minutes; a hard 10-minute deadline aborts them. The raw material
   already exists and is being thrown away — `runTurn` emits live, one line per tool call
   (`kind: "tool"`) and each completed text block (`kind: "text"`); the bridge currently **drops the
   former and buffers the latter** (see the typed emit channel on [[capabilities/telegram-frontend]]).
2. **Answer buried in prose.** Final replies narrate process before conclusion; Gary reads reams to
   find out what happened.
3. **Silence after detached work.** Nothing can start a Rachel turn except an inbound Telegram
   message — `runTurn`'s only caller is `drainFifo`. Detached jobs finish into the void. This is
   finding 2 of the RCA, restated as a design problem.

## Policy constraints (Gary, 2026-07-24)

- **Act then FYI. Never ask-approval.** Reuse `push.ts`'s quiet-hours / dedup / daily-budget for
  anything outbound-proactive ([[capabilities/proactive-layer]]).
- **Telegram-API-friendly**: single chat, respect 429 `retry_after`, jitter + backoff, no
  notification spam.

## Decision A — and why B, C and D lost

Gary delegated the relay choice to a background decision agent (2026-07-24) and adopted its verdict
verbatim: **Decision A — mechanical edit-in-place ticker + answer-first reply rule.**

The rejected alternatives are the durable part of this record — they are what stops the question
being relitigated:

| Candidate | Verdict |
|---|---|
| **A. Mechanical edit-in-place ticker + answer-first rule** | **Chosen.** Zero model calls; fixes both problem 1 and problem 2. |
| B. Ticker alone | **Rejected** — fixes only half the complaint. Progress becomes visible, but the answer is still buried in prose. |
| C. Chunked text streaming | **Rejected** — a notification per chunk; superseded prose litters the chat history; duplicates the voice note on voice-origin turns. |
| D. Model-narrated ticker | **Rejected** — up to **40 one-shot queries per turn** of cost, latency and failure surface, spent restyling lines that are already readable. |

The through-line: the ticker must cost nothing and must not carry reply prose. That constraint is
what kills C and D, and it is why the ticker survives voice-origin turns unchanged.

## Part A — Ticker + answer-first rule

### Ticker behaviour

- **Grace window 3s.** A turn finishing sooner never shows a ticker at all. First render at T+3s:
  `working 0m03s — thinking`.
- **Transport: exactly one `sendMessage` per turn**, sent with `disable_notification: true`
  (message_id captured); every subsequent update is `editMessageText`, which never notifies. **The
  final reply remains the turn's only notifying message.** Both go through `bridge/api.ts`'s
  existing `tg()` generic — `editMessageText` and a silent-send variant must be added there
  (verified 2026-07-24: neither exists today).
- **Cadence:** base 6s with uniform ±2s jitter (4–8s between edits). Each tick renders the **latest
  state only** — events coalesce between ticks, so edit rate is constant regardless of event rate.
  Skip the edit when rendered text is unchanged (Telegram rejects identical edits with "message is
  not modified" — inferred from documented API behaviour, not observed).
- **Content — mechanical, zero model calls:** `working 3m12s — <latest event>`, where the event is
  the most recent `runTurn` emit: a tool line (`[Bash] npm test`) or the first line of the latest
  completed text block, through the existing `stripMarkdown`. Event snippet ≤100 chars, whole line
  ≤200.
- **Terminal edit** immediately before the final send, mirroring the bridge's three existing outcome
  branches: `done — 4m32s` / `failed — 2m10s` / `timed out at 10m`. **The ticker message is never
  deleted.**
- **Voice-origin turns:** ticker runs unchanged — it never carries final prose, so nothing
  duplicates the voice note.

### Failure handling — strictly best-effort

- On **429**: sleep `retry_after` exactly, then **double the cadence** for the remainder of the turn
  (cap 60s).
- Other transport errors: exponential backoff 2s/4s/8s… cap 60s; after **3 consecutive failures,
  freeze** the ticker for the turn.
- **Any exception disables the ticker for that turn and must never block, delay, or fail the turn or
  the final reply.** The ticker is decoration; the reply is the product.

### Caps

1 ticker `sendMessage` per turn; **≤120 edits per turn** (covers the 10-minute deadline at minimum
jitter), then freeze. One ticker at a time, guaranteed by `drainFifo`'s existing single-flight
property.

### The answer-first rule

One edit to `prompts/system.md` — the designated behaviour file
([[patterns/extending-system-md]]): final replies state the answer/outcome in the first sentence;
supporting detail after; no process narration before the conclusion.

**This is an addition, not a swap.** It changes how replies are ordered, removes nothing, interacts
with no code, and can be reverted independently by deleting the one rule.

### Touch points

| File | Change |
|---|---|
| `bridge/api.ts` | add `editMessageText` + silent-send variant via `tg()` |
| `bridge/telegram-bridge.ts` | capture `tool`/`text` emits into per-turn latest-event state; ticker loop beside the existing typing-indicator timer; terminal edit on the three outcome branches |
| `prompts/system.md` | the answer-first rule |
| `rachel.ts` | **no changes** |

## Part B — Completion→wake channel

The answer to RCA finding 2 ("Rachel structurally cannot report back later") and to fix-list item 17.

### Schema — `~/.rachel/wake/<id>.json`

```json
{ "id": "adhoc-tunnel-2026-07-24T09-00", "source": "adhoc:tunnel", "mode": "narrate",
  "severity": "info", "message": "Tunnel task finished: …", "created_at": "…" }
```

- **`mode: "narrate"`** → enqueue a synthetic FIFO message `[wake: <source>] <message>` → a **real
  Rachel turn** (the ticker applies; she reads result artifacts and reports).
- **`mode: "fyi"`** → route through `push.ts` (quiet hours, dedup, daily budget all enforced) as a
  plain push.
- **Untagged rule (Gary, 2026-07-24): missing or invalid `mode` → FYI**, message prefixed
  `[untagged wake: <source>]`. The reasoning is the load-bearing part: **an unknown producer can
  never trigger SDK spend.** Narration costs a model turn; FYI doesn't. Fail toward the cheap path.
- **Malformed JSON** → rename `.bad`, log, continue.
- **Consumed files** → atomic rename to `.done` **before** dispatch. This is deliberately
  at-most-once: a crash between rename and dispatch loses one wake rather than replaying it forever.

### Consumer — the bridge poll loop, not the sweep

The bridge scans `~/.rachel/wake/` once per `getUpdates` iteration: **≤30s latency, zero Telegram
API cost**. Per-iteration cap of **5 files** (flood guard); `narrate` wakes queue behind live
conversation in the FIFO. If the bridge is down, files simply wait — the existing bridge-liveness
sweep family already alerts on that.

**The 30-minute sweep is deliberately NOT the consumer** — its latency would defeat the entire point
("she comes back to you").

### Producers (initial)

1. **Detached ad-hoc jobs** — the final step in the task template writes the wake file. Default
   `mode: narrate`. **The step must be stated in the task file's *body*: frontmatter never reaches
   the spawned process** — the same constraint recorded on
   [[sources/2026-07-22-adhoc-background-escalation]].
2. **Sweep families** — `mode: fyi`. Including the upgraded **stale-process family (fix-list item
   11)**: restart the bridge onto new main (bootout → poll-until-gone → bootstrap, per the launchd
   teardown-race lesson on [[capabilities/installation]]), then wake-FYI "restarted bridge onto
   `<sha>`". No approval asked — act, then tell him.

### Overlap rule — wake file beats watchdog ping

The bridge already pings on LOOP-STOP / 60-min stall via `push.ts`. Without reconciliation a clean
completion would reach Gary **twice** — the watchdog's "it finished" ping *and* the narrated wake
turn.

**Precedence: the agent's wake file is the primary report; the watchdog ping is the crash/stall
fallback.** On loop exit the watchdog checks `~/.rachel/wake/` for a file (pending `.json` or
consumed `.done`) from that slug newer than the loop's start, and skips its own exit ping when one
exists. A job that crashes or stalls *before* writing its wake file still gets the watchdog ping —
that is precisely the case the watchdog exists for. **Stall pings are never suppressed.**

## Out of scope — explicitly

- Chunked reply-text streaming (rejected candidate C).
- Model-narrated ticker (rejected candidate D).
- Exposing thinking blocks to the ticker — `runTurn` does not emit them.

## Testing plan

Repo idiom: vitest with injectable fakes, mirroring `bridge/telegram-bridge.test.ts`.

- **Ticker** (fake clock + fake transport): grace window (no ticker <3s), jitter bounds 4–8s,
  coalescing renders latest event only, skip-identical-edit, placeholder sent with
  `disable_notification`, 429 honours `retry_after` then doubles cadence, exponential backoff +
  freeze-after-3, the 120-edit cap, terminal edit per outcome branch, **ticker exception never
  propagates to the drain loop**, voice turn's ticker carries no reply prose.
- **Wake consumer** (temp dir): tagged narrate→FIFO, tagged fyi→push spy, untagged→push with the
  `[untagged wake:]` prefix, malformed→`.bad`, **rename-before-dispatch ordering**, 5-per-iteration
  cap.
- **Overlap rule**: watchdog exit ping skipped when a slug-matching wake file (pending or `.done`)
  newer than loop start exists; fired when absent; stall ping never suppressed.
- **Wake producers**: task-template text asserted to contain the wake-file step **in the body**.
- **Answer-first rule**: verified by inspection — it's a prompt file, no unit test.

## Rollout — 3 PRs plus a live drill

| # | Contents |
|---|---|
| 1 | `bridge/api.ts` (`editMessageText` + silent send) + ticker in `telegram-bridge.ts` + the `system.md` answer-first rule |
| 2 | Wake channel consumer + schema + untagged rule |
| 3 | Producers — ad-hoc template final step + the item-11 auto-remediate family + the watchdog overlap rule |
| 4 | **Live end-to-end drill** |

**Each PR lands with tests and a bridge restart after merge** — fix-list item 10 applies to this work
too, and the RCA's aggravating factor is exactly what happens when it doesn't.

The drill is one real "background it" from Gary's phone through spawn → completion → wake → narrated
report, with **exactly one** report arriving (wake narration, no duplicate watchdog ping). Note the
starting condition: **the ad-hoc path has never been exercised** — verified 2026-07-24, zero
`tasks/adhoc-*.md` have ever been created, despite the feature shipping in PR #56.

**One friction is documented in advance**: if the triggering turn was aborted, Rachel must ask Gary
to restate the request verbatim before synthesising the task file — the protocol forbids paraphrasing
from memory (`prompts/system.md:161`). Given that turn aborts are exactly what motivates
backgrounding, expect this to fire on the first real attempt.

## Related pages

- [[sources/2026-07-23-rejection-rca-and-fix-list]] — the RCA this derives from; canonical fix-list
  numbering (items 10, 11, 17)
- [[capabilities/telegram-frontend]] — the bridge this modifies: the typed emit channel the ticker
  consumes, the FIFO the wake channel writes into, the 10-minute deadline throughout
- [[capabilities/proactive-layer]] — `push.ts`, the chokepoint every `fyi` wake routes through
- [[sources/2026-07-22-adhoc-background-escalation]] — the detached-spawn path that becomes the
  first wake producer, and the frontmatter-never-reaches-the-spawn constraint
- [[patterns/extending-system-md]] — why the answer-first rule is a prompt edit, not code
- [[capabilities/installation]] — the launchd bootout→poll→bootstrap sequence item 11 must follow
