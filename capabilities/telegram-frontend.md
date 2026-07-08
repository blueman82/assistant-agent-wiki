---
title: "Telegram Front-End"
type: capability
created: 2026-07-07
last_updated: 2026-07-08
sources: ["bridge/telegram-bridge.ts", "bridge/api.ts", "bridge/launchd.plist", "secretary.ts", "gate/surfaces/telegram.ts", "prompts/system.md", "CLAUDE.md", "AGENTS.md"]
tags: [capability, telegram, bridge, front-end]
---

## What it does

A second front-end onto the same secretary, alongside the terminal REPL. `bridge/telegram-bridge.ts` forwards Gary's Telegram chat messages into the same turn loop (`runTurn`, exported from `secretary.ts`) the terminal uses, and relays replies back as chunked Telegram messages (Telegram caps a single message at 4096 characters). Session continuity, tool access, and agent behaviour are identical to the terminal — only the transport differs.

Single-user: the bridge only accepts messages and approval-button taps from Gary's own configured Telegram chat/user ID (`~/.secretary/telegram.json` or `SECRETARY_TELEGRAM_TOKEN`/`SECRETARY_TELEGRAM_CHAT_ID`). Anything else is logged and dropped — there is exactly one authorised operator, no multi-user routing.

## Update-routing (the ownership split)

Telegram allows exactly **one** `getUpdates` long-poll consumer per bot token. `bridge/telegram-bridge.ts` owns that single loop — it is the only thing in the repo calling `getUpdates`. The bridge routes what it receives two ways:

- **Ordinary chat messages** → pushed onto a FIFO queue, drained one at a time through `secretary.ts`'s exported `runTurn`.
- **`callback_query` updates** (Approve/Deny button taps) → routed immediately into the imported `telegramSurface` instance from `secretary.ts` via its `handleCallbackQuery` method, ahead of any queued chat turn — a gate decision may already be blocking a turn, so a callback tap must never wait behind the FIFO.

The gate's Telegram approval surface (`gate/surfaces/telegram.ts`, see [[capabilities/send-gate]]) does **not** poll Telegram itself. It only exposes `handleCallbackQuery` and expects the bridge to feed it updates. Both the chat path and the callback path converge on the same `telegramSurface` instance exported from `secretary.ts` — the bridge imports it rather than constructing a second one, so a button tap always resolves the exact surface the send gate is racing against.

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

For persistence (survives reboots/crashes): copy `bridge/launchd.plist` to `~/Library/LaunchAgents/com.secretary.telegram-bridge.plist`, replace `__REPO_PATH__` with the absolute path to the checkout, then `launchctl load ~/Library/LaunchAgents/com.secretary.telegram-bridge.plist`. `KeepAlive` restarts the process on crash or on the loud `exit(1)` the bridge performs when Telegram reports a second `getUpdates` consumer (HTTP 409).

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

## Constraints / gotchas

- **One consumer only.** A second process polling `getUpdates` on the same bot token causes Telegram to reject both — the bridge detects the resulting 409/conflict, exits loud, and relies on launchd's `KeepAlive` to restart it. Never run two bridge instances (or a bridge alongside an ad hoc polling script) against the same token.
- **Log redaction.** `bridge/api.ts`'s `redact()` strips the bot token from every URL before it reaches a log line or thrown error — launchd persists `.secretary/telegram-bridge.log` to disk, so this must run on every logged path, not just the happy path.
- **Chunking boundary.** Replies over 4096 characters are split preferring the last newline at-or-before the boundary (falls back to a hard cut, nudged off a UTF-16 surrogate-pair boundary if one lands there). Markdown stripping runs on the whole reply before this split, not per-chunk.
- **No new runtime deps.** The Telegram client (`bridge/api.ts`) is plain `fetch` — no SDK.

## Relationships

- [[capabilities/send-gate]] — the approval surface whose callback delivery this bridge owns
- [[architecture/overview]] — plumbing/brain split; the bridge is a second thin plumbing layer on top of the same `runTurn`
