---
title: "Telegram Front-End"
type: capability
created: 2026-07-07
last_updated: 2026-07-18
sources: ["bridge/telegram-bridge.ts", "bridge/api.ts", "bridge/speech.ts", "bridge/launchd.plist", "bridge/notify.ts", "rachel.ts", "gate/surfaces/telegram.ts", "proactive/push.ts", "proactive/sweep.ts", "prompts/system.md", "CLAUDE.md", "AGENTS.md"]
tags: [capability, telegram, bridge, front-end, emit-channel, image-reception, voice, stt, tts, self-monitoring, heartbeat, proactive, character-count]
---

## What it does

A second front-end onto the same Rachel, alongside the terminal REPL. `bridge/telegram-bridge.ts` forwards Gary's Telegram chat messages into the same turn loop (`runTurn`, exported from `rachel.ts`) the terminal uses, and relays replies back as chunked Telegram messages (Telegram caps a single message at 4096 characters). Session continuity, tool access, and agent behaviour are identical to the terminal — only the transport differs.

Single-user: the bridge only accepts messages and approval-button taps from Gary's own configured Telegram chat/user ID (`~/.rachel/telegram.json` or `RACHEL_TELEGRAM_TOKEN`/`RACHEL_TELEGRAM_CHAT_ID`). Anything else is logged and dropped — there is exactly one authorised operator, no multi-user routing.

## Update-routing (the ownership split)

Telegram allows exactly **one** `getUpdates` long-poll consumer per bot token. `bridge/telegram-bridge.ts` owns that single loop — it is the only thing in the repo calling `getUpdates`. The bridge routes what it receives two ways:

- **Ordinary chat messages** → pushed onto a FIFO queue, drained one at a time through `rachel.ts`'s exported `runTurn`.
- **`callback_query` updates** (Approve/Deny button taps) → routed immediately into the imported `telegramSurface` instance from `rachel.ts` via its `handleCallbackQuery` method, ahead of any queued chat turn — a gate decision may already be blocking a turn, so a callback tap must never wait behind the FIFO.

The gate's Telegram approval surface (`gate/surfaces/telegram.ts`, see [[capabilities/send-gate]]) does **not** poll Telegram itself. It only exposes `handleCallbackQuery` and expects the bridge to feed it updates. Both the chat path and the callback path converge on the same `telegramSurface` instance exported from `rachel.ts` — the bridge imports it rather than constructing a second one, so a button tap always resolves the exact surface the send gate is racing against.

## Bridge-level commands

Handled by the bridge before input ever reaches the agent:

