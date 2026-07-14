---
title: "Terminal tool echoes untruncated"
type: source
created: 2026-07-14
last_updated: 2026-07-14
sources: ["rachel.ts"]
tags: [terminal, repl, emit]
---

## What changed

`rachel.ts`'s tool-use summary lines (`kind: "tool"` on the typed emit channel) were capped at 100 characters via `.slice(0, 100)` in two places: Bash command echoes and the JSON-stringified inputs of all other non-file tools. Both caps removed 2026-07-14 — tool echoes now print in full.

## Why

Gary reported truncated output when talking to Rachel in the terminal and asked for all truncation removed.

## What was NOT truncation (left alone)

- `bridge/api.ts` `slice` calls — Telegram message chunking (Telegram's 4096-char message limit), splits into multiple messages, loses nothing
- `bridge/telegram-bridge.ts` `slice` calls — glob-pattern matching, log tailing, and file-extension sanitising
- Read/Write/Edit summaries — always printed the full file path, never capped

## Scope of impact

Terminal REPL only in practice. The Telegram bridge filters the emit channel to `kind === "text"`, so tool echoes never reach the phone regardless (see [[capabilities/telegram-frontend]]).

## State at ingest

Change verified by `npm run typecheck` and `npm test` (107/107 pass + probe suite). Committed as `0f9d5da` on branch `fix/bridge-409-backoff` (auto-commit hook), not merged at time of ingest. Per the 2026-07-14 coordination agreement, that branch is slated to be closed unmerged (superseded by the bridge self-monitoring PR) and `0f9d5da` re-raised separately — this change rides on that re-raise.

## Relationships

- [[architecture/overview]] — documents the typed emit channel this touches
