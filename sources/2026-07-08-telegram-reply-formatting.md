---
title: "Telegram Reply Formatting Fix (PR #14)"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: ["bridge/api.ts", "bridge/api.test.ts", "prompts/system.md", "CLAUDE.md"]
tags: [source, telegram, bridge, markdown]
---

## What changed

PR #14 (`fix/telegram-reply-formatting`, merge commit `4a70688`, merged 2026-07-08) fixed replies showing literal markdown (`**bold**`, `## headers`, backticks) on Gary's phone. Root cause: the bridge sends Telegram messages with no `parse_mode` while the model wrote markdown-formatted text, so Telegram rendered it as plain literal characters.

Two changes, layered:

1. **`prompts/system.md`** gained a ground rule, "Plain text replies": replies are read in Telegram and the terminal REPL, not a markdown renderer, so write plain conversational text — no headers, no bold/italic, no tables, no code fences unless quoting real code, hyphen bullets fine. This is the primary fix.
2. **`bridge/api.ts`** gained an exported `stripMarkdown(text)` sanitiser, wired into `sendChunked` and run on the full reply before chunking. Strips header hashes, `**bold**`/`*italic*`/`__underscore__` markers, inline backticks, and converts `[text](url)` to `text (url)`. Fenced code blocks are split out and passed through byte-identical. Emphasis regexes use a `(?<!\w)` lookbehind plus non-whitespace content anchors so spaced maths (`2 * 3`), unspaced arithmetic (`3*4=12`), `snake_case`, and underscore-bearing URLs all survive — single underscores are never treated as markers, only doubled `__pairs__` are. This is the belt-and-braces backstop.

`bridge/api.test.ts` gained matching test coverage: each stripping rule individually, a fence-preservation case, a case sanitising text around a fence while leaving the fence untouched, and a boundary case proving `sendChunked` strips before chunking (so a `**bold**` pair straddling the 4096-char cut can't leak asterisks on one side).

## Rejected alternatives (and why)

- **`parse_mode: HTML` or `MarkdownV2`** — rejected. A single malformed entity in the reply causes Telegram to reject the entire `sendMessage` call with an HTTP 400, silently losing the message. Deterministic stripping can't fail that way.
- **Structured JSON envelope for replies** — rejected for the same reason: adds a failure mode (malformed/unparseable envelope) that can lose a message, for no benefit over plain-text stripping.

## Wiki pages touched by this ingest

- [[capabilities/telegram-frontend]] — added a "Reply formatting (plain text, no parse_mode)" section documenting the sanitiser rules and the parse_mode-rejection rationale; updated the chunking-boundary gotcha to note stripping runs pre-chunk.

## Relationships

- [[capabilities/telegram-frontend]]
- [[capabilities/send-gate]] — unrelated system, but shares the same bridge/transport layer