| Command | Effect |
|---------|--------|
| `/reset` | Clears the resumed session (same effect as the terminal's `/reset`) |
| `/status` | Replies with uptime, session ID, model, and whether a turn is in flight |
| `/stop` | Aborts the in-flight turn via its `AbortController` |

## Running it

```bash
npm run bridge   # tsx bridge/telegram-bridge.ts
```

For persistence (survives reboots/crashes): run `./scripts/install.sh` from the repo root — it stamps, installs, and verifies this plist along with the other three Rachel services (see [[capabilities/installation]]; the old manual copy-and-`launchctl load` route is superseded). `KeepAlive` restarts the process on any crash or exit; `ThrottleInterval: 60` caps launchd to at most one restart per 60s, so it can never restart faster than Telegram releases its single-consumer `getUpdates` lock (~30-60s). The bridge no longer exits on the first 409 — see [[#Self-monitoring (PR #21 + #22)]] for how a getUpdates conflict is now handled (65s backoff, threshold-5 exit).

## Reply-content contract (typed emit channel)

`rachel.ts`'s `runTurn` emits every line of turn output tagged with a `TurnEmitKind`: `"text"` (the model's own reply text), `"tool"` (a tool-use echo like `  [Read] path/to/file`), or `"meta"` (the `[Rachel] done turns=N cost=$X` completion footer). The type is `TurnEmit = (line: string, kind: TurnEmitKind) => void`.

The terminal REPL prints every kind unchanged — tool echoes and the cost footer stay visible to the operator at the terminal. `bridge/telegram-bridge.ts` buffers **only** `kind === "text"` lines for the Telegram reply, so tool echoes and the turn footer no longer leak to Gary's phone. This means a Telegram reply is composed of exactly: the model's own text blocks, plus two things that bypass the buffer entirely — the bridge's own `"[Rachel] error: ..."` line (pushed directly on a thrown turn, not routed through `emit`) and the `"(no output)"` fallback (fires when a turn yields no text lines at all, so a reply is never silently empty).

The filter is structural — it switches on the typed `kind` field, never on regex/pattern-matching over line content. This was a deliberate fix for the previous behaviour (see [[sources/2026-07-08-telegram-emit-channel]]): before this change, `runTurn` emitted a flat, untyped stream and the bridge forwarded all of it, so tool-use summaries and the done/cost footer leaked into Telegram replies alongside the model's actual answer.

## Reply formatting (plain text, no parse_mode)

Telegram messages are sent with no `parse_mode`, and `prompts/system.md`'s ground rules now say so explicitly: **"Plain text replies"** — replies are read in Telegram and the terminal REPL, not a markdown renderer, so the model should write plain conversational text (no headers, no bold/italic markers, no tables, no code fences unless quoting actual code; simple hyphen bullets are fine).

That prompt rule is the primary fix but not the only one. `bridge/api.ts` exports a deterministic sanitiser, `stripMarkdown(text)`, wired into `sendChunked` and run on the full reply **before** chunking (so a marker straddling the 4096-char boundary can't leak half-stripped):

- Strips leading `#`–`######` header hashes at line start.
- Converts `[text](url)` to `text (url)`.
- Strips `**bold**`, `*italic*`, and `__double-underscore__` markers, keeping their content. Single underscores are never treated as markers by design — only doubled `__pairs__` are — so `snake_case` identifiers and underscore-bearing URLs pass through untouched.
- Strips inline `` `backticks` ``, keeping their content.
- Fenced ` ```code blocks``` ` are split out first and passed through byte-identical — hashes, asterisks, and backticks inside a fence are never touched, since the plain-text rule already permits fences for real code.
- The emphasis regexes use a `(?<!\w)` lookbehind plus non-whitespace content anchors, so spaced maths (`2 * 3`), unspaced arithmetic (`3*4=12`), and `snake_case` all survive without being read as emphasis markers.

**Design decision — no `parse_mode`, ever.** Sending with `parse_mode: HTML` or `MarkdownV2` was considered and rejected: a single malformed entity in the reply causes Telegram to reject the whole `sendMessage` call with an HTTP 400, silently losing the message. Deterministic string stripping can't fail that way — worst case is over-aggressive stripping of a literal asterisk, never a lost reply. Same reasoning ruled out wrapping replies in a structured JSON envelope. Belt-and-braces: the prompt rule is meant to stop markdown at the source; the sanitiser is the backstop for when the model writes it anyway.

## Image reception (PR #17)

Gary can send photos or image documents to Rachel via Telegram. The bridge detects these and forwards them as a synthetic text turn.

**Photo messages**: Telegram sends a `photo` array of multiple compressed variants. The bridge picks the last (largest) entry — `photo[photo.length - 1]` — uses its `file_id`, and derives a `.jpg` extension.

**Image document messages**: Documents with `mime_type` starting with `image/` are accepted. The extension is derived from the document's `file_name` (dot-split) or from the MIME type if no filename is present. Non-image documents (any other MIME) receive an immediate reply: "I can only receive images. Try sending a JPEG, PNG, etc." — `runTurn` is never called.

**Download flow**:
1. `bridge/api.ts`'s `downloadFile(config, fileId, destPath)` calls Telegram's `getFile` API to resolve a download URL.
2. The URL is constructed and consumed entirely inside `downloadFile` — it is never returned or logged (bot token stays internal; `redact()` covers error paths).
3. If Telegram returns no `file_path` (e.g. file >20 MB), a clear error is thrown: "file may exceed the 20 MB API limit".
4. The file is streamed to `~/.rachel/tmp/<fileId>.<ext>` (directory created on first use).
5. `downloadFileFn` is injectable via `CreateBridgeOptions` so tests never touch the filesystem or network.

**FIFO input format**: After download the bridge pushes `[image: /absolute/path/to/file.jpg]` (with caption appended on a new line if present) into the turn FIFO, exactly like a text message.

**Rachel's response contract**: `prompts/system.md` now instructs Rachel:
> When a message begins with `[image: <path>]`, always call the `Read` tool on that absolute path to view the image, then respond based on what you see.

**Known debt**:
- Temp files in `~/.rachel/tmp/` are never cleaned up — the directory grows over time. No cleanup is scheduled.
- No abort signal is threaded into `downloadFile` — a hung Telegram CDN will stall the entire poll loop until Node's default fetch times out.

## Voice in/out — STT+TTS (Tasks 4, 6-9, PR #42)

Gary can send Telegram voice notes to Rachel and get spoken replies back, using local STT/TTS — no cloud speech API. `bridge/speech.ts` shells out to a dedicated Python 3.12 venv (`~/.rachel/venvs/speech`, `scripts/speech/setup.sh`) via `execFile` with a timeout, mirroring `proactive/sweep.ts`'s injectable exec-function pattern: `transcribe()` (mlx-whisper, 30s timeout), `synthesize()` (mlx-audio/Kokoro, 20s timeout), `convertToOgg()` (ffmpeg → Opus/OGG for Telegram, 15s timeout). All three throw on nonzero exit/timeout rather than returning a sentinel; callers decide the fallback.

**Inbound.** `handleMessage`'s `msg.voice` branch (`bridge/telegram-bridge.ts`) downloads the `.ogg` via the existing `downloadFileFn` seam, transcribes it locally, and pushes the transcript onto the FIFO. Download or transcription failure never silently drops the message — each gets its own user-facing text reply ("Failed to download...", "I couldn't transcribe that..."). The downloaded `.ogg` is best-effort unlinked after transcription (success or failure).

**Inbound success log (PR #44, 2026-07-18, merged `f31cacf`).** Until this PR, only inbound *failures* were logged — a successful transcription was silent, so the bridge log had no independent evidence an inbound voice round-trip actually happened; only the outbound `voice reply: N chars` line existed. Surfaced while verifying Task 10 of the voice-STT-TTS plan (the first real end-to-end Telegram check). Fix: a mirrored `console.log("[telegram-bridge] voice received: N chars")` line right after a successful `transcribeFn` call, before the transcript is pushed to the FIFO — same character-count convention as the existing outbound log line. Built TDD (red confirmed without the line, green with it). `npm run typecheck` clean, `npm test` 396/396 (up from 395 baseline).

**FIFO shape.** PR (Task 6) migrated the FIFO from `string[]` to `{ text: string; voice: boolean }[]` — a voice-origin turn is tagged `voice: true` at push time so `drainFifo` knows to speak the reply back. Text-origin and image-origin pushes are tagged `voice: false`.

**Outbound — voice-origin turns always answer in voice, whatever the length.** `drainFifo` (`bridge/telegram-bridge.ts:715-734`) branches on the FIFO entry's `voice` flag: for `voice: true`, it runs the joined reply text through the existing `stripMarkdown` sanitiser (the same one `sendChunked` uses for text replies — reused here so a stray `**bold**`/backtick isn't read aloud literally), then `synthesizeFn` → `convertToOggFn` → `sendVoiceFn` in sequence, writing temp files to `~/.rachel/tmp/reply-<timestamp>.{wav,ogg}` (best-effort unlinked in a `finally`, whichever step failed or succeeded). **Text is the fallback only if that pipeline throws** — a `synthesize`, `convertToOgg`, or `sendVoice` failure alike is caught and logged, and the plain reply text is sent via the ordinary `reply()`/`sendChunked` path instead. It is never a silent, size-based downgrade.

**The 1000-character cap (Task 8) and its removal (PR #42).** Task 8's first implementation of the outbound branch added `VOICE_REPLY_CHAR_LIMIT = 1000`: a voice-origin turn whose stripped reply exceeded 1000 chars silently sent text instead of synthesizing speech, and a test pinned 1000 as if it were a requirement. **The number was never a design decision or a measured constraint — it was invented during implementation, sourced from nothing in the spec or the wiki.** It surfaced as a user-visible bug: voice appeared to "work once, then stop", because a short first reply spoke normally and a longer second reply (e.g. a status report) silently became text with no indication why. Gary's call (PR #42, `d476555`): drop the cap outright. There is now no length branch at all — every voice-origin turn attempts synthesis regardless of reply length; only a thrown error falls back to text.

**Two risks are now live, not theoretical, because of this removal:**
- **No split-into-multiple-voice-messages story.** Text replies have `chunkText`'s 4096-char Telegram boundary (see Chunking boundary below) — voice has no equivalent. A very long reply (e.g. a full inbox brief spoken back) is handed to `synthesizeFn` as one uninterrupted string with no multi-message fallback.
- **`sendVoiceFn` is unchecked against Telegram's own voice-message size ceiling.** If a long synthesized `.ogg` exceeds it, Telegram would presumably reject the `sendVoice` call, which would throw and fall back to text — but this path has never executed. It's an inferred failure mode from the fallback design, not an observed one.

**Real long-reply synthesis is unverified.** The cap made synthesis above 1000 characters unreachable in the running system until PR #42, and the test suite stubs `synthesizeFn`/`convertToOggFn`/`sendVoiceFn` throughout — so no real Kokoro synthesis of a long reply, and no real oversized-`.ogg` `sendVoice` call, has ever actually round-tripped end to end. `npm run typecheck` and `npm test` (391/391) are clean as of PR #42, but that only confirms the stubbed logic paths, not real synthesis behaviour at length. See [[investigations/2026-07-18-telegram-voice-stt-tts-spec-gaps]] (gap 3) for the fuller spec-vs-code review this was resolved against. Gary independently confirmed a real over-1000-char voice reply went out correctly after PR #42 landed (observed live in a Telegram exchange the same day), which is the first real-world signal on this gap — still not a substitute for an explicit end-to-end test.

**Character count — log + caption (commit `6f409be`, merged directly to `main`, no PR).** After the cap's removal, a long voice reply gave Gary no way to gauge reply length from the Telegram UI alone. `drainFifo` now computes `` `${spoken.length} chars` `` from the same `stripMarkdown`-sanitised text handed to `synthesizeFn`, logs it server-side (`console.log("[telegram-bridge] voice reply: N chars")`) immediately before synthesis starts, and passes it as the third (optional) argument to `sendVoiceFn` — which `bridge/api.ts`'s `sendVoice(config, audioPath, caption?)` appends to the multipart form as Telegram's native `caption` field on the voice note, so the count is visible in the Telegram UI right next to the voice bubble, not just in logs. The param is optional and additive: existing `sendVoiceFn` call sites/mocks with the old two-arg shape still typecheck. Built TDD (RED confirmed for both the `sendVoice` caption-forwarding behaviour and the bridge-level log/caption wiring before implementing); `npm run typecheck` and `npm test` (395/395) clean post-merge.

**Known debt (compounds the PR #17 temp-file gap above):** each voice turn now adds an inbound `.ogg` plus outbound `.wav`+`.ogg` intermediates to `~/.rachel/tmp/` — all three are best-effort unlinked in their respective `finally` blocks, but an unlink failure is swallowed silently (no retry, no alert), same as the pre-existing image-reception leak.

## Self-monitoring (PR #21 + #22)

The bridge watches its own health so a failure surfaces on Telegram, instead of only being noticed when Rachel goes quiet. `run()` in `bridge/telegram-bridge.ts` tracks a three-state health machine and alerts Gary on state **transitions only** — never per-error, so an outage can't spam his phone.

**Health states.** `BridgeHealth = "healthy" | "conflict" | "failed"`, starting `healthy`.

- **`conflict`** — entered on a `409 Conflict` from `getUpdates` (a second consumer holds the single-consumer lock). One alert fires on entry. The bridge backs off `CONFLICT_BACKOFF_MS` (65s) and retries. Only after `CONFLICT_EXIT_THRESHOLD` (5) **consecutive** 409s (~5 min) does it `process.exit(1)` for launchd to restart. The FATAL exit alert is **`await`ed** before the exit — a fire-and-forget send would be killed by `process.exit` before the HTTP request completes.
- **`failed`** — entered on any non-409 poll error. One alert on entry. A non-conflict error resets the 409 streak.
- **recovery** — a successful poll from any unhealthy state returns to `healthy` and fires one recovery alert.

**Why 65s / threshold 5, not exit-on-first-409.** The old handler did `process.exit(1)` on the first 409, and launchd's `KeepAlive` restarted in ~1s — faster than Telegram releases the `getUpdates` lock (~30-60s). That produced an infinite self-conflict crash loop, invisible until someone noticed Rachel wasn't responding. 65s clears the lock-release window with margin; threshold-5 distinguishes a launchd restart race (self-resolves) from a genuine second consumer. `launchd.plist`'s `ThrottleInterval: 60` is belt-and-suspenders so a future regression can't recreate the fast-restart loop.

**`/status` health reporting.** `/status` now reports `health: <state>` and, when the bridge has been unhealthy, `last error: <msg> (<ts>, recovered|ongoing)` — so a past or ongoing problem is visible on demand, not just at the moment it happened.

**Fetch timeout (PR #22, gap 1).** `bridge/api.ts`'s `tg()` passes `signal: AbortSignal.timeout(config.requestTimeoutMs ?? 45_000)` to the transport. **Why:** a wedged getUpdates long-poll (network drop without RST, sleep/wake) previously never returned and never threw — so the health machine, which assumes failures throw, never fired and the bridge looked `healthy` while dead. 45s is deliberately longer than the 30s server-side long-poll (`timeout=30`) so a valid slow poll is never aborted, but short enough to catch a genuine hang. The abort surfaces as a thrown error the `failed`-state path handles, turning an invisible hang into an observable alert.

**Startup alert (PR #22, gap 2; delivery reworked by PR #26).** `run()` fires a one-time best-effort startup alert ("Rachel bridge started.") after `setMyCommands`, before the poll loop. **Why:** the FATAL 409 exit was the *only* exit that alerted. Any other death (OOM, reboot, uncaught exception, `launchctl bootout`, the gap-1 timeout crash) sends nothing, because dead processes can't — and the restarted process boots silently `healthy` with `lastError = null`. Since PR #26 this is no longer the only signal a non-409 death leaves — the proactive sweep detects launchd-level death and a wedged poll loop externally within one 30-min tick (see [[capabilities/proactive-layer]]) — but the boot announcement remains the fastest and cheapest one. It is fire-and-forget (`void`, unlike the FATAL-exit alert's `await`) — the process is continuing into the poll loop, not exiting — so a failed startup alert never blocks or crashes boot.

**Chokepoint-routed alerts (PR #26).** The startup alert, the health-transition alerts (entering `conflict`/`failed`, recovery to `healthy`), and the loop-watchdog pings no longer call `sendChunked` directly. They route through `proactive/push.ts`'s `push()` via a `pushAlert` wrapper, under families `bridge-startup` (event `bridge:startup`, state `boot:<ISO boot time>` — per-process, so in-process re-entry dedups while a crash-restart re-arms and announces itself), `bridge-health` (event `bridge:health`, state = the health state entered), and `loop-watchdog`, all severity `normal`. If `push()` itself throws, `pushAlert` falls back to a direct `sendChunked` — and because `push()` treats post-send bookkeeping failures as sent (never rethrows), the fallback only ever fires for **pre-send** failures and can never double-deliver. **The FATAL 409 exit alert is unchanged**: still a direct, `await`ed `sendChunked` — the process is about to exit and must not depend on the chokepoint.

**Heartbeat (PR #26).** The bridge writes `~/.rachel/bridge-heartbeat.json` on every poll iteration: `{schema_version: 1, last_poll_at, queue_depth, turn_in_flight_since}`. Timestamps are strictly monotonic per iteration (distinguishable even under a coarse/frozen clock); the write is temp-file + rename (atomic, same recipe as push.ts's store writes); a write failure must never break polling — it logs once on entering the failing state and re-arms on recovery. This file is what lets the proactive sweep detect a **wedged-alive** bridge (launchctl says running, but `last_poll_at` stale >10 min → urgent alert) and a **drain-stall** (one turn in flight >30 min while the queue starves behind it → normal alert). Path injectable via `CreateBridgeOptions.heartbeatPath`.

**Testability.** `conflictBackoffMs` is injectable via `CreateBridgeOptions` so tests exercise the backoff and threshold-exit paths without real 65s waits.

See [[sources/2026-07-14-bridge-self-monitoring]] for the PR #21/#22 design and rationale, and [[sources/2026-07-15-proactive-layer]] for the PR #26 layer on top.

## External liveness watch (PR #26) — the boundary is closed

Self-monitoring above is the bridge watching itself — it cannot cover its own death. Since PR #26 the proactive sweep ([[capabilities/proactive-layer]], `com.rachel.proactive-sweep`, every 30 min) watches the bridge from outside and alerts through the same chokepoint:

| Failure | Detection | Alert |
|---|---|---|
| launchd-level death | `launchctl print` not running | urgent, within one tick |
| Wedged-alive (process up, poll loop silent) | heartbeat `last_poll_at` stale >10 min | urgent, within one tick |
| Drain-stall (poll fine, one turn >30 min) | heartbeat `turn_in_flight_since` | normal, within one tick |

Recovery is announced only after a recorded down. Honest residuals: drain stalls under 30 min; Telegram itself being down (no alert path can use Telegram then); anything the heartbeat can't express.

## Standalone outbound path for headless runs (`bridge/notify.ts`)

Not part of this bridge — a separate, standalone sender for a different execution mode. A headless one-shot `bin/rachel` invocation (launchd job, or a dashboard button) never starts this bridge's inbound loop, so it has no reply path to Telegram at all: its ordinary text output only reaches stdout/a log. `bridge/notify.ts` covers that gap by reusing `bridge/api.ts`'s `sendChunked` directly — no polling, no FIFO, no `callback_query` handling, just a one-shot send to whatever chat `loadTelegramConfig()` resolves. First consumer: [[capabilities/inbox-brief]]. See [[sources/2026-07-14-inbox-brief]] for the full design (why it reads the message from a file rather than argv, and why it's deliberately ungated — [[capabilities/send-gate]]).

## Constraints / gotchas

- **One consumer only.** A second process polling `getUpdates` on the same bot token causes Telegram to reply `409 Conflict`. The bridge no longer exits on the first 409 — it enters the `conflict` health state, backs off 65s, and retries, exiting (to be restarted by launchd) only after 5 consecutive 409s (~5 min = a genuine second consumer, not a launchd restart race). See [[#Self-monitoring (PR #21 + #22)]]. Never run two bridge instances (or a bridge alongside an ad hoc polling script) against the same token.
- **Log redaction.** `bridge/api.ts`'s `redact()` strips the bot token from every URL before it reaches a log line or thrown error — launchd persists `.rachel/telegram-bridge.log` to disk, so this must run on every logged path, not just the happy path.
- **Chunking boundary.** Replies over 4096 characters are split preferring the last newline at-or-before the boundary (falls back to a hard cut, nudged off a UTF-16 surrogate-pair boundary if one lands there). Markdown stripping runs on the whole reply before this split, not per-chunk.
- **No new runtime deps.** The Telegram client (`bridge/api.ts`) is plain `fetch` — no SDK.

## Relationships

- [[capabilities/send-gate]] — the approval surface whose callback delivery this bridge owns
- [[architecture/overview]] — plumbing/brain split; the bridge is a second thin plumbing layer on top of the same `runTurn`
- [[sources/2026-07-08-telegram-emit-channel]] — the PR that introduced the typed `TurnEmitKind` channel
- [[sources/2026-07-08-telegram-image-reception]] — PR #17: image reception implementation, security decisions, known debt
- [[sources/2026-07-14-bridge-self-monitoring]] — PRs #21 + #22: bridge health state machine, 409 backoff, fetch timeout, startup alert
- [[capabilities/inbox-brief]] — first consumer of the standalone `notify.ts` outbound path
- [[sources/2026-07-14-inbox-brief]] — PR #23: `notify.ts` design and rationale
- [[capabilities/proactive-layer]] — the chokepoint the bridge's own alerts now route through, and the sweep that watches the bridge from outside
- [[sources/2026-07-15-proactive-layer]] — PR #26: heartbeat, chokepoint-routed alerts, closed liveness boundary
- [[investigations/2026-07-18-telegram-voice-stt-tts-spec-gaps]] — spec-vs-code review the voice feature (Tasks 4, 6-9) was built against, and PR #42's resolution of gap 3 (the unsourced 1000-char cap and its removal)
