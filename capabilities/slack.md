---
title: "Slack Capability"
type: capability
created: 2026-06-29
last_updated: 2026-06-29
sources: ["prompts/system.md", "rachel.ts"]
tags: [capability, slack, mcp]
---

## What it does

Reads and sends messages on Gary's **personal** Slack via the claude.ai Slack MCP connector. Sending is gated: Rachel drafts, Gary confirms, then it sends — same rule as email.

## Tool names

`mcp__claude_ai_Slack__*` (claude.ai Slack connector, same family as Gmail/Calendar). Granted via `allowedTools` in `rachel.ts`.

Key tools:

| Action | Tool |
|--------|------|
| Find a channel | `slack_search_channels` |
| Find a person | `slack_search_users` |
| Search message content | `slack_search_public` (no extra consent) |
| Search incl. DMs/private | `slack_search_public_and_private` (needs Gary's consent per call) |
| Read a channel / DM | `slack_read_channel` |
| Read a thread | `slack_read_thread` |
| Draft a message | `slack_send_message_draft` |
| Send a message | `slack_send_message` |

## How to invoke

- "Catch me up on #channel" → `slack_search_channels` then `slack_read_channel`
- "What did <person> say about X" → `slack_search_public` with `from:` modifier
- "Message <person> that ..." → draft with `slack_send_message_draft`, show Gary, **send only after he confirms**

## Constraints / gotchas

- **Confirm before sending** — never send unprompted. The send flow is draft → confirm → send. (See [[architecture/overview]] ground rules.)
- **Default to public search.** `slack_search_public` needs no extra consent; `slack_search_public_and_private` (which covers DMs and private channels) requires explicit consent per call, so Rachel asks first.
- Personal Slack only. This replaced an earlier curl/API token path — see [[sources/2026-06-29-wire-in-slack]].
- Cannot post to externally shared (Slack Connect) channels.

## Relationships

- [[architecture/mcp-integrations]] — the Slack tool surface in the wiring table
- [[patterns/extending-system-md]] — Slack is the worked example of a capability needing both an allowlist entry AND a prompt section
- [[sources/2026-06-29-wire-in-slack]] — the change that added this
