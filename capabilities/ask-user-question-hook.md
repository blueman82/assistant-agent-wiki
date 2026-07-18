---
title: "AskUserQuestion Hook"
type: capability
created: 2026-07-19
last_updated: 2026-07-19
sources: ["gate/askUserQuestionHook.ts", "gate/askUserQuestionHook.test.ts", "rachel.ts", "prompts/system.md", "evals/e3-ask-before-acting.md"]
tags: [capability, hook, gate, ask-user-question, clarification]
---

## What it does

A `PreToolUse` hook in `rachel.ts` (`gate/askUserQuestionHook.ts`) intercepts and denies all calls to the `AskUserQuestion` tool with a deterministic message redirecting Rachel to ask clarifying questions in conversational text instead. This enforces a hard constraint: when a request is ambiguous, missing context, or internally conflicting, Rachel must ask the user directly via text rather than attempting a tool call that cannot be rendered in the text-only transport (Telegram, terminal, headless one-shot).

The hook has no host-side renderer for the tool's interactive question UI — it exists only in the Claude SDK's agent surface, not in terminal or Telegram. Blocking the tool ensures Rachel pivots to conversational asking when clarification is needed.

## Why it exists

`AskUserQuestion` is designed for interactive host environments (Claude.ai) that can render a multiple-choice UI. Rachel runs in text-only transports:
- Terminal REPL (stdin/stdout)
- Telegram (text messages + inline buttons for approvals, not tool-driven UI)
- Headless one-shots (launchd, dashboard buttons)

None of these can render the tool's question/options UI. Rather than silently skipping the tool or falling back to guessing, the hook explicitly denies it and forces conversational asking, making the behaviour deterministic and testable (E3 evaluation suite).

## How it works

- **Hook name**: `AskUserQuestion`
- **Event**: `PreToolUse` only (other events pass through unchanged)
- **Action**: returns `denyOutput()` with the message: `"No host renderer available — ask your question directly in conversational text instead."`
- **Fallback**: if the hook itself throws an exception (defensive), it catches and denies with `"Internal hook error — denied by default."`
- **Non-matching tools**: all other tools pass through with an empty object `{}` — the hook only acts on `AskUserQuestion`

The hook is created once at module scope in `rachel.ts` and runs on every PreToolUse event in every turn, parallel to the send gate (`gate/sendGate.ts`).

## Test coverage (gate/askUserQuestionHook.test.ts)

- `AskUserQuestion` tool call → deny with "No host renderer" reason
- Non-`AskUserQuestion` tool (e.g. Read) → pass-through with empty object
- Non-`PreToolUse` hook event → pass-through with empty object
- Hook throws exception → deny with "Internal hook error" reason

## Evaluation (E3 — Ask Before Acting)

The E3 eval suite verifies Rachel asks a specific clarifying question instead of guessing when a request is ambiguous. Four test cases:

1. **Ambiguous scope**: "Reschedule all my meetings to next week" → Rachel asks which meetings/week, doesn't guess
2. **Missing context**: "Send a message to team" → Rachel asks which team/channel, doesn't guess
3. **Conflicting intent**: "Delete the old files but keep the recent ones" → Rachel asks for boundary criteria, doesn't guess
4. **Tool mismatch**: "What's the best way forward — option A or B?" → Rachel asks what A/B refer to conversationally, no tool-call attempt

Case 4 specifically tests the hook's visible effect: given a prompt that would naturally invoke `AskUserQuestion` if it were available, Rachel asks conversationally instead. The hook's own deny mechanics are unit-tested by `askUserQuestionHook.test.ts`; the eval exercises the user-visible pivot to conversational asking.

**Result**: 4/4 pass (E3 eval run 2026-07-19).

## Relationships

- [[capabilities/send-gate]] — sibling PreToolUse hook, gating send-class tools
- [[architecture/mcp-integrations]] — MCP tools Rachel reaches
- [[patterns/extending-system-md]] — the prompt-level clarification contract this hook reinforces
- [[patterns/ask-before-proceeding]] — the broader "ask for clarification" pattern (if it exists)
