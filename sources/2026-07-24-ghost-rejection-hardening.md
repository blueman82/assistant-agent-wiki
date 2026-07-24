---
title: "Ghost-rejection hardening (RCA items 5-9)"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - bridge/telegram-bridge.ts
  - prompts/system.md
tags:
  - rca
  - rejection
  - turn-death
  - logging
  - error-handling
---

## Background

Strings like "The user doesn't want to proceed with this tool use" appeared in Rachel's transcripts with no human involvement — a machine-generated artifact, not an actual decision by Gary.

Two distinct mechanisms produced these strings:

**Mechanism A (turn-death ghost):** The bridge aborts every turn at 10 minutes (`DEFAULT_TURN_TIMEOUT_MS`, PR #46). As the turn dies, the harness injects rejection wording for the in-flight tool call. **Distinguishing tells:** the string co-occurs in the same sub-second with `[Request interrupted by user for tool use]` AND **the turn ends** (no continuation). Strongest proof: an RCA examined message timestamp 22:20:19 and abort timestamp 22:30:19.589 — exactly 600 seconds, matching the deadline with no variance.

**Mechanism B (unanswerable permission prompt):** A headless process (e.g., a detached `claude -p` spawn) had nobody to answer a permission prompt, so the permission layer denied the call with "The user doesn't want to take this action right now. STOP...". This mechanism was fixed by PR #59 (permissionMode flipped to `bypassPermissions`), documented separately in [[sources/2026-07-23-pr59-bypass-permissions]].

The 10-minute deadline itself (Mechanism A's root) was already decided as a fixed constraint — PR #72 does not change it, only inoculates against misreading its artifacts.

## What PR #72 fixes

Five parts, all aiming to stop misattributing machine-generated rejections to the human operator.

**Item 5 — `prompts/system.md`, new "Machine-generated rejection strings" section.** Documents both rejection strings, their contexts, and three distinguishing rules:
1. Always check elapsed time before attributing a rejection to anyone.
2. Never attribute either string to the operator.
3. Never promise report-backs from in-turn background work (structurally impossible — every turn is a fresh subprocess, turns only start from inbound messages, an aborted turn's reply text is replaced by the cutoff notice, so no later report can reach the session).

**Item 6 — abort inoculation, `bridge/telegram-bridge.ts`.** After a deadline abort, `drainFifo` prefixes the **next turn's input** with a `[bridge note]` explaining that the previous turn was auto-aborted and its rejection strings are artifacts. This is distinct from the existing `timedOut` branch (which pushes a user-facing message into the reply buffer to explain the event to Gary) — this one explains the event to the model, so it stops misreading the prior turn's residue. One-shot: consumed by exactly the next turn, and cleared by `/reset` (a fresh session holds no residue, so carrying the note across would assert an abort that left no trace).

**Item 7 — turn budgeting, `prompts/system.md`.** New instructions to split long investigations across turns, go detached past roughly 8 minutes, and prefer narrow tool calls when a turn is already long. States plainly that the 10-minute deadline is fixed and to work inside it.

**Item 8 — turn-start logging, `bridge/telegram-bridge.ts`.** New log line `turn started (queue depth N)` with the FIFO backlog at that moment. Before this PR, only turn completion or abort was logged, which is why the RCA work required painful JSONL cross-referencing to establish when a turn began (painful timestamps + offline reconstruction).

**Item 9 — synthesis-failure log truncation, `bridge/telegram-bridge.ts`.** One voice-reply failure wrote the full 9,696-character private reply into the log — `synthesize.py` takes reply text as an argv element, so an execFile timeout error echoes the whole reply back in its error message. New `truncateErrorDetail(text, 300)` caps error detail at 300 chars, keeps the leading diagnostic, and marks itself `[truncated, N chars total]` so a short message is never mistaken for a clipped one.

## Additional fix (found during review)

`/stop` command (PR #36) fires the same `AbortController` that `DEFAULT_TURN_TIMEOUT_MS` uses, and was never inoculated. It produced the identical "The user doesn't want to proceed" string when it was the operator explicitly stopping a turn, turning a legitimate operator action into an ambiguous ghost. PR #72 added the abort inoculation to `/stop`'s path as well, so both mechanisms (deadline and explicit stop) now prefix the next turn with the `[bridge note]` explaining that the turn was aborted.

## Verification

Built test-first for items 6, 8, 9 — 8 new tests, each watched failing before implementation. Item 9's red run reproduced the 12,125-char log-line leak exactly. `/reset` behaviour was additionally verified by reverting only that production line and confirming the test goes red again (not a timing artifact).

Items 5 and 7 are prose edits to `prompts/system.md` with no testable code — verified by inspection.

Two pre-existing tests asserted exact `runTurn` inputs after an abort to prove the queue kept draining. That assertion incidentally pinned input text, which item 6 now changes by design. Both tests were relaxed to match the operator's message (the semantics they cover) rather than exact equality — queue-drain semantics preserved, input text now expected to include the bridge note.

Final: `npm run typecheck` clean, `npm test` 558 passing (baseline 550 + 8 new).

## Related pages

- [[capabilities/telegram-frontend]] — turn timeout, duration logging, and ad-hoc backgrounding (PR #56) — context for the 10-min ceiling
- [[sources/2026-07-23-rejection-rca-and-fix-list]] — the parent RCA with findings and 19-item fix list (items 5-9 are this PR's scope)
- [[sources/2026-07-24-rca-telebot-hardening]] — cluster source page merging all six PRs
