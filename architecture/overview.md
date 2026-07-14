---
title: "Architecture Overview"
type: architecture
created: 2026-06-07
last_updated: 2026-07-08
sources: ["rachel.ts", "prompts/system.md", "CLAUDE.md"]
tags: [architecture, sdk, core]
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
- `q` mid-turn aborts via `AbortController`

**prompts/system.md**
- All behavioural contracts live here: tool routing, ground rules, capability docs
- The correct place to add new capabilities or change routing

## Session model

One long-lived process, one session. `sessionId` is set on the first turn's `init` message and reused for all subsequent turns until `/reset` or process exit.

## Config

| Variable | Default | Effect |
|----------|---------|--------|
| `RACHEL_MODEL` | `claude-sonnet-4-6` | Model |
| `RACHEL_MAX_TURNS` | `200` | Max agent turns per request |

## Relationships

- [[architecture/mcp-integrations]] — which tools the agent can call
- [[patterns/extending-system-md]] — how to safely evolve the brain
- [[capabilities/telegram-frontend]] — consumer of the typed emit channel that filters replies to text-only
