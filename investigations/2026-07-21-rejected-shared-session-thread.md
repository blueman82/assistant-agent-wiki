---
title: "Rejected: one shared session thread across terminal and Telegram"
type: investigation
created: 2026-07-21
last_updated: 2026-07-21
status: rejected
sources:
  - rachel.ts
  - proactive/sessionPersist.ts
  - proactive/memoryIndex.ts
tags: [memory, session-persistence, design-decision, sdk-transcript, compaction]
---

# Rejected: one shared session thread across terminal and Telegram — 2026-07-21

While building cross-platform persistent memory ([[sources/2026-07-21-cross-platform-persistent-memory]]), a simpler-looking design was on the table: persist **one** `sessionId` and have both the terminal REPL and the Telegram bridge resume it, so the two surfaces share one continuous conversation thread. Rejected for three independent reasons — any one of them alone would have been enough.

## 1. Concurrent same-id resume is unsafe

SDK transcripts are a linked `parentUuid` chain — verified directly against a real transcript via `jq` (2934 chained entries). If two processes (terminal and bridge) both resume the same session id at the same time, each appends to the chain from its own last-seen point, forking it. There is no merge; one branch's history silently becomes unreachable from the other.

## 2. Decisive: compaction defeats the entire purpose

SDK context compaction discards old transcript content once a session grows past its window. A persisted, never-dying shared thread is **not durable memory** — a fact recorded three weeks ago is simply gone once compaction has run since. Wanting "Rachel remembers things across time" and implementing it as "keep one session alive forever" are incompatible: the second doesn't deliver the first. This is why the memory store (PR #49/#50) is a separate file-based fact store, not session continuity — durability has to survive compaction, and only a store outside the transcript can do that.

## 3. Headless one-shots would pollute the operator's personal thread

Roughly 8 automated headless one-shots run per day (`inbox-brief`, `proactive-calendar`, sweep-spawned calendar checks). Under a shared-thread design these would load the same persisted session id and pour their sweep output into the operator's own ongoing conversation — mixing automated noise into a thread meant for actual back-and-forth.

## What shipped instead

Two independent, narrowly-scoped mechanisms:

- **Memory store** (works everywhere: terminal, Telegram, one-shots) — durable facts in `~/.rachel/memory/`, survives compaction because it's outside the transcript entirely. See [[capabilities/memory]].
- **Bridge-only session persistence** (Telegram continuity only, across a bridge *process restart*, not a design for cross-surface sharing) — `RACHEL_SESSION_FILE`, set only in `bridge/launchd.plist`, exactly one writer by construction. See [[sources/2026-07-21-cross-platform-persistent-memory]] for the one-writer invariant and the runtime hole (PR #52) found and closed in it.

## Relationships

- [[sources/2026-07-21-cross-platform-persistent-memory]] — the cluster this decision came out of
- [[capabilities/memory]] — the store built as the actual durable-memory answer
- [[architecture/overview]] — session model, corrected for the bridge-restart exception
