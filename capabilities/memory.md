---
title: "Memory"
type: capability
created: 2026-07-21
last_updated: 2026-07-24
sources: ["prompts/system.md", "proactive/memoryIndex.ts", "proactive/memoryIndex.test.ts", "proactive/memoryLint.ts", "gate/memoryGate.ts", "gate/auditLog.ts", "rachel.ts", "proactive/memoryLock.ts", "proactive/memoryAppend.ts"]
tags: [capability, memory, persistence, deterministic-injection, cross-platform, write-gate, lint]
---

## What it does

Rachel has a persistent, file-based fact store at `~/.rachel/memory/`, shared across the terminal and Telegram since both run through the same `runTurn` (`rachel.ts`). One fact per markdown file:

```yaml
---
name: <short-kebab-case-slug>
description: <one line — used to judge relevance during recall>
type: preference | decision | ongoing | reference
date: <ISO 8601 — added by PR #63; missing date warns, doesn't error, since files predate the field>
---
```

A pointer-only index at `~/.rachel/memory/MEMORY.md`, one line per memory: `- [Title](file.md) — hook`. The index holds pointers only, never memory content — the model reads the index, then reads the specific file(s) that look relevant.

The directory and index are created on first write; an absent index means no memories yet, not an error.

## Behavioural contract (`prompts/system.md`'s `## Memory` section)

