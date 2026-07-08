---
title: "Extending system.md"
type: pattern
created: 2026-06-07
last_updated: 2026-07-08
sources: ["prompts/system.md", "CLAUDE.md", "rachel.ts"]
tags: [pattern, system-prompt, behaviour]
---

## Problem

You want to change how Rachel behaves — new capability, different routing, new ground rule — without breaking existing behaviour or touching TypeScript.

## Approach

Edit `prompts/system.md` only. The file is loaded fresh on every `rachel.ts` startup, so changes take effect immediately on the next run. No rebuild, no restart of a server — just save and re-run.

Structure to follow:
1. Add a new `### Section` under `## Capabilities and how to use them`
2. List the tool names, read pattern, write pattern, and confirmation rules
3. Add any routing rules to `## Default routing` if the new capability needs a keyword trigger

## Example

Adding the Slack capability (2026-06-29) — the two-part change in practice:

1. **Allowlist (TypeScript)** — `rachel.ts`, one line: add `"mcp__claude_ai_Slack__*"` to `allowedTools`. This is the *can it touch the tool* grant.
2. **Prompt (markdown)** — `prompts/system.md`, a routing line + a `### Slack (via MCP)` section: which tools, read vs send, and the draft→confirm→send rule. This is the *how and when*.

```markdown
### Slack (via MCP)
- Use the `mcp__claude_ai_Slack__*` tools. This is Gary's personal Slack.
- To send: draft with `slack_send_message_draft`, confirm with Gary, then `slack_send_message`.
```

See [[sources/2026-06-29-wire-in-slack]] for the full change.

## Trade-offs

- **Pro**: zero TypeScript required; changes are instant
- **Pro**: the system prompt is the single source of truth for behaviour
- **Con**: no validation — a poorly written instruction can confuse the agent without a clear error
- **Con**: tool availability is still constrained by `allowedTools` in `rachel.ts` — adding a capability that needs a new MCP tool requires a TypeScript change too (see the Slack example above: allowlist entry + prompt section)

## Relationships

- [[architecture/overview]] — where system.md fits in the architecture
- [[architecture/mcp-integrations]] — when a TypeScript change IS needed
