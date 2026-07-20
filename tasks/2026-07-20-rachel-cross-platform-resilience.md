---
title: "Rachel: cross-platform resilience"
slug: rachel-cross-platform-resilience
status: backlog
priority: medium
repo: "[assistant-agent]"
due: null
---

## Overview

Improve Rachel's resilience across CLI (terminal) and Telegram transports. Two sub-tasks: (1) persistent memory system, (2) stream-event for Telegram responses.

## Part 1: Cross-Platform Persistent Memory

### Problem (verified from tasks_to_do.md line 75)

Rachel's conversations on Telegram or via CLI need a persistent memory system, not session-bound history.

### Scope

- Current state: conversation context lives in session memory only
- Goal: persistent memory accessible across sessions, both transports
- Storage: yaml/md files in ~/.rachel/memory/ or assistant-agent-wiki/memory/
- Query: Rachel can read/write/search across CLI and Telegram turns

### Acceptance Criteria

- Persistent memory directory created (~/.rachel/memory/ or wiki-based)
- Memory structure: per-date or per-session files (same format as .remember/)
- Rachel reads memory on startup (both CLI and Telegram bridge)
- Cross-session continuity verified: ask question in session 1 (CLI), reference in session 2 (Telegram)

## Part 2: Stream-Event for Telegram Responses

### Problem (verified from tasks_to_do.md line 77)

Telegram responses should use stream-event to avoid overwhelming API and give streamed updates. Current: one big text dump at end.

### Scope

- Implement streaming via Telegram bot API (chunked messages, with minimal delay to avoid rate limits)
- Rachel knows transport context (terminal vs. Telegram)
- Terminal: single response (no change)
- Telegram: streamed updates (text for text input, voice for voice input — inferred)

### Acceptance Criteria

- Stream-event integrated into Telegram bridge (bridge/telegram-bridge.ts)
- Responses chunked and sent incrementally
- No API rate limit hits observed
- Gary receives updates in real-time (not end-of-turn dump)

## Dependencies

- None (independent improvements)

## Notes

Original task (lines 73-78 of tasks_to_do.md): "macOS timeout issue", "rachels converasations on telegram or via cli, we need a cross-platform memory persistent system", "rachels telegram conversations need to use stream-event... rachel needs to know where shes talking and receiving responses from, terminal or telegram"

macOS timeout issue (line 73) is a separate item — add as sub-note or separate task if needed.
