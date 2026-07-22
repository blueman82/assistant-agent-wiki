---
title: "PR #17 — Telegram Image Reception"
type: source
created: 2026-07-08
last_updated: 2026-07-22
sources: ["bridge/telegram-bridge.ts", "bridge/api.ts", "bridge/telegram-bridge.test.ts", "prompts/system.md"]
tags: [telegram, bridge, image, photo, capability, security, superseded-pdf-scope]
---

## Origin

PR #17 (`feat/telegram-image-reception`) — merged 2026-07-08.
Files changed: `bridge/api.ts`, `bridge/telegram-bridge.ts`, `bridge/telegram-bridge.test.ts`, `prompts/system.md`.

## What it adds

Rachel can now receive photos and image documents sent by Gary via Telegram. Previously `handleMessage` dropped any message without `msg.text`.

## Key decisions

**Photo selection**: `photo[photo.length - 1]` — Telegram sends the array sorted ascending by size; the last entry is always the largest variant. A `reduce` was initially used and replaced with this simpler approach.

**Token leak prevention**: The download URL (`https://api.telegram.org/file/bot<TOKEN>/...`) is constructed inside `downloadFile` and never returned. The original design had a separate `getFileUrl()` that returned the full URL as a string — this was a security finding (the URL could appear in error messages and reach the launchd log). Collapsed into a single function.

**20 MB limit guard**: If Telegram's `getFile` API returns no `file_path` (happens for files >20 MB), a clear error is thrown — "file may exceed the 20 MB API limit" — rather than producing a confusing HTTP 404 on the download.

**PDF scope dropped**: An initial implementation accepted `application/pdf` documents. Removed as unasked-for scope (CLAUDE.md: "No features beyond what was asked").

> **Superseded 2026-07-22.** This exclusion held for two weeks. PR #54 (`feature/telegram-pdf-ingestion`, merged `6e96a41`) reintroduced `application/pdf` document support — not a re-litigation of this decision but a new request: Gary explicitly asked ("I need you to be able to read PDF files"), so the YAGNI basis for dropping it here no longer applies. This entry is left as-is rather than rewritten: it accurately records why PDF support was *out* of scope at the time PR #17 shipped. See [[sources/2026-07-22-telegram-pdf-ingestion]] for the reversal and [[capabilities/telegram-frontend]]'s "Image and PDF reception" section for the current, merged behaviour.

**Non-image reply**: Non-image documents receive an immediate user-facing reply rather than a silent drop.

**system.md contract**: `prompts/system.md` now documents the `[image: /path]` input format and instructs Rachel to call `Read` on the path. Without this, image handling depended on model instinct rather than contract.

**`downloadFileFn` seam**: The download function is injectable via `CreateBridgeOptions` so tests mock it without touching the filesystem or network. The real `downloadFile` streams via `fetch` + `ReadableStream` pump + Node `WriteStream`. `writer.destroy()` is called on all `reject` paths to avoid file descriptor leaks in the long-running bridge process.

## Temp file path

`~/.rachel/tmp/<fileId>.<ext>` — created on first use. No cleanup is implemented (known debt).

## Input format to Rachel

```
[image: /Users/harrison/.rachel/tmp/<fileId>.jpg]
<caption if present>
```

Rachel is instructed (via `prompts/system.md`) to call `Read` on the path before responding.

## Known debt

- `~/.rachel/tmp/` grows unboundedly — no cleanup scheduled.
- No `AbortSignal`/timeout on `downloadFile`'s fetch — a hung Telegram CDN stalls the poll loop.

## Relationships

- [[capabilities/telegram-frontend]] — the capability this PR extended
- [[architecture/overview]] — `bridge/api.ts` role in the plumbing layer
- [[sources/2026-07-22-telegram-pdf-ingestion]] — PR #54, which supersedes this page's "PDF scope dropped" decision by explicit later request