- **Write** when the operator states a preference, makes a decision, commits to something with a deadline, or corrects Rachel. A time-bound action item (something to *do* by a date) is a task in `tasks/`, not a memory — the two stores are for different things.
- **Recall**: consult the index whenever a request touches remembered ground.
- **Update over duplicate**: check for an existing file covering the same fact before writing; edit it rather than creating a near-duplicate.
- **Delete when wrong**: a memory contradicted by reality is removed, not kept stale.
- **Self-maintenance**: once the index passes roughly 50 entries, consolidate — merge overlapping facts, drop what's gone stale.
- **Non-goal, stated explicitly**: `.remember/` belongs to a separate Claude Code plugin, not Rachel — never read or write it. (A prior task file wrongly asserted `.remember/` was Rachel's memory system; that error cost investigation time before this capability existed.)

## Deterministic injection (`proactive/memoryIndex.ts`)

Recall is not left to the model noticing the file exists — `composeSystemPrompt(basePrompt, memoryPath)` reads the index and appends it to the system prompt on **every turn**, wired at `rachel.ts:246`: `prompt: composeSystemPrompt(systemPrompt, resolveMemoryPath())`. This makes the write/recall contract above a backstop on top of a deterministic floor: even if the model never thinks to check, the index is already in its prompt.

`resolveMemoryPath()` resolves a `RACHEL_MEMORY_PATH` env seam (mirrors the existing `RACHEL_AUDIT_LOG_PATH` idiom; unset in production, lets tests redirect reads away from the real store), falling back to `~/.rachel/memory/MEMORY.md`.

**Absent-is-empty, corruption-is-loud**: matching `proactive/push.ts`'s `readJson` contract, only `ENOENT` is treated as "no memories yet" (returns the base prompt unchanged). Any other error (corrupt file, `EACCES`, `EISDIR`) throws loud, naming the path — a real read failure must surface, not silently degrade to an unchanged prompt that hides a broken store.

**Size guard, tail-keep since PR #62 (`0ee4c28`).** The injected index is capped at 32 KiB. Over that, `composeSystemPrompt` truncates to the **tail** — not the head — plus an explicit marker. Current marker text: `[MEMORY.md truncated — older entries were dropped, keeping the most recent ${MAX_INDEX_BYTES} bytes; consolidate the index (see prompts/system.md's Memory contract).]` — never a silent drop, so the model can see truncation happened and knows to self-consolidate. An empty or whitespace-only `MEMORY.md` returns the base prompt unchanged (no trailing-whitespace artifact).

**Tail, not head — direction fix, PR #62.** `MEMORY.md` is append-ordered: new pointer lines land at the end, so the *oldest* entries sit at the head and the *newest* at the tail. The original implementation kept a head slice, which silently evicted the newest — statistically the most relevant — memories on every truncation. Now keeps the tail instead, preserving the leading `# Memory Index` heading when present. See [[sources/2026-07-24-memory-hardening-cluster]] for the fix's full detail, including a total-silent-wipe edge case (a single oversized line) that PR #62 also closed.

**UTF-8-safe truncation.** A raw byte cut at the boundary can land mid-character (the operator's writing uses em dashes and accented names), which would otherwise produce a `U+FFFD` replacement character in the truncated tail. Because the kept slice is now the tail, the cut point advances **forward** over UTF-8 continuation bytes (`(byte & 0xc0) === 0x80`) until it lands on a character boundary — the opposite direction from a head-keep's backward scan, not a mirror of it.

## Write-time enforcement (`gate/memoryGate.ts`, PR #64), periodic detection (`proactive/memoryLint.ts`, PR #63), and locked appends (`proactive/memoryLock.ts` + `proactive/memoryAppend.ts`, PR #65)

The write/recall contract above was, until PRs #62-#65, enforced by nothing but the prompt — verified empirically: the first-ever memory file had no frontmatter at all. Three code layers now back it (all merged, `main` HEAD `4d4ea4a`), deliberately divided so each covers a different gap:

- **Write-time gate**: a third `PreToolUse` hook in `rachel.ts` (alongside [[capabilities/send-gate]]'s hook) denies a `Write` inside the memory dir whose frontmatter fails validation (missing `name`/`description`/`type`, invalid `type`, or a `name`≠filename-slug mismatch), and denies *any* memory-dir `Write`/`Edit`/`Bash` outright when `RACHEL_UNTRUSTED_CONTENT` is set (currently only in `tasks/inbox-brief-launchd.plist`, since that one-shot processes hostile email while holding `Write`). Every deny is now audit-logged to the same `RACHEL_AUDIT_LOG_PATH` trail the send gate writes to, tagged with a `surface` label identifying which check denied (PR #68 — see [[sources/2026-07-24-memorygate-audit-logging]]); the hook's own outer catch also binds and logs the real thrown error instead of discarding it, a fix from the same PR.
- **Periodic lint**: the same schema validator (`validateFrontmatter`, shared — one implementation, two callers) runs as a 6th family in the 30-minute proactive sweep, catching anything the hook doesn't see: `Edit` results, obfuscated `Bash`, manual filesystem edits, or a detached `claude -p` spawn (which loads no hooks at all).
- **Locked appends**: `MEMORY.md` is read-modify-write across the terminal, the Telegram bridge, and headless one-shots concurrently; the repo's usual atomic temp-file+rename idiom gives durability but not mutual exclusion, so two near-simultaneous appends could silently drop one pointer line. `proactive/memoryAppend.ts` (invoked as `npx tsx proactive/memoryAppend.ts "<title>" "<file>" "<hook>"`, per `prompts/system.md`) wraps the append in an `O_EXCL` lockfile mutex (`proactive/memoryLock.ts`) and rejects (never sanitises) a title/file/hook containing a newline, carriage return, or `[ ] ( )` that could corrupt or forge a pointer line.

**Known, explicitly unclosed gaps** — do not read this capability as fully locked down even with all four PRs merged: (1) `rachel.ts` runs `permissionMode: "bypassPermissions"` and sets `allowedTools` but never the SDK's `tools` option, so `mcp__mcp-exec__execute_code_with_wrappers` remains reachable in the untrusted one-shot and no tool-name-based hook can cover it; (2) routing writes through the locked-append CLI is a **prompt contract, not code-enforced** — `prompts/system.md` says so explicitly — so a freehand `Write` to `MEMORY.md` still bypasses the lock entirely. Full detail, including the three security-fix rounds that found the write-gate's path-resolution bypasses one at a time: [[sources/2026-07-24-memory-hardening-cluster]].

## Design note — why this is not session persistence

Memory and Telegram session continuity ([[capabilities/telegram-frontend]]'s bridge-restart note) are deliberately two separate mechanisms, not one. A single shared-conversation-thread design was considered and rejected — see [[investigations/2026-07-21-rejected-shared-session-thread]] for the three reasons, the decisive one being that SDK context compaction discards old transcript content, so a persisted session thread is not actually durable across time. The memory store lives outside the transcript entirely, which is what makes it survive compaction.

## Relationships

- [[sources/2026-07-21-cross-platform-persistent-memory]] — the PR cluster (#49, #50) that built this
- [[sources/2026-07-24-memory-hardening-cluster]] — PRs #62-#65 (all merged): truncation direction fix, periodic lint, write-time gate, locked-append write lock
- [[sources/2026-07-24-memorygate-audit-logging]] — PR #68: audit logging on every write-gate deny path, plus a fix for a catch block that was discarding the real thrown error
- [[investigations/2026-07-21-rejected-shared-session-thread]] — why this is separate from session persistence
- [[architecture/overview]] — where `composeSystemPrompt` sits in the turn-construction pipeline
- [[capabilities/tasks]] — the other store; time-bound action items go there, not here
- [[capabilities/proactive-layer]] — `proactive/push.ts`'s `readJson` contract this module's error handling mirrors; also now runs the memory-lint sweep family
- [[capabilities/send-gate]] — the sibling `PreToolUse` hook `gate/memoryGate.ts` sits alongside
- [[capabilities/inbox-brief]] — sets `RACHEL_UNTRUSTED_CONTENT`, the trigger for the memory-write lockout
