---
title: "Memory"
type: capability
created: 2026-07-21
last_updated: 2026-07-21
sources: ["prompts/system.md", "proactive/memoryIndex.ts", "proactive/memoryIndex.test.ts", "rachel.ts"]
tags: [capability, memory, persistence, deterministic-injection, cross-platform]
---

## What it does

Rachel has a persistent, file-based fact store at `~/.rachel/memory/`, shared across the terminal and Telegram since both run through the same `runTurn` (`rachel.ts`). One fact per markdown file:

```yaml
---
name: <short-kebab-case-slug>
description: <one line — used to judge relevance during recall>
type: preference | decision | ongoing | reference
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

**Size guard.** The injected index is capped at 32 KiB. Over that, `composeSystemPrompt` truncates to a head slice plus an explicit marker: `[MEMORY.md truncated at 32768 bytes — consolidate the index (see prompts/system.md's Memory contract).]` — never a silent drop, so the model can see truncation happened and knows to self-consolidate. An empty or whitespace-only `MEMORY.md` returns the base prompt unchanged (no trailing-whitespace artifact).

**UTF-8-safe truncation.** A raw byte cut at exactly 32 KiB can land mid-character (the operator's writing uses em dashes and accented names), which would otherwise produce a `U+FFFD` replacement character in the truncated tail. The cut point backs up over UTF-8 continuation bytes (`(byte & 0xc0) === 0x80`) until it lands on a character boundary, so truncation never corrupts a character at the boundary.

## Design note — why this is not session persistence

Memory and Telegram session continuity ([[capabilities/telegram-frontend]]'s bridge-restart note) are deliberately two separate mechanisms, not one. A single shared-conversation-thread design was considered and rejected — see [[decisions/2026-07-21-rejected-shared-session-thread]] for the three reasons, the decisive one being that SDK context compaction discards old transcript content, so a persisted session thread is not actually durable across time. The memory store lives outside the transcript entirely, which is what makes it survive compaction.

## Relationships

- [[sources/2026-07-21-cross-platform-persistent-memory]] — the PR cluster (#49, #50) that built this
- [[decisions/2026-07-21-rejected-shared-session-thread]] — why this is separate from session persistence
- [[architecture/overview]] — where `composeSystemPrompt` sits in the turn-construction pipeline
- [[capabilities/tasks]] — the other store; time-bound action items go there, not here
- [[capabilities/proactive-layer]] — `proactive/push.ts`'s `readJson` contract this module's error handling mirrors
