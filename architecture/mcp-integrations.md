---
title: "MCP Integrations"
type: architecture
created: 2026-06-07
last_updated: 2026-06-29
sources: ["rachel.ts"]
tags: [architecture, mcp, tools]
---

## Summary

No MCP servers are spawned by the Rachel process. All MCP tools come from existing Claude Code connections. The surfaces are wired in via `allowedTools` in `rachel.ts`.

## Tool surfaces

| Surface | Tool prefix | Source | Purpose |
|---------|-------------|--------|---------|
| Gmail | `mcp__claude_ai_Gmail__*` | Claude Code session | Personal email (gjharrison01@gmail.com) |
| Google Calendar | `mcp__claude_ai_Google_Calendar__*` | Claude Code session | Personal calendar |
| Slack | `mcp__claude_ai_Slack__*` | Claude Code session | Personal Slack (read + send, confirm before sending) |
| Chrome extension | `mcp__claude-in-chrome__*` | Browser extension | General browser tasks |
| mcp-exec | `mcp__mcp-exec__*` | Claude Code session | Playwright fallback, code execution |

## Key wiring detail

The Chrome extension requires `extraArgs: { "chrome": null }` in the SDK query options (secretary.ts line ~92). Without this, `mcp__claude-in-chrome__*` tools won't appear to the agent even if the extension is connected.

## mcpServers

`mcpServers = {}` — deliberately empty. No servers are spawned. This differs from typical Agent SDK setups where you'd spawn e.g. a filesystem or database MCP server.

## Relationships

- [[architecture/overview]] — how tools are wired into the agent
- [[capabilities/email]] — Gmail tool usage patterns
- [[capabilities/calendar]] — Calendar tool usage patterns
- [[capabilities/slack]] — Slack tool usage patterns
