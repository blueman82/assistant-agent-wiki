---
title: "Telegram Front-End"
type: capability
created: 2026-07-07
last_updated: 2026-07-08
sources: ["bridge/telegram-bridge.ts", "bridge/api.ts", "bridge/launchd.plist", "rachel.ts", "gate/surfaces/telegram.ts", "prompts/system.md", "CLAUDE.md", "AGENTS.md"]
tags: [capability, telegram, bridge, front-end, emit-channel, image-reception]
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

For persistence (survives reboots/crashes): copy `bridge/launchd.plist` to `~/Library/LaunchAgents/com.rachel.telegram-bridge.plist`, replace `__REPO_PATH__` with the absolute path to the checkout, then `launchctl load ~/Library/LaunchAgents/com.rachel.telegram-bridge.plist`. `KeepAlive` restarts the process on any crash or exit; `ThrottleInterval: 60` caps launchd to at most one restart per 60s, so it can never restart faster than Telegram releases its single-consumer `getUpdates` lock (~30-60s). The bridge no longer exits on the first 409 — see [[#Self-monitoring (PR #21 + #22)]] for how a getUpdates conflict is now handled (65s backoff, threshold-5 exit).

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

## Self-monitoring (PR #21 + #22)

The bridge watches its own health so a failure surfaces on Telegram, instead of only being noticed when Rachel goes quiet. `run()` in `bridge/telegram-bridge.ts` tracks a three-state health machine and alerts Gary on state **transitions only** — never per-error, so an outage can't spam his phone.

**Health states.** `BridgeHealth = "healthy" | "conflict" | "failed"`, starting `healthy`.

- **`conflict`** — entered on a `409 Conflict` from `getUpdates` (a second consumer holds the single-consumer lock). One alert fires on entry. The bridge backs off `CONFLICT_BACKOFF_MS` (65s) and retries. Only after `CONFLICT_EXIT_THRESHOLD` (5) **consecutive** 409s (~5 min) does it `process.exit(1)` for launchd to restart. The FATAL exit alert is **`await`ed** before the exit — a fire-and-forget send would be killed by `process.exit` before the HTTP request completes.
- **`failed`** — entered on any non-409 poll error. One alert on entry. A non-conflict error resets the 409 streak.
- **recovery** — a successful poll from any unhealthy state returns to `healthy` and fires one recovery alert.

**Why 65s / threshold 5, not exit-on-first-409.** The old handler did `process.exit(1)` on the first 409, and launchd's `KeepAlive` restarted in ~1s — faster than Telegram releases the `getUpdates` lock (~30-60s). That produced an infinite self-conflict crash loop, invisible until someone noticed Rachel wasn't responding. 65s clears the lock-release window with margin; threshold-5 distinguishes a launchd restart race (self-resolves) from a genuine second consumer. `launchd.plist`'s `ThrottleInterval: 60` is belt-and-suspenders so a future regression can't recreate the fast-restart loop.

**`/status` health reporting.** `/status` now reports `health: <state>` and, when the bridge has been unhealthy, `last error: <msg> (<ts>, recovered|ongoing)` — so a past or ongoing problem is visible on demand, not just at the moment it happened.

**Fetch timeout (PR #22, gap 1).** `bridge/api.ts`'s `tg()` passes `signal: AbortSignal.timeout(config.requestTimeoutMs ?? 45_000)` to the transport. **Why:** a wedged getUpdates long-poll (network drop without RST, sleep/wake) previously never returned and never threw — so the health machine, which assumes failures throw, never fired and the bridge looked `healthy` while dead. 45s is deliberately longer than the 30s server-side long-poll (`timeout=30`) so a valid slow poll is never aborted, but short enough to catch a genuine hang. The abort surfaces as a thrown error the `failed`-state path handles, turning an invisible hang into an observable alert.

**Startup alert (PR #22, gap 2).** `run()` fires a one-time best-effort `sendChunked(config, "Rachel bridge started.").catch(() => {})` after `setMyCommands`, before the poll loop. **Why:** the FATAL 409 exit was the *only* exit that alerted. Any other death (OOM, reboot, uncaught exception, `launchctl bootout`, the gap-1 timeout crash) sends nothing, because dead processes can't — and the restarted process boots silently `healthy` with `lastError = null`. The next boot announcing itself is the only signal a non-409 crash-restart loop leaves. It does **not** `await` (unlike the FATAL-exit alert) — the process is continuing into the poll loop, not exiting — and its `.catch(() => {})` means a failed startup alert never blocks or crashes boot.

**Testability.** `conflictBackoffMs` is injectable via `CreateBridgeOptions` so tests exercise the backoff and threshold-exit paths without real 65s waits.

See [[sources/2026-07-14-bridge-self-monitoring]] for the full design and rationale.

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
