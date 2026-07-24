---
title: "Memory write gate: audit logging + error-swallowing catch fix (PR #68)"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources: ["gate/memoryGate.ts", "gate/memoryGate.test.ts", "gate/auditLog.ts", "rachel.ts"]
tags: [memory, security, audit, hardening]
---

## What this is

PR #68 (`https://github.com/blueman82/assistant-agent/pull/68`, merged `2026-07-24T01:07:41Z`) fixes two findings against `gate/memoryGate.ts`, the write-time memory gate PR #64 shipped (see [[sources/2026-07-24-memory-hardening-cluster]]). Both findings were caught by a code-reviewer subagent 25 minutes after #64 merged, and fixed in this follow-up PR rather than by amending #64.

**Framing, stated explicitly**: this is not a previously-documented gap being closed. [[sources/2026-07-24-memory-hardening-cluster]]'s own "known, explicitly unclosed gaps" section names exactly two open items — `mcp-exec` reachability and prompt-only lock routing — and mentions neither audit logging nor an error-swallowing catch anywhere. PR #68 fixes an omission the wiki never recorded, not a flagged gap flipping to closed.

## Fix 1 — no audit logging on any deny path

Every deny in `gate/memoryGate.ts` (untrusted-context lockout on `Write`/`Edit`, the same lockout on `Bash`, frontmatter-schema rejection, the internal-error catch-all) returned a decision to the SDK with zero durable record. The sibling gate, `gate/sendGate.ts`, audits every attempt and decision to `~/.rachel/send-gate-audit.jsonl` ([[capabilities/send-gate]]); the memory gate audited nothing, so a memory-write deny left no trace to investigate later.

`createMemoryGateHook` now takes a required `auditLogPath` parameter — the same `RACHEL_AUDIT_LOG_PATH`-resolved value `rachel.ts` already threads into `createSendGateHook`, so both gates' decisions land in one shared audit trail rather than two. A local `appendAudit()` helper (mirroring `sendGate.ts`'s own pattern — fail-silent, an audit-write failure must never affect the gate decision already returned) is called at every deny site, each tagged with a distinct `surface` label:

| Surface label | Deny site |
|---|---|
| `untrusted-lockout` | `Write`/`Edit` into the memory dir under `RACHEL_UNTRUSTED_CONTENT` |
| `untrusted-lockout-bash` | `Bash` command touching the memory dir under `RACHEL_UNTRUSTED_CONTENT` |
| `frontmatter-schema` | Frontmatter validation failure (missing/invalid required field) |
| `internal-error` | The hook's own outer catch |

A pass-through call (no deny) still writes no audit row at all — only decisions are logged, matching `sendGate.ts`'s own event model (`"attempt" | "decision"`).

## Fix 2 — outer catch blocks discarded the actual exception

Two `catch { ... }` blocks (the hook's own top-level catch, used elsewhere in the file) didn't bind the caught error — so a real failure, e.g. `EACCES`/`ELOOP` thrown by the path-resolution logic PR #64's three security rounds hardened (see [[sources/2026-07-24-memory-hardening-cluster]]), was discarded before the hook returned a generic `"Internal hook error"` message. The actual cause was unrecoverable after the fact.

`gate/auditLog.ts`'s `AuditRow` interface gained two new optional fields, `errorCode` and `errorMessage`, to carry this. The catch block now binds the error (`catch (err)`) and logs `err.code` / `err.message` (or `String(err)` for a non-`Error` throw) into the audit row before denying.

**Distinct from PR #64's Round 3 fix, same theme.** [[sources/2026-07-24-memory-hardening-cluster]]'s Round 3 fixed a different site — `resolveReal`'s internal catch, which was swallowing non-`ENOENT` errnos and silently falling back to a wrong lexical comparison (a security bypass: an unreadable ancestor could hide a symlink into the memory dir). This PR's fix is the hook's own **outer** catch, further up the call stack, which never had a bypass problem — it always denied correctly (fail-closed) — but discarded the diagnostic information about *why* it denied. Two separate catch sites, two separate bugs, one shared lesson: an unconditional `catch { }` with no bound identifier is a recurring shape worth grep-checking for elsewhere.

## A build-time regression, caught before merge

The PR body notes `toolName`/`hash` are captured as mutable `let`s, reassigned inside the `try`, specifically so a throw while reading a hostile/malformed `input` (e.g. a `Proxy` that throws on property access — exactly what the test suite's "hook throws exception" fixture constructs) still lands in the catch with safe `"unknown"` defaults, rather than the reassignment itself escaping the try/catch. The PR description states this was verified against a regression during development: an earlier version of the fix crashed on exactly this case. No further detail on the crash mode is available beyond the PR body's own account `(unverifiable: the intermediate crashing version was never itself committed to a branch, only described after the fact in the merged PR body — nothing to diff against)`.

## Verification (from the PR body, not independently re-run this ingest)

- `npm run typecheck`: clean.
- `npx tsx --test gate/memoryGate.test.ts`: 29/29 pass (25 existing + 4 new).
- `npm test` (full suite): 550/550 pass (up from the 546 baseline recorded in [[sources/2026-07-24-memory-hardening-cluster]]).
- Independently verified by a separate code-reviewer subagent (not self-graded), per the PR body: confirmed every `denyOutput` call site has an audit call immediately before it, confirmed the catch-block fix doesn't reintroduce the original crash on a throwing/hostile input, confirmed no other `createMemoryGateHook` callers were missed (repo-wide grep — `rachel.ts` is the only production caller), confirmed the audit row shape doesn't break any downstream consumer (none exist outside the writers and tests).

## Relationships

- [[sources/2026-07-24-memory-hardening-cluster]] — the PR #62-#65 cluster this PR follows up on; its PR #64 section and "known, explicitly unclosed gaps" list neither audit logging nor this catch bug, per the framing note above
- [[capabilities/memory]] — the write-time-gate section now notes the shared audit trail
- [[capabilities/send-gate]] — `gate/memoryGate.ts`'s sibling hook, whose audit pattern (`appendAudit`, fail-silent, `RACHEL_AUDIT_LOG_PATH`) this PR brings the memory gate into line with
