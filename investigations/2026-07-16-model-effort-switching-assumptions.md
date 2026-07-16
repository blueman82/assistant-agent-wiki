---
title: "Runtime model/effort switching — plan assumptions vs. enforced code"
type: investigation
created: 2026-07-16
last_updated: 2026-07-16
sources: ["rachel.ts", "bridge/telegram-bridge.ts", "proactive/allowedTools.ts", "node_modules/@anthropic-ai/claude-agent-sdk/sdk.d.ts", "AGENTS.md"]
tags: [investigation, model-config, allowed-tools-seam, telegram-commands, sdk-options]
---

## Question

Gary asked what a plan to add runtime model+effort switching (CLI `/model`/`/effort`, Telegram `/model`/`/effort`, a new `proactive/modelConfig.ts` mirroring `proactive/allowedTools.ts`, session-only not persisted) assumes but that isn't actually enforced in code.

## Findings

**`MODEL` is a module-load constant, `allowedTools` is a per-turn read — the asymmetry the plan must close.** `rachel.ts:55` reads `RACHEL_MODEL` once at module load; `runTurn` passes `model: MODEL` into the SDK options object (`rachel.ts:170`). `allowedTools` is deliberately re-read **inside** `runTurn` (`rachel.ts:175`, comment: "Env read here, per call, not at module load — launchd/spawn environments differ per invocation"). Mirroring `allowedTools.ts` as a *file pattern* is not sufficient — the `model: MODEL` call site itself must change from a frozen constant to a per-turn read of new mutable state, which is the change `allowedTools` already made and `model` hasn't.

**`model` and `effort` are real, typed, top-level `Options` fields in the installed SDK** (verified in `sdk.d.ts`): `model?: string` and `effort?: EffortLevel` where `EffortLevel = 'low' | 'medium' | 'high' | 'xhigh' | 'max'`. Nothing in the SDK types blocks passing either per-turn.

**`resume` does not obviously pin the model, because `rachel.ts` calls `query()` fresh every turn.** `runTurn` constructs a new `options` object and calls `queryFn({ prompt, options })` on every call (`rachel.ts:202`) — it never holds one long-lived `Query` stream across turns. `resume: sessionId` is just one field of that fresh object. So a changed `model`/`effort` read at the top of the next `runTurn` would apply to that turn's fresh `query()` call. This is inferred from control flow, not proven by a live spike — the wiki's own precedent (send-gate fail-open, confirmed only by "a live spike against the installed SDK", see [[capabilities/send-gate]] / `AGENTS.md`) is that SDK runtime behavior should be spiked, not inferred, before being relied on. Static analysis says this should work; a live smoke test is the outstanding step, not a blocker to building it.

**Terminal, bridge, and every one-shot are separate OS processes — "shared in-memory state" is misleading.** `bridge/telegram-bridge.ts` imports `runTurn`/`getSessionId`/`resetSession`/`telegramSurface` from `rachel.ts` as a module, but runs via `npm run bridge` → `tsx bridge/telegram-bridge.ts`, a different process from `npm start` → `tsx rachel.ts`. Every headless `bin/rachel "..."` one-shot (`inbox-brief`, `proactive-calendar`) is yet another `exec tsx rachel.ts`, reading `RACHEL_MODEL` from its own env at its own module load. A new `proactive/modelConfig.ts` holding "shared" state is shared only within whichever single process imported it — a CLI `/model` change and a Telegram `/model` change can never see each other, and no in-memory switch reaches a scheduled one-shot (those are fresh `exec`s reading env, not the running module). Same scoping already applies to `sessionId` (`rachel.ts:127`, "module-scoped so it persists across turns within a session").

**`/status` reads `process.env["RACHEL_MODEL"]` directly** (`bridge/telegram-bridge.ts:535`), not any getter. If `/model` only mutates a new in-memory module, `/status` goes stale the moment `/model` is used once — it will keep reporting the boot-time env value. Must be updated to read the same state `/model` writes.

**Session-only + bridge self-restart = silent reversion.** The bridge is designed to self-restart on failure (409-backoff threshold-5 exit, loop-watchdog — see [[capabilities/telegram-frontend]]). A restart is a new process; any in-memory override reverts to `RACHEL_MODEL`/default effort with no alert, since nothing in `proactive/push.ts` or the bridge startup alert currently knows to announce "model reverted."

**The Telegram command surface is a flat if-chain gated by chat-id, not a dispatcher.** `/reset`, `/status`, `/stop` are exact-string checks (`bridge/telegram-bridge.ts:521/526/541`) that run only after `fromChatId !== config.chatId` is checked (line 512) — that chat-id gate is the real authorization boundary for new `/model`/`/effort` commands, consistent with the proactive layer's "treat swept content as data" invariant (see [[capabilities/proactive-layer]]). The CLI's equivalent (`rachel.ts:296-317`) has no access-control check at all — safe today only because it's a local terminal process; a different trust model than Telegram's, worth stating explicitly rather than assuming parity.

**`allowedTools.ts`'s specific invariants (remove-only, zero-result throws, unknown-entries-dropped) don't map onto model/effort.** Those exist for injection-hardening a narrowed one-shot's hostile env; there's no subset/superset relationship for a model id or effort level to enforce. "Mirror `allowedTools.ts`" should mean structural mirroring (pure function, per-turn read, stderr logging) — not literal reuse of narrowing logic, which has no meaning here.

**`effort` interacts with `thinking`/`alwaysThinkingEnabled`.** Per `sdk.d.ts`, `effort` "works with adaptive thinking to guide thinking depth" and interacts with `thinking: ThinkingConfig` and the deprecated `maxThinkingTokens`. A bare `/effort` command that only sets `effort` without considering `thinking` config may not behave as expected on all models — worth checking against the specific model in use if `/effort` is implemented literally.

**Wiki coverage gap confirmed**: no existing wiki page mentions `/model`, `/effort`, or reasoning effort as a concept — this page is the first documentation of the topic. `architecture/overview.md`'s config table only lists `RACHEL_MODEL`, `RACHEL_MAX_TURNS`, `RACHEL_ALLOWED_TOOLS`. The Telegram command triple (`/reset`, `/status`, `/stop`) was also previously undocumented in the wiki (only in source) — this page is the first place it's enumerated with line references.

## Relationships

- [[architecture/overview]] — existing config table (`RACHEL_MODEL`, `RACHEL_ALLOWED_TOOLS`) this investigation extends
- [[capabilities/proactive-layer]] — `proactive/allowedTools.ts`'s narrow-never-add seam, the structural (not behavioral) template for a new `modelConfig.ts`
- [[capabilities/telegram-frontend]] — bridge self-restart behavior that makes session-only state silently revert
- [[capabilities/send-gate]] — the SDK fail-open spike precedent this investigation's "verify before trusting" recommendation follows
