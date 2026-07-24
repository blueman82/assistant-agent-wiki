---
title: "Cross-platform persistent memory for Rachel (PRs #49-#52)"
type: source
created: 2026-07-21
last_updated: 2026-07-24
sources: ["prompts/system.md", "proactive/memoryIndex.ts", "proactive/sessionPersist.ts", "rachel.ts", "bridge/telegram-bridge.ts", "bridge/launchd.plist"]
tags: [memory, session-persistence, telegram, bridge, cross-platform, sdk-env]
---

## What shipped

Four merged PRs, one cluster, two independent mechanisms:

1. **A fact-based memory store** (PRs #49, #50) — works on every surface (terminal, Telegram, headless one-shots).
2. **Bridge-only session persistence** (PRs #51, #52) — Telegram conversation continuity across a bridge process restart only.

These are deliberately separate systems, not one unified mechanism — see [[investigations/2026-07-21-rejected-shared-session-thread]] for why a single shared-thread design was considered and rejected.

## 1. Memory store (PR #49 — `a002085`, PR #50 — `7be07d2`)

`prompts/system.md` gained a `## Memory` section: the behavioural contract. Store at `~/.rachel/memory/`, one fact per markdown file, frontmatter `name` (kebab-case slug), `description` (one line, used for recall relevance), `type` (`preference | decision | ongoing | reference`). Pointer-only index at `~/.rachel/memory/MEMORY.md`, format `- [Title](file.md) — hook`. Write triggers: stated preference, decision, deadline commitment, or correction — but a time-bound action item is a task in `tasks/`, not a memory. Recall: consult the index whenever a request touches remembered ground. Update-over-duplicate, delete-when-wrong, self-maintenance once the index passes ~50 entries. Explicit non-goal, stated because a prior task file got this wrong and it cost investigation time: **`.remember/` belongs to a separate Claude Code plugin, not Rachel — never read or write it.**

`proactive/memoryIndex.ts` (new, PR #50) makes recall deterministic rather than prompt-hoped. `composeSystemPrompt(basePrompt, memoryPath)` reads the index and appends it to the system prompt; wired into `rachel.ts` at the `agents.rachel.prompt` field (`rachel.ts:246`): `prompt: composeSystemPrompt(systemPrompt, resolveMemoryPath())`. This runs on every turn, so the index is composed fresh each time rather than loaded once at startup. `resolveMemoryPath()` reads a `RACHEL_MEMORY_PATH` env seam (mirrors the existing `RACHEL_AUDIT_LOG_PATH` idiom), falling back to `~/.rachel/memory/MEMORY.md`. Absent-is-empty contract matching `proactive/push.ts`'s `readJson`: only `ENOENT` means "no memories yet" and returns the base prompt unchanged; any other errno (corrupt file, EACCES, EISDIR) throws loud, naming the path.

**Size guard (PR #51).** `composeSystemPrompt` caps the injected index at 32 KiB. Over that, it truncates to a head slice plus an explicit marker (`[MEMORY.md truncated at 32768 bytes — consolidate the index...]`) — never a silent drop, so the agent can tell truncation happened and knows to self-consolidate per the prompt contract. Also an early return for an empty or whitespace-only `MEMORY.md` (no trailing-whitespace artifact in the prompt).

**UTF-8 truncation boundary fix (PR #52).** The first truncation cut was a raw byte slice (`Buffer.subarray(0, MAX_INDEX_BYTES)`), which can land mid-character — the operator's writing style uses em dashes and accented names routinely — producing a `U+FFFD` replacement character in the truncated tail. Fixed with a backward scan over UTF-8 continuation bytes (`(buf[cut] & 0xc0) === 0x80`) from the cut point, so the slice always lands on a character boundary. Regression test pins an em dash (3-byte UTF-8) straddling exactly the 32 KiB boundary.

> ⚠️ **Superseded (2026-07-24, PR #62, `0ee4c28`)**: the head-slice truncation direction described in the two paragraphs above is no longer current. Because `MEMORY.md` is append-ordered, keeping a head slice silently evicted the *newest* memories on every truncation — the opposite of what PR #51/#52 intended. PR #62 flips this to a tail-keep (and the UTF-8 boundary scan now advances forward, not backward, since the kept slice is the end of the buffer). The marker text also changed. See [[capabilities/memory]] for the corrected, current description and [[sources/2026-07-24-memory-hardening-cluster]] for the full fix.

## 2. Bridge session persistence (PR #51 — `7f8d2a4`, PR #52 — `f38c074`)

Narrow, bridge-only fix for a real bug: a launchd bridge restart wiped the Telegram conversation thread mid-conversation, because `rachel.ts`'s `sessionId` was module-scoped and never survived a process restart.

`proactive/sessionPersist.ts` (new) — pure `readSession`/`writeSession`/`clearSession` functions, atomic temp-file+rename matching `push.ts`'s write idiom. Wired into `rachel.ts` behind a `RACHEL_SESSION_FILE` env seam, set **only** in `bridge/launchd.plist`:
- Written on every session-id capture (the SDK `init` message branch).
- Read at startup via exported `hydratePersistedSession()`, called only from `bridge/telegram-bridge.ts`'s CLI guard (`bridge/telegram-bridge.ts:948`) — never a module-load side effect, so the CLI/terminal REPL and all headless one-shots stay inert by construction (seam unset = today's exact behaviour, byte-for-byte, pinned by a dedicated test).
- `resetSession()` also unlinks the persisted file, so `/reset` followed by a bridge restart doesn't resurrect the just-reset session.

Session file location: `<repo>/.rachel/bridge-session.json` (repo-local, gitignored) — **not** `~/.rachel/bridge-session.json` as the originating task spec literally named. Deviation flagged explicitly by the implementer: launchd's `EnvironmentVariables` can't expand `~`, and stamping `$HOME` would have required extending `scripts/install.sh`'s placeholder-substitution loop plus its test suite. Repo-local matches the existing convention already used by this same plist's `StandardOutPath`/`StandardErrorPath`.

### The one-writer invariant hole (PR #52's real content)

PR #51 documented `RACHEL_SESSION_FILE` as "exactly one writer" (the bridge) to keep the SDK's parentUuid transcript chain from forking under concurrent resume (see [[investigations/2026-07-21-rejected-shared-session-thread]]). Post-merge review found this was true as a *static* claim but false at *runtime*: `rachel.ts` gated the session write purely on the env var being **set**. The bridge's plist sets no `RACHEL_ALLOWED_TOOLS`, so bridge turns run with unrestricted Bash — and a Bash-spawned child (a nested `bin/rachel "..."` one-shot, an established pattern per `prompts/system.md`) inherits `RACHEL_SESSION_FILE` via ordinary process env inheritance. That child would capture its own session id under the same path and silently clobber the bridge's live session pointer — the next bridge restart would resume the wrong session.

Fix (`rachel.ts` ~line 263): when the seam is active, `options.env` is set to a spread of `process.env` with `RACHEL_SESSION_FILE` deleted, so the SDK subprocess (and anything *it* spawns via Bash) never sees the variable. Critical implementation detail, load-bearing: the SDK's `sdk.d.ts` documents `Options.env` as **replacing** the subprocess env entirely, not merging with it — spreading `process.env` first is mandatory, or the subprocess loses `PATH`/`HOME`/everything else and Bash breaks. When the seam is unset (CLI, all headless one-shots), `options.env` is left untouched entirely, so the SDK keeps managing subprocess env exactly as before — pinned by a dedicated test asserting `options.env` stays `undefined` in that case.

## Verified state

436 tests, 436 pass, 0 fail; `npm run typecheck` clean. All four launchd services (bridge + 3 scheduled jobs) redeployed onto merged code and verified live by the orchestrating loop (not re-verified independently by this ingest).

## Relationships

- [[investigations/2026-07-21-rejected-shared-session-thread]] — why one unified shared-thread design was rejected in favour of these two separate mechanisms
- [[capabilities/memory]] — the memory store as a capability page
- [[capabilities/telegram-frontend]] — bridge session continuity note
- [[architecture/overview]] — session model correction (bridge-restart survival is now a documented exception)
- [[architecture/mcp-integrations]] — n/a, not touched by this cluster
