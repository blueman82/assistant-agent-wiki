---
title: "PR #59 — permissionMode flipped to bypassPermissions"
type: source
created: 2026-07-23
last_updated: 2026-07-24
sources: ["rachel.ts", "gate/sendGate.ts", "node_modules/@anthropic-ai/claude-agent-sdk/sdk.d.ts", "node_modules/@anthropic-ai/claude-agent-sdk/sdk.mjs", "https://github.com/blueman82/assistant-agent/pull/59"]
tags: [source, security, gate, sdk, permissions]
---

## What changed

One line in `rachel.ts`'s per-turn `options` (in `runTurn`): `permissionMode: "auto"` → `permissionMode: "bypassPermissions"`. Merged as PR #59 (merge commit `30d6d49`, `origin/main` HEAD as of 2026-07-23). Diff was one insertion, one deletion — nothing else. In particular the diff added **no** `allowDangerouslySkipPermissions: true` and **no** send-gate test.

## Why it matters — the send gate

The change touches the one line the [[capabilities/send-gate]] page had explicitly called out as *not* a security boundary. The question this raises: with permissions bypassed, does the deterministic [[capabilities/send-gate]] `PreToolUse` hook still fire and still block Slack/Calendar sends?

**Answer: yes, per SDK type docs — the gate still fires.** (verified from source; not empirically spiked — see caveat.) `bypassPermissions` acts on the `canUseTool` permission-check path. PreToolUse hook denies are a separate path: `sdk.d.ts:4128` states, of the `PermissionDenied` event, *"PreToolUse hook denies bypass canUseTool and are not covered here."* So the send gate's `permissionDecision: "deny"` is honoured regardless of `permissionMode` — the mode flip changes which tool calls get an auto-*allow*, never whether a hook's *deny* is respected. This matches the send-gate page's long-standing note that `permissionMode` "is a model-classifier mode, not a security boundary — the gate does not rely on it." The class of mode changed (`auto` → `bypassPermissions`); the gate's independence from it did not.

**Caveat (register).** This is sourced from the SDK's type-doc comments, not from an empirical spike of the `bypassPermissions` + PreToolUse-deny combination. AGENTS.md's own record shows this SDK's hook semantics are version-dependent and have flipped once across a minor version (the fail-open-on-throw / fail-closed-on-timeout split, spiked on 0.3.216). A behavioural spike of this exact combination has not been run.

## Open question — the missing required flag

`sdk.d.ts:1721-1724` documents `allowDangerouslySkipPermissions` as: *"Must be set to `true` when using `permissionMode: 'bypassPermissions'`. This is a safety measure to ensure intentional bypassing of permissions."* PR #59 set the mode but not the flag.

Traced into the compiled SDK (`sdk.mjs`): the two are **independent conditionals** in the CLI-arg builder — `if(y)H.push("--permission-mode",y); if(b)H.push("--allow-dangerously-skip-permissions")`. The SDK layer does **not** throw when the mode is set without the flag; it simply omits `--allow-dangerously-skip-permissions` and still passes `--permission-mode bypassPermissions` to the spawned `claude` CLI binary. So:

- The change is **not** a no-op at the SDK boundary (the SDK doesn't reject it).
- Whether the downstream `claude` CLI binary honours `bypassPermissions` **without** the accompanying flag — silently proceeds, warns, or refuses — is **not determinable from readable source**: that enforcement, if any, lives in the bundled CLI binary, not in `sdk.mjs`. (unverifiable from source: bundled binary; needs a live behavioural spike.)

Net: the mode flip reaches the CLI, but the "Must be set" contract from the type docs is unsatisfied, and its runtime consequence is unconfirmed. This is a candidate PR defect worth a follow-up, not a settled improvement.

## Impact

- The draft-first send gate is **not** silently defeated by this change (per SDK docs).
- `capabilities/send-gate.md`'s stale line asserting `permissionMode: "auto"` was corrected in this ingest.
- Follow-up recommended: (1) add `allowDangerouslySkipPermissions: true` to satisfy the documented contract, or confirm empirically the CLI honours the mode without it; (2) add a send-gate test that asserts a gated tool is still denied under `permissionMode: "bypassPermissions"`, closing the "no test added" gap.

## Relationships

- [[capabilities/send-gate]] — the hook this change stress-tests; its `permissionMode` note updated by this ingest
- [[architecture/overview]] — `runTurn`'s per-turn `options` object is where the flag lives
- [[sources/2026-07-23-rejection-rca-and-fix-list]] — **the symptom this PR fixed, diagnosed after the fact.** Under `permissionMode: "auto"` a headless bridge turn raised permission prompts nobody could answer, auto-denying with `"The user doesn't want to take this action right now. STOP…"` (RCA mechanism B) — no human involved. The RCA also records the deploy lag: this PR merged 16:55 but the launchd bridge kept running pre-#59 code until its 22:56 restart, so the fix sat dead for six hours. **The RCA does not resolve this page's open `allowDangerouslySkipPermissions` question** — that remains outstanding. Its fix-list item 14 adds a related concern: under bypass, `gate/bashPatterns.ts` is the only send enforcement left in bridge turns, and it has known holes.
