---
title: "Wire in personal Slack"
type: source
origin: "branch wire-in-slack (assistant-agent)"
created: 2026-06-29
last_updated: 2026-06-29
sources: ["secretary.ts", "prompts/system.md", "CLAUDE.md"]
tags: [slack, mcp, capability, decision]
---

## Key takeaways

- The secretary gained a fourth MCP capability: **personal Slack**, read + send, with confirm-before-send.
- The claude.ai Slack MCP connector (`mcp__claude_ai_Slack__*`) came online mid-session and is now the secretary's Slack.
- The change was two functional edits, no new code: one `allowedTools` entry in `secretary.ts`, one capability section + routing line in `prompts/system.md`. Textbook [[patterns/extending-system-md]].

## Decision: replace the curl path, don't add alongside

Gary's global `CLAUDE.md` previously defined the secretary's Slack as a **curl against the Slack API** with a token at `~/.claude/skills/slack/.env.ketchup`. The new MCP connector is a separate mechanism. Decision (Gary, 2026-06-29): **replace** — the MCP connector is the only Slack path. The `CLAUDE.md` line was updated to match, so the two no longer contradict.

## Send safety

Sending mirrors the email rule: draft with `slack_send_message_draft`, show Gary, send with `slack_send_message` only after he confirms. "Slack send" was added to the confirm-before-acting ground rule in `system.md`.

## Search consent

`slack_search_public` needs no extra consent and is the default. `slack_search_public_and_private` (DMs + private channels) requires explicit consent per call, so the secretary asks before using it.

## Impact

- New page: [[capabilities/slack]]
- Updated: [[architecture/mcp-integrations]] (Slack row), [[patterns/extending-system-md]] (Slack as the worked MCP-tool example)
- Also shipped this branch: a commit test gate running `npm run typecheck` (`.claude/test_command`).
