---
title: "Send Gate"
type: capability
created: 2026-07-07
last_updated: 2026-07-24
sources: ["rachel.ts", "gate/sendGate.ts", "gate/memoryGate.ts", "gate/auditLog.ts", "gate/surfaces/telegram.ts", "gate/surfaces/queue.ts", "bridge/telegram-bridge.ts", "bridge/notify.ts", "proactive/push.ts", "prompts/system.md", "AGENTS.md", "node_modules/@anthropic-ai/claude-agent-sdk/sdk.d.ts"]
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
- **Fail-closed**: no approval within the gate's own internal timer → denied with a message telling the agent to redraft or ask Gary directly. A hook callback that throws is also denied, not allowed through. The SDK's own behaviour is version-dependent and split across the two failure modes (live spikes, `AGENTS.md`): a hook that **throws** is swallowed and the tool proceeds — fail-*open*, re-verified on `@anthropic-ai/claude-agent-sdk` 0.3.216 (2026-07-21); a hook resolving after its declared `HookCallbackMatcher.timeout` now fails **closed** on 0.3.216 (1s declared timeout vs 8s hook → blocked), where the 0.2-era SDK let the tool proceed. The gate does not rely on either: the try/catch is load-bearing because the throw path is still fail-open, and the internal deny-timer (strictly shorter than any matcher timeout, so it still settles first) is now defence-in-depth rather than the sole timeout enforcement. Do not relax either — the split changed once across a minor version and can change again.
- **Approval surfaces** (first answer wins): terminal y/n (interactive TTY), Telegram inline Approve/Deny buttons, or the dashboard queue file under `~/.claude/coderails-dashboard/approvals/` (`DEFAULT_QUEUE_DIR` in `gate/surfaces/queue.ts`).
- **Telegram surface routing**: the Telegram approval surface (`gate/surfaces/telegram.ts`) does not poll Telegram itself — it exposes `handleCallbackQuery`, and the Telegram front-end bridge (`bridge/telegram-bridge.ts`, see [[capabilities/telegram-frontend]]) owns the single `getUpdates` long-poll loop and feeds matching `callback_query` updates into it. A button tap is routed ahead of any queued chat turn, since a gate decision may be blocking a turn already in flight.
- **Bash defense-in-depth**: a `Bash` command matching a known send-API pattern (Slack `chat.postMessage`, Telegram `sendMessage`, Gmail `messages/send`, a `POST` to the Calendar events endpoint) is denied outright and redirected to the corresponding MCP tool.
- **Audit**: every attempt and decision is appended to `~/.rachel/send-gate-audit.jsonl`. Since PR #68 (2026-07-24), the sibling memory write gate (`gate/memoryGate.ts`, [[capabilities/memory]]) writes its own deny decisions to this same `RACHEL_AUDIT_LOG_PATH`-resolved file, distinguished by a `surface` label (`untrusted-lockout`, `untrusted-lockout-bash`, `frontmatter-schema`, `internal-error`) rather than a separate log — see [[sources/2026-07-24-memorygate-audit-logging]].

## Constraints / gotchas

- **Accepted residual, not closed**: browser-automation sends (`mcp__claude-in-chrome__*` driving the Slack/Gmail web UI) aren't pattern-matchable and aren't gated. Detection is audit-log-only — a documented tradeoff, not an oversight.
- **`bridge/notify.ts` and `proactive/push.ts` are deliberately ungated, by design not oversight** (see [[capabilities/inbox-brief]] and [[capabilities/proactive-layer]]). Both send only to the operator's own configured Telegram chat — no destination argument exists in either (`notify.ts` takes exactly one arg, a file path; the push CLI takes exactly five, none a chat id; both are test-pinned) — putting them in the same trust class as the bridge's own reply/alert messages and the approval surface's own sends, both already ungated; the gate's threat model is sends *to others* (Slack channels, Calendar invitees), not a notification to Gary himself. Both also read their message text from a file rather than argv, specifically so a swept email body containing a string like `api.telegram.org/.../sendMessage` can't land in the Bash `tool_use` command and trip the "Bash defense-in-depth" block above on Rachel's own legitimate call to the script.
- **Slack tool-name confirmation is documentation-sourced, not live-introspected** for this gate's build — if Rachel's live Slack tool names ever drift from what's documented, `GATED_TOOL_NAMES` in `gate/sendGate.ts` silently stops matching. Closing this needs a startup-time live schema check or a future hook-probe run.
- **`permissionMode` is not a security boundary — the gate does not rely on it.** As of PR #59 (2026-07-23) Rachel's SDK session runs `permissionMode: "bypassPermissions"` (was `"auto"`). The gate is unaffected: `bypassPermissions` acts on the `canUseTool` permission-check path, whereas a `PreToolUse` hook deny is a separate path the SDK honours regardless of mode (`sdk.d.ts:4128`: "PreToolUse hook denies bypass canUseTool"). The mode flip only changes which calls get auto-*allowed*, never whether the gate's *deny* is respected. Sourced from SDK type docs, not an empirical spike of this mode+hook combination — see [[sources/2026-07-23-pr59-bypass-permissions]]. That source also flags an open question: PR #59 set the mode without the documented-as-required `allowDangerouslySkipPermissions: true`, and its runtime consequence is unconfirmed.

- **Under bypass, `gate/bashPatterns.ts` is the only send enforcement left in bridge turns — and it has named holes.** Recorded by [[sources/2026-07-23-rejection-rca-and-fix-list]] (fix-list item 14). The Bash defence-in-depth block above is not a secondary layer in that context; it is the layer. Known gaps, **verified still present 2026-07-24**: `curl --data` POSTs without an explicit `-X POST` evade the calendar detector, and Telegram `sendVoice`/`sendDocument`, Slack `chat.update`/`chat.delete`, and Gmail `drafts.send` have no patterns at all. Closing this is **decided, not built** (medium, with tests).

## Relationships

- [[capabilities/slack]] — Slack's `slack_send_message` is one of the gated tools
- [[capabilities/telegram-frontend]] — owns the `getUpdates` loop this gate's Telegram surface depends on for callback delivery
- [[capabilities/inbox-brief]] — consumer of the deliberately-ungated `notify.ts` path documented above
- [[capabilities/proactive-layer]] — the `push.ts` chokepoint, the other deliberately-ungated own-chat sender
- [[architecture/mcp-integrations]] — the tool surfaces the gate sits in front of
- [[patterns/extending-system-md]] — the prompt-level draft-first contract this gate enforces mechanically
- [[sources/2026-07-23-pr59-bypass-permissions]] — the `permissionMode` flip to `bypassPermissions` and why the gate still fires under it
- [[sources/2026-07-23-rejection-rca-and-fix-list]] — fix-list item 14: `bashPatterns.ts` as the sole remaining send enforcement under bypass, with its named coverage holes
- [[sources/2026-07-24-memory-hardening-cluster]] — a sibling `PreToolUse` hook, `gate/memoryGate.ts` (PR #64), added alongside this one; same detached-`claude -p`-loads-no-hooks blind spot, and the same three-round path-resolution bypass pattern (string check vs. filesystem symlink resolution) worth reading if extending this gate
- [[sources/2026-07-24-memorygate-audit-logging]] — PR #68: the memory gate now shares this gate's audit file and `appendAudit` pattern, having previously audited nothing
