---
title: "Send Gate"
type: capability
created: 2026-07-07
last_updated: 2026-07-14
sources: ["rachel.ts", "gate/sendGate.ts", "gate/surfaces/telegram.ts", "gate/surfaces/queue.ts", "bridge/telegram-bridge.ts", "bridge/notify.ts", "prompts/system.md", "AGENTS.md"]
tags: [capability, gate, security, slack, calendar, telegram]
---

## What it does

A `PreToolUse` hook in `rachel.ts` (`gate/sendGate.ts`) intercepts every call to a gated (send-class) tool and blocks it until an approval surface resolves. This is the deterministic enforcement floor beneath the prompt-level draft-first rules in `prompts/system.md` — those rules remain the UX contract, not the only enforcement. There's no talking the agent around it: even if a send tool is called without asking Gary first, the gate still blocks and waits.

## Tool names

Gated (blocked pending approval):

| Tool |
|------|
| `mcp__claude_ai_Slack__slack_send_message` |
| `mcp__claude_ai_Google_Calendar__create_event` |
| `mcp__claude_ai_Google_Calendar__update_event` |
| `mcp__claude_ai_Google_Calendar__delete_event` |
| `mcp__claude_ai_Google_Calendar__respond_to_event` |

Gmail has no send tool (only `create_draft` + read/label/search) — nothing to gate there. Draft tools (`slack_send_message_draft`, Gmail `create_draft`) stay deliberately ungated: gating a draft would break the confirm flow for no security benefit.

## How it works

- **Per-item, one-shot**: approval is bound to a SHA-256 hash of the canonicalised (sorted-key) tool input. Approving one message never approves a different one. An approval is consumed on first use — a replay of the same hash is denied.
- **Fail-closed**: no approval within the gate's own internal timer → denied with a message telling the agent to redraft or ask Gary directly. A hook callback that throws is also denied, not allowed through (the SDK itself is fail-*open* on both thrown exceptions and timeout — the gate does not rely on it).
- **Approval surfaces** (first answer wins): terminal y/n (interactive TTY), Telegram inline Approve/Deny buttons, or the dashboard queue file under `~/.claude/coderails-dashboard/approvals/` (`DEFAULT_QUEUE_DIR` in `gate/surfaces/queue.ts`).
- **Telegram surface routing**: the Telegram approval surface (`gate/surfaces/telegram.ts`) does not poll Telegram itself — it exposes `handleCallbackQuery`, and the Telegram front-end bridge (`bridge/telegram-bridge.ts`, see [[capabilities/telegram-frontend]]) owns the single `getUpdates` long-poll loop and feeds matching `callback_query` updates into it. A button tap is routed ahead of any queued chat turn, since a gate decision may be blocking a turn already in flight.
- **Bash defense-in-depth**: a `Bash` command matching a known send-API pattern (Slack `chat.postMessage`, Telegram `sendMessage`, Gmail `messages/send`, a `POST` to the Calendar events endpoint) is denied outright and redirected to the corresponding MCP tool.
- **Audit**: every attempt and decision is appended to `~/.rachel/send-gate-audit.jsonl`.

## Constraints / gotchas

- **Accepted residual, not closed**: browser-automation sends (`mcp__claude-in-chrome__*` driving the Slack/Gmail web UI) aren't pattern-matchable and aren't gated. Detection is audit-log-only — a documented tradeoff, not an oversight.
- **`bridge/notify.ts` is deliberately ungated, by design not oversight** (see [[capabilities/inbox-brief]]). It sends only to the operator's own configured Telegram chat — no destination argument exists — putting it in the same trust class as the bridge's own reply/alert messages and the approval surface's own sends, both already ungated; the gate's threat model is sends *to others* (Slack channels, Calendar invitees), not a notification to Gary himself. It also reads its message from a file rather than argv specifically so a swept email body containing a string like `api.telegram.org/.../sendMessage` can't land in the Bash `tool_use` command and trip the "Bash defense-in-depth" block below on Rachel's own legitimate call to the script.
- **Slack tool-name confirmation is documentation-sourced, not live-introspected** for this gate's build — if Rachel's live Slack tool names ever drift from what's documented, `GATED_TOOL_NAMES` in `gate/sendGate.ts` silently stops matching. Closing this needs a startup-time live schema check or a future hook-probe run.
- `permissionMode: "auto"` on Rachel's SDK session is a model-classifier mode, not a security boundary — the gate does not rely on it.

## Relationships

- [[capabilities/slack]] — Slack's `slack_send_message` is one of the gated tools
- [[capabilities/telegram-frontend]] — owns the `getUpdates` loop this gate's Telegram surface depends on for callback delivery
- [[capabilities/inbox-brief]] — consumer of the deliberately-ungated `notify.ts` path documented above
- [[architecture/mcp-integrations]] — the tool surfaces the gate sits in front of
- [[patterns/extending-system-md]] — the prompt-level draft-first contract this gate enforces mechanically
