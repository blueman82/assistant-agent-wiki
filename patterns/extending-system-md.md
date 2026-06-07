---
title: "Extending system.md"
type: pattern
created: 2026-06-07
last_updated: 2026-06-07
sources: ["prompts/system.md", "CLAUDE.md"]
tags: [pattern, system-prompt, behaviour]
---

## Problem

You want to change how the secretary behaves — new capability, different routing, new ground rule — without breaking existing behaviour or touching TypeScript.

## Approach

Edit `prompts/system.md` only. The file is loaded fresh on every `secretary.ts` startup, so changes take effect immediately on the next run. No rebuild, no restart of a server — just save and re-run.

Structure to follow:
1. Add a new `### Section` under `## Capabilities and how to use them`
2. List the tool names, read pattern, write pattern, and confirmation rules
3. Add any routing rules to `## Default routing` if the new capability needs a keyword trigger

## Example

Adding a "web research" capability:
```markdown
### Web research
- Use `WebSearch` to find information, `WebFetch` to read a specific URL
- Always cite sources in the response
- Do not summarise paywalled content
```

## Trade-offs

- **Pro**: zero TypeScript required; changes are instant
- **Pro**: the system prompt is the single source of truth for behaviour
- **Con**: no validation — a poorly written instruction can confuse the agent without a clear error
- **Con**: tool availability is still constrained by `allowedTools` in `secretary.ts` — adding a capability that needs a new MCP tool requires a TypeScript change too

## Relationships

- [[architecture/overview]] — where system.md fits in the architecture
- [[architecture/mcp-integrations]] — when a TypeScript change IS needed
