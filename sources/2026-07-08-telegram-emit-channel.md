---
title: "Telegram emit channel (typed kind classification)"
type: source
origin: "PR #16 fix/telegram-emit-channel (assistant-agent), merged 6552813"
created: 2026-07-08
last_updated: 2026-07-08
sources: ["rachel.ts", "bridge/telegram-bridge.ts", "CLAUDE.md", ".claude/evals-fix-telegram-emit-channel.json"]
tags: [telegram, bridge, emit-channel, fix, decision]
---

## Problem

`runTurn`'s emit callback was a flat, untyped stream: `(line: string) => void`. The Telegram bridge forwarded every emitted line into its reply buffer, so tool-use echoes (`  [ToolName] summary`) and the turn's `[Rachel] done turns=N cost=$X` completion footer leaked into Gary's Telegram replies alongside the model's actual answer.

## Fix

`rachel.ts` now emits a `(line, kind)` pair, `TurnEmit = (line: string, kind: TurnEmitKind) => void`, with `TurnEmitKind = "text" | "tool" | "meta"`:
- `"text"` — the model's own reply text (`block.type === "text"` content)
- `"tool"` — a tool-use summary line
- `"meta"` — the `[Rachel] done turns=…` footer, emitted on the SDK's `result` message

The terminal REPL (`rachel.ts`'s `main()`) is unchanged in behaviour — it prints every kind via `(line, _kind) => process.stdout.write(...)`. `bridge/telegram-bridge.ts`'s `drainFifo` now buffers with `if (kind === "text") buffer.push(line)`, so only model text reaches the Telegram reply.

Two lines still reach Telegram outside this filter, by design: the bridge's own `"[Rachel] error: ..."` line (pushed directly into the buffer in the `catch`, not routed through `emit`) and the `"(no output)"` fallback string sent when a turn produces no text lines at all.

**Classification is structural, not regex.** The fix deliberately switches on the typed `kind` field rather than pattern-matching on line content (e.g. sniffing for a leading `[` or `done turns=`) — a regex approach would be fragile against any future tool name or footer wording change.

## Review-driven classification-pinning test

The independent review process for this PR included a verifier agent (no implementation context, fresh clone only) tasked with confirming three criteria by reading `rachel.ts` and `bridge/telegram-bridge.ts` in full: (1) the terminal REPL still receives and prints both tool-use summaries and the done/cost footer — nothing was deleted from `runTurn`, only classified; (2) the bridge's reply buffer collects only model text, excluded by a typed/structural mechanism rather than regex; (3) the bridge's own error line still reaches the Telegram reply on a thrown turn. This is recorded as eval **N2** in `.claude/evals-fix-telegram-emit-channel.json` (agent-run mode, no scripted cmd) and passed — the graded evidence cites exact line numbers (`rachel.ts:266`, `bridge/telegram-bridge.ts:134-136`, `137-138`, `144-145`).

Eval **N1** (scripted, P0) additionally pinned the fix with a real `runTurn` + `sendChunked` round-trip against a fake tool-use-then-text SDK stream, asserting the model's text survives while the tool echo and done footer are absent from the rendered Telegram message — with a negative control that reproduces the pre-fix leak on the frozen base SHA (`4a70688`). Eval N4 confirmed the `"(no output)"` fallback still fires on the filtered path so a text-free turn never sends an empty message.

## PR #15's rename reconciled mid-flight

This branch (`fix/telegram-emit-channel`) was in flight when PR #15 (the secretary → Rachel rebrand, see [[sources/2026-07-08-rachel-rebrand]]) merged into `main` first. The frozen evals (N1, N2) were amended in place — retargeting the probe's import path (`secretary.ts` → `rachel.ts`) and fabricated env var names (`SECRETARY_*` → `RACHEL_*`) — with every assertion left unchanged, since the "done turns=" absence check already matched the rebranded `[Rachel]` footer. The amendments are recorded in the evals file's `amendments` array, timestamped 2026-07-08T13:50:00Z, and are explicit that the negative controls were deliberately **not** amended — they intentionally run against the pre-rename base commit where the old names are still correct.

## Impact

Updated: [[capabilities/telegram-frontend]] (new "Reply-content contract" section), [[architecture/overview]] (brief `TurnEmit`/`TurnEmitKind` note on `rachel.ts`), [[index]].

## Relationships

- [[capabilities/telegram-frontend]] — the capability page this channel belongs to
- [[architecture/overview]] — `rachel.ts`'s role as plumbing, now including the typed emit channel
- [[sources/2026-07-08-rachel-rebrand]] — the concurrent rename this branch reconciled against
- [[sources/2026-07-08-telegram-reply-formatting]] — the earlier, separate fix for markdown formatting in replies (unrelated bug, same front-end)
