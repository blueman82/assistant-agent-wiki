---
title: "Architecture Overview"
type: architecture
created: 2026-06-07
last_updated: 2026-07-21
sources: ["rachel.ts", "prompts/system.md", "CLAUDE.md", "proactive/memoryIndex.ts", "proactive/sessionPersist.ts"]
tags: [architecture, sdk, core, memory, session-persistence]
---

## Summary

Rachel splits cleanly into **plumbing** (`rachel.ts`) and **brain** (`prompts/system.md`). To change what Rachel does, edit the brain. The plumbing rarely needs touching.

## Key components

**rachel.ts** (~190 lines)
- Wraps the Agent SDK's `query()` in a REPL loop
- Loads `prompts/system.md` as the system prompt at startup
- `runTurn` streams turn output via a typed emit channel: `TurnEmit = (line, kind) => void`, where `kind` is `"text"` | `"tool"` | `"meta"` — model text, tool-use echoes, and the done/cost footer respectively. The terminal REPL prints every kind; `bridge/telegram-bridge.ts` filters to `"text"` only (see [[capabilities/telegram-frontend]])
- Tool-use echoes (`kind: "tool"`) print in full — Bash commands and JSON tool inputs are no longer capped at 100 chars (cap removed 2026-07-14 after Gary hit truncated terminal output; see [[sources/2026-07-14-terminal-tool-echo-truncation]]). File-path summaries (Read/Write/Edit) were never truncated
- Session continuity: captures `session_id` from the SDK `init` message, passes it as `resume` on subsequent turns — this is what makes Rachel remember context across turns in the same session
- `/reset` clears `sessionId`, starting a fresh session
- The system prompt is composed fresh every turn via `proactive/memoryIndex.ts`'s `composeSystemPrompt`, which appends the persistent memory index (see [[capabilities/memory]]) to the base prompt before it's passed to `query()`
- `q` mid-turn aborts via `AbortController`

**prompts/system.md**
- All behavioural contracts live here: tool routing, ground rules, capability docs
- The correct place to add new capabilities or change routing

## Session model

One long-lived process, one session. `sessionId` is set on the first turn's `init` message and reused for all subsequent turns until `/reset` or process exit.

**Bridge exception (since PR #51, 2026-07-21).** For the Telegram bridge specifically, "process exit" no longer ends the session: `proactive/sessionPersist.ts` persists `sessionId` to `<repo>/.rachel/bridge-session.json` behind a `RACHEL_SESSION_FILE` env seam set only in `bridge/launchd.plist`, and `hydratePersistedSession()` (called from the bridge's CLI guard only) restores it on the next process start. A bridge restart therefore survives with the same conversation thread; the terminal REPL and all headless one-shots are unaffected (seam unset = today's exact in-memory-only behaviour). `resetSession()` also clears the persisted file, so `/reset` is not undone by a subsequent restart. See [[capabilities/memory]] and [[decisions/2026-07-21-rejected-shared-session-thread]] for why this is scoped to the bridge alone rather than a general shared-session design.

**Headless one-shot mode.** `bin/rachel "<prompt>" < /dev/null` runs a single turn and exits cleanly on its own — closed stdin means the REPL's `rl.question` hits EOF immediately after the turn completes, no `KeepAlive` process manager needed. Used by launchd-scheduled jobs (e.g. `tasks/inbox-brief-launchd.plist`, see [[capabilities/inbox-brief]]) and dashboard buttons. This mode has no bridge/REPL loop in front of it, so its ordinary text-reply path only reaches stdout — anything that needs to proactively reach Telegram from a one-shot run uses the standalone `bridge/notify.ts` sender or the `proactive/push.ts` chokepoint instead of the normal reply path (see [[capabilities/telegram-frontend]] and [[capabilities/proactive-layer]]). One-shots typically run with a narrowed toolset via `RACHEL_ALLOWED_TOOLS` — resolved per turn in `rachel.ts` by `proactive/allowedTools.ts`'s `resolveAllowedTools` against the exported `DEFAULT_ALLOWED_TOOLS`; the env var can only remove tools from that list, never add one, and a value filtering to zero tools throws rather than running tool-less.

**Proactive layer.** Since PRs #24–#26 and #28 (2026-07-15), Rachel also runs unprompted: a 30-minute deterministic launchd sweep (`proactive/sweep.ts`) plus scheduled LLM one-shots (calendar conflicts, inbox brief), all delivering through the single `proactive/push.ts` chokepoint (dedup, quiet hours, daily budget). See [[capabilities/proactive-layer]].

## Config

| Variable | Default | Effect |
|----------|---------|--------|
| `RACHEL_MODEL` | `claude-sonnet-4-6` | Model |
| `RACHEL_MAX_TURNS` | `200` | Max agent turns per request |
| `RACHEL_ALLOWED_TOOLS` | unset (= full default list) | Comma-separated narrowing of the agent's allowedTools; remove-only, zero-tools throws (see [[capabilities/proactive-layer]]) |
| `RACHEL_MEMORY_PATH` | unset (= `~/.rachel/memory/MEMORY.md`) | Path to the memory index composed into the system prompt every turn (see [[capabilities/memory]]) |
| `RACHEL_SESSION_FILE` | unset (= no persistence) | Bridge-only seam; set only in `bridge/launchd.plist`. Persists/restores `sessionId` across a bridge restart |

## Relationships

- [[architecture/mcp-integrations]] — which tools the agent can call
- [[patterns/extending-system-md]] — how to safely evolve the brain
- [[capabilities/telegram-frontend]] — consumer of the typed emit channel that filters replies to text-only; also the sole consumer of the bridge session-persistence exception above
- [[capabilities/memory]] — the persistent fact store composed into the prompt every turn
- [[decisions/2026-07-21-rejected-shared-session-thread]] — why memory and bridge session persistence are two separate mechanisms, not one
