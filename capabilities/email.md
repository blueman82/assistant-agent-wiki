---
title: "Email (Gmail via MCP)"
type: capability
created: 2026-07-24
last_updated: 2026-07-24
sources: ["prompts/system.md", "rachel.ts", "gate/sendGate.ts", "tasks/inbox-brief-launchd.plist", "tasks/inbox-brief.md", "AGENTS.md"]
tags: [capability, email, gmail, mcp, tool-routing, drafts, send-gate, one-shot, untrusted-content]
---

## What it does

"Email" routes to **Gmail** — Gary's personal account (`gjharrison01@gmail.com`) — via the
`mcp__claude_ai_Gmail__*` MCP tools. This is the **default and only** wired email surface:
`prompts/system.md:11` states it as the tool-routing rule, and `mcp__claude_ai_Gmail__*` is in
`DEFAULT_ALLOWED_TOOLS` in `rachel.ts`.

> **Historical note.** Older `CLAUDE.md` prose describes email defaulting to Outlook via
> claude-in-chrome, with Gmail as the "personal email" opt-in. That is **not** what the brain says
> today — `prompts/system.md`'s routing rules are the operative contract, and they name Gmail as the
> default with no Outlook path. Rachel is personal-only since the depersonalisation work (see
> [[investigations/2026-07-21-repo-depersonalisation]]).

## Tool names

| Purpose | Tool |
|---|---|
| Find threads | `mcp__claude_ai_Gmail__search_threads` (Gmail query syntax — `is:unread`, `from:<name>`) |
| Read one thread | `mcp__claude_ai_Gmail__get_thread` |
| Read one message | `mcp__claude_ai_Gmail__get_message` |
| Compose | `mcp__claude_ai_Gmail__create_draft` |
| Labels | `create_label`, `list_labels`, `label_message`/`label_thread`, `unlabel_message`/`unlabel_thread` |
| Sensitive-label helpers | `apply_sensitive_message_label`, `apply_sensitive_thread_label` |
| Drafts | `list_drafts` |

## Reading vs sending — the asymmetry that matters

**Read**: `search_threads` with a query, then `get_thread` for the one you want.

**Send: there is no send.** The Gmail connector wires **no send tool at all** — drafting via
`create_draft` is the only outbound path, and a human completes the send in Gmail itself. This is a
property of the connector, not a policy Rachel enforces.

The consequence is recorded on [[capabilities/send-gate]] and in `AGENTS.md`: because no send tool
exists, **there is nothing for the send gate to intercept on the email path.** `GATED_TOOL_NAMES` in
`gate/sendGate.ts` contains five entries — one Slack send and four Calendar mutators — and **no
Gmail tool** (verified against the constant). `create_draft` is deliberately left ungated: gating a
draft would break the confirm-then-send flow for no security benefit.

`prompts/system.md:35` still says "always confirm with the operator before sending any email". That
rule is a UX contract with no mechanical enforcement behind it, and it costs nothing precisely
because the connector can't send anyway.

## The proactive consumer — Inbox Brief

The main automated reader of this surface is [[capabilities/inbox-brief]] — a recommend-only sweep
that classifies recent mail and pushes a brief to Telegram. Two hardening details are set on its
launchd job (`tasks/inbox-brief-launchd.plist`, verified from the plist):

- **`RACHEL_ALLOWED_TOOLS=Read,Write,Bash,mcp__claude_ai_Gmail__*`** — the one-shot is narrowed to
  the minimum set. A run that reads potentially hostile email must not carry Chrome, WebFetch, or
  `mcp-exec`. The env seam can only *remove* tools from `DEFAULT_ALLOWED_TOOLS`, never add
  (see [[capabilities/proactive-layer]]).
- **`RACHEL_UNTRUSTED_CONTENT=1`** — blocks memory writes for the whole run regardless of the tool
  narrowing. Rationale, quoted from the plist: a hostile email must never persuade the run to
  "remember" attacker text, which would otherwise compromise the system prompt on **every future
  turn, on every surface** ([[capabilities/memory]] injects the index into every turn).

Mail alerts flow through the `push.ts` chokepoint under the `mail` family — event-id
`mail:<threadId>`, state `<tier>:<latest-message-id>`, severity `urgent` or `normal`.

## Constraints / gotchas

- **No send tool.** Anything that reads like "Rachel emailed X" is wrong — she drafted it and a
  human sent it.
- **Untrusted content by construction.** Email bodies are third-party text. `prompts/system.md`'s
  ad-hoc backgrounding rules require any pasted email body handed to a spawned agent to be fenced
  and labelled as untrusted source material, never inlined as if Gary wrote it (see
  [[sources/2026-07-22-adhoc-background-escalation]]).
- **Bash cannot substitute.** A Bash command hitting a send API directly (including Gmail's
  `messages/send`) is denied outright by `gate/bashPatterns.ts` with a message pointing back to the
  MCP tools — though note that path only exists inside `rachel.ts`'s own `query()`; a detached
  `claude -p` never loads it.
- **Message text for pushes comes from a file, never argv.** A swept email body containing a
  send-looking string would otherwise trip the Bash send-pattern gate on Rachel's own push call
  (`prompts/system.md:203`).

## Relationships

- [[capabilities/inbox-brief]] — the recommend-only sweep that is this surface's main automated
  consumer
- [[capabilities/send-gate]] — why the email path has nothing to gate, and why drafts stay ungated
- [[capabilities/proactive-layer]] — the `mail` family, the push chokepoint, and the
  `RACHEL_ALLOWED_TOOLS` narrowing seam
- [[capabilities/calendar]] — the sibling default-routed personal surface, which *does* have gated
  mutators
- [[architecture/mcp-integrations]] — how the Gmail connector is wired
- [[capabilities/memory]] — the store `RACHEL_UNTRUSTED_CONTENT=1` protects during a mail sweep
