---
title: "2026-07-23 RCA: Telegram failures hardening cluster"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - /Users/harrison/Github/assistant-agent
  - /Users/harrison/Github/coderails
tags:
  - rca
  - voice
  - send-gate
  - proactive
  - security
  - telegram
---

## Overview

Six merged PRs implement the 2026-07-23 RCA response: hardening Rachel's Telegram voice pipeline, send-gate enforcement, rejection handling, infrastructure cleanup, and cross-repo secret-file protection.

**PR status:** #69, #70, #71, #72, #76 merged to assistant-agent/main; coderails #298 merged to coderails/main. All merged 2026-07-24.

**RCA source:** `docs/coderails/specs/rca-2026-07-23-fix-list.md` (private; root causes and 19-item fix list with three states — closed, specced, decided-not-built).

## RCA item 1-4: Voice pipeline HF_HUB_OFFLINE (PR #69)

**Problem:** `mlx-whisper` and `mlx-audio` perform a HuggingFace Hub freshness check on every call, even with fully cached models. Measured: 12.2s with check vs 4.3s without. When HF was unreachable, it hung past 45s before the bridge's 30s `execFile` timeout SIGTERMed it — live log evidence: five `transcribe failed … signal=SIGTERM` entries.

**Fix:** Three parts.
- (A1/A2) `HF_HUB_OFFLINE=1` in `transcribe()` and `synthesize()` child env via a new `hfOfflineEnv()` helper. Node's `execFile` **replaces** the child env (doesn't merge), so must spread `process.env` first — otherwise `HOME` is stripped and offline mode + no cache path becomes a hard failure. Two tests pin this directly.
- (A3) `scripts/speech/setup.sh` pre-fetches both models (`mlx-community/whisper-small.en-mlx` and `mlx-community/Kokoro-82M-bf16`, cross-checked against `.py` files) and verifies both under `HF_HUB_OFFLINE=1` — so a cold cache becomes a setup-time failure, not a runtime one.
- (A4) Transcribe timeout: 30s → 120s (Gary's call). Humans may legitimately send longer voice notes, so flat 30s was a real constraint.

**Scope:** `bridge/speech.ts`, `bridge/speech.test.ts`, `scripts/speech/setup.sh`. Six new tests, built red→green.

**Deployed:** No. Bridge needs a restart to pick up the code change (launchd service `com.rachel.telegram-bridge`).

## RCA item 14: Send-gate bashPatterns coverage (PR #70)

**Problem:** Five escape holes in `gate/bashPatterns.ts`, the only send enforcement that runs inside bridge turns (the send-gate `PreToolUse` hook doesn't run under `bypassPermissions` mode; `bashPatterns` is the fallback).

| Hole | Fix | Positive test | Near-miss control |
|---|---|---|---|
| curl `--data` / `--data-raw` / `--data-binary` / `--data-urlencode` evaded the calendar detector (curl infers POST from body flag, pattern matched `-X POST` only) | `POST_METHOD_PATTERN` also accepts `--data` | 4 tests: all `--data` variants → `true` | `curl --dump-header /dev/null` → `false` |
| Short `-d` flag absent, including attached-value forms (`-d@file`, `-d'{...}'`, `-dkey=value`) | `SHORT_DATA_FLAG_PATTERN = /(^\\|\\s)-d/` (anchored to word-start, captures everything after regardless of attachment) | 4 tests: spaced, attached-file, attached-JSON, attached-pairs → `true` | `curl -D headers.txt` → `false` (case-sensitive anchor distinguishes `-D` dump-header) |
| Telegram `sendVoice` and `sendDocument` absent (`sendMessage` was covered) | `send(Message\\|Voice\\|Document)\\b` | Both → `true` | `.../bot123:ABC/getUpdates` → `false` |
| Slack `chat.update` and `chat.delete` absent | `chat\\.(postMessage\\|update\\|delete)\\b` | Both → `true` | `chat.getPermalink` → `false` |
| Gmail `drafts/send` absent (a draft could be sent without touching `messages/send`) | new pattern `/gmail/v1/users/[^/]+/drafts/send\\b` | → `true` | `.../users/me/drafts` (list) → `false` |

**Verification:** 18 new tests, all watched failing with red (false !== true) before the pattern existed. Mutation-tested: each pattern reverted individually, all six mutants killed, zero survivors.

**Scope:** `gate/bashPatterns.ts`, `gate/bashPatterns.test.ts`, `.evals/pr70-evals.json`. Total: 568 tests passing (baseline 550 + 18 new).

## RCA item 16: Stale ~/.rachel/tmp sweep (PR #71)

**Problem:** 750+ voice artifacts from test runs and killed processes (mid-synthesis termination leaves files behind). Bridge's own per-turn cleanup is correct; recurring sweep needed for long-lived service.

**Fix:** New 7th family `tmp-sweep` in `proactive/sweep.ts`. Runs every 30 minutes (existing tick), not on startup.
- Non-recursive: single `readdirSync` on the one directory only.
- `lstat` + `isFile()` rejects directories and symlinks in one check (symlink never followed).
- Age threshold: 1 hour (`TMP_MAX_AGE_MS = 60 * 60_000`). Longest legitimate file lifetime is the 10-min turn deadline + synthesis ceiling (5 min), so 1 hour is 6x that and still prevents days-long accumulation.
- Each `unlink` individually wrapped: one unremovable file is logged and sweep continues.
- Deliberately silent: no `push()` call, so routine hygiene never consumes the daily interrupt budget.

**Scope:** `proactive/sweep.ts`, `proactive/sweep.test.ts`, `CLAUDE.md` (one-line update: six families → seven). Total: 560 tests passing (baseline 550 + 10 new).

## RCA items 5-9: Ghost-rejection hardening (PR #72)

**Problem:** Strings like "The user doesn't want to proceed with this tool use" appeared in transcripts with no human involvement. Two mechanisms:
- **Mechanism A (turn-death ghost):** Bridge aborts every turn at 10 minutes. The harness injects rejection wording as the turn dies. Sign: co-occurs with `[Request interrupted by user for tool use]` and the turn ends. Proof: message 22:20:19, abort 22:30:19.589 — exactly 600 seconds.
- **Mechanism B (unanswerable permission prompt):** Headless process with nobody to answer, fixed by PR #59. Pre-existing, not new.

**Fix:** Five parts.
- (Item 5) `prompts/system.md` new "Machine-generated rejection strings" section documenting both strings with distinguishing tells. Three rules: check elapsed time; never attribute to operator; never promise report-backs from in-turn background work.
- (Item 6) `bridge/telegram-bridge.ts` abort inoculation: after deadline abort, next turn's input gets a `[bridge note]` explaining the prior turn was auto-aborted. Consumed by exactly the next turn; cleared by `/reset`.
- (Item 7) `prompts/system.md` turn budgeting: split long investigations across turns; go detached past ~8 min; prefer narrow tool calls when turn is already long.
- (Item 8) `bridge/telegram-bridge.ts` turn-start logging: `turn started (queue depth N)` — critical for RCA work (only completion/abort were logged before).
- (Item 9) `bridge/telegram-bridge.ts` synthesis-failure log truncation: cap error details at 300 chars (synthesis failure echoes 9,696-char private reply in its error message).

**Scope:** `prompts/system.ts`, `bridge/telegram-bridge.ts`, `bridge/telegram-bridge.test.ts`. Total: 558 tests (baseline 550 + 8 new).

**Deployed:** No. Bridge needs a restart.

## Wake channel consumer (PR #76)

**Problem:** Detached work has no way to start a Rachel turn. `runTurn`'s only caller is `drainFifo`, which wakes only on Telegram message. Background job that finishes reports into the void.

**Solution:** File-drop channel at `~/.rachel/wake/`. A producer writes `<id>.json`:
```json
{ "id": "adhoc-tunnel", "source": "adhoc:tunnel", "mode": "narrate",
  "severity": "normal", "message": "Tunnel task finished: 3 files changed.",
  "created_at": "2026-07-24T09:00:00Z" }
```

`checkWakeFiles()` scans the directory **once per `getUpdates` poll iteration** (not the 30-min sweep — sweep latency would defeat the point). Cost: zero Telegram API calls, ~30s latency.

Routing:
- `mode: "narrate"` — synthetic FIFO message `[wake: <source>] <message>`, then kick `drainFifo()`. Single-flight, so if a turn is running the kick no-ops; the message waits in FIFO and the existing drain loop picks it up when current turn ends.
- `mode: "fyi"` — through `pushAlert()` wrapper (family `wake`, inherits quiet hours/dedup/budget).

**Security guard:** Missing or invalid mode routes to FYI with `[untagged wake: <source>]` prefix. Only explicitly tagged `narrate` can start a billed Rachel turn. Unknown producers can reach Gary but never trigger SDK spend.

**At-most-once semantics:** Consumed file renamed `<id>.json` → `<id>.done` **before** dispatch (not after). Malformed JSON renamed to `.bad`. Flood cap: 5 files per iteration.

**Scope:** `bridge/telegram-bridge.ts`, `bridge/telegram-bridge.test.ts`. Total: 558 tests (baseline 550 + 8 new). Live eval: `WAKE EVAL PASS`.

**Status:** Inert (PR 2 of spec — consumers not yet written in PR 3).

## Coderails #298: .env secret file protection

**Problem:** `destructive_bash_gate.sh` had **zero** `.env` matches before this PR. This is load-bearing for Rachel's send-gate isolation — when Rachel runs Bash via the bridge, this hook is the fallback enforcement (the send-gate's `PreToolUse` hook doesn't run under `bypassPermissions` mode).

**Fix:** Command-agnostic path token match anywhere on line (no verb enumeration risk — every omitted reader/writer verb is a silent bypass). Allows `.env.example` / `.env.sample` / `.env.template` / `.env.dist` (committed placeholders, no real secrets). Denies `.env.local`, `.env.production`, `.env.local.bak` (real secret files).

**Why two branches, not one regex:** "Deny `.env.<suffix>` EXCEPT templates" requires negative lookahead, which POSIX ERE (bash 3.2, no PCRE) cannot express. The suffixed case extracts each token and tests its first-suffix-segment only, so `.env.example.md` (documentation) stays allowed while `.env.local.bak` (backup of secrets) denies.

**Verification:** 330 test checks, `PASS`. Mutation-tested: 8 mutants all killed, zero survivors. Each pattern covers non-overlapping test cases (M2 vs M1 prove isolation).

**Scope:** `hooks/scripts/destructive_bash_gate.sh`, `hooks/scripts/tests/destructive_bash_gate.test.sh`, docs. Repo: coderails (not assistant-agent).

**Why in assistant-agent-wiki:** Rachel's bridge invokes `destructive_bash_gate.sh` (per AGENTS.md line 102). This gap directly affects Rachel's isolation story when she runs Bash commands under `bypassPermissions` mode.

## Design patterns and architectural notes

- **HuggingFace Hub stall:** The offline-mode + pre-fetch pattern is generalizable to other model-loading paths that don't require fresh metadata.
- **Send-gate coverage:** Five holes were real escapes. The bashPatterns detector is command-agnostic (path-token-first matching) — this philosophy extends to other send patterns.
- **Proactive architecture:** tmp-sweep joins six existing families in the same 30-min tick. Each family gets its own try/catch; failure of one family doesn't break the tick — structural insurance per `runFamily` injection pattern.
- **Ghost rejection hardening:** Machine-generated vs. human-generated was a hard-to-separate signal. The 600s exact-deadline-match is the cleanest discriminator; turn-start logging is now mandatory for RCA work.
- **Wake channel:** File-drop beats polling for latency and cost. At-most-once semantics (rename-before-dispatch) prevent replay storms on the narrate path. The untagged-mode guard is the security property — unknown producers can't spend SDK budget.
- **Secret file protection:** Path-token matching (not verb enumeration) is future-proof. Template carve-out by first-suffix-segment keeps documentation pages reachable while real secrets stay denied.

## Known residuals and follow-ups

- **PR #69 bridge deployment:** Code merged, launchd service restart still pending. Setup script should be run on any machine running the bridge to pre-warm the HF cache.
- **PR #72 bridge deployment:** Code merged, launchd service restart still pending.
- **PR #76 producers:** This PR (consumer) lands inert. PR 3 will write the first producers (ad-hoc backgrounding task completion, loop-launcher task completion).
- **Send-gate under bypassPermissions:** PR #70 proves `PreToolUse` hook is honoured by the SDK. PR #70's own coverage does not include a synthetic test of bashPatterns + bypassPermissions together — that integration is live only.
- **Coderails #298 mention-safety:** Pattern ID is `dotenv-access`, not `.env` — the id contains no `.env` substring so `echo dotenv-access` cannot self-trigger the gate. Checked: three new lines all carry the pattern ID, not the path token.

## Cross-repo dependencies

- **assistant-agent → coderails:** Rachel's bridge invokes the `destructive_bash_gate.sh` hook. PR #70 documents why `bashPatterns.ts` is the fallback when the send-gate's PreToolUse hook doesn't run. PR #298 closes the `.env` gap in that fallback. Both PRs together complete the send-gate story for `bypassPermissions` mode.
