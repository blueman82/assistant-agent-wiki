---
title: "Architecture Overview"
type: architecture
created: 2026-06-07
last_updated: 2026-06-07
sources: ["secretary.ts", "prompts/system.md", "CLAUDE.md"]
tags: [architecture, sdk, core]
---

## Summary

The secretary splits cleanly into **plumbing** (`secretary.ts`) and **brain** (`prompts/system.md`). To change what the secretary does, edit the brain. The plumbing rarely needs touching.

## Key components

**secretary.ts** (~190 lines)
- Wraps the Agent SDK's `query()` in a REPL loop
- Loads `prompts/system.md` as the system prompt at startup
- Streams `assistant` messages (text + tool-use) to stdout
- Session continuity: captures `session_id` from the SDK `init` message, passes it as `resume` on subsequent turns — this is what makes the secretary remember context across turns in the same session
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
| `SECRETARY_MODEL` | `claude-sonnet-4-6` | Model |
| `SECRETARY_MAX_TURNS` | `200` | Max agent turns per request |

## Relationships

- [[architecture/mcp-integrations]] — which tools the agent can call
- [[patterns/extending-system-md]] — how to safely evolve the brain
