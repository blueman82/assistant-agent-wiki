---
title: "Voice Pipeline Architecture"
type: architecture
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - bridge/speech.ts
  - bridge/api.ts
  - bridge/telegram-bridge.ts
  - scripts/speech/setup.sh
  - scripts/speech/transcribe.py
  - scripts/speech/synthesize.py
tags:
  - voice
  - stt
  - tts
  - mlx-whisper
  - mlx-audio
  - kokoro
  - huggingface
  - subprocess
  - timeout
---

## Overview

Rachel receives and sends voice via Telegram using local STT/TTS (no cloud speech API). `bridge/speech.ts` shells out to a dedicated Python 3.12 venv at `~/.rachel/venvs/speech` via `execFile` with injected timeouts, keeping audio processing off the Node event loop and in a separate process lifecycle.

## Three operations

| Operation | Tool | Timeout | Exit on | Fallback |
|-----------|------|---------|---------|----------|
| Transcribe | `mlx-whisper` STT | 120s (2 min, raised from 30s in PR #69) | nonzero exit / timeout | user-facing error, turn continues |
| Synthesize | `mlx-audio/Kokoro` TTS | length-scaled (see below) | nonzero exit / timeout | send text instead (PR #42+#55) |
| Convert | `ffmpeg` WAV→OGG/Opus | 15s | nonzero exit / timeout | send text instead |

Each operation is a separate `execFile` call (three independent child processes per voice reply). Each throws on failure; callers decide the fallback — tests can stub any of the three independently.

## HuggingFace Hub stall fix (PR #69)

**Problem:** `mlx-whisper` and `mlx-audio` perform a HuggingFace Hub freshness check on every call, even when the model is fully cached locally. Measured: 12.2s with check vs 4.3s without. When HF was unreachable, the bridge stalled past 45s, hitting the 30s timeout — live evidence: five `transcribe failed … signal=SIGTERM` entries in bridge logs.

**Solution:** `HF_HUB_OFFLINE=1` env var tells `huggingface_hub` to resolve from the local cache only (no network round-trip on the hot path). Implemented in two parts.

1. **Pass the env var to child processes.** New `hfOfflineEnv()` helper in `bridge/speech.ts` spreads `process.env` first, then adds `HF_HUB_OFFLINE: "1"`. **Critical trap:** Node's `execFile` **replaces** the child environment rather than merging it — a bare `{ HF_HUB_OFFLINE: "1" }` would strip `HOME`, and offline mode + no cache path becomes a hard failure. Two tests in `bridge/speech.test.ts` pin this directly.

2. **Pre-fetch models at setup time.** `scripts/speech/setup.sh` (PR #69) calls `huggingface_hub`'s `snapshot_download` for both models:
   - `mlx-community/whisper-small.en-mlx` (inbound STT)
   - `mlx-community/Kokoro-82M-bf16` (outbound TTS)
   
   Both model IDs are cross-checked against the actual repo refs in `transcribe.py` / `synthesize.py` — a typo there would make this prefetch inert. After download, both are re-resolved under `HF_HUB_OFFLINE=1` as a safety net: so a half-completed download becomes a setup-time failure, not a runtime one.

**Trade-off:** This approach (offline mode + pre-fetch) was chosen over passing a local snapshot path instead, per the RCA's decision. Setup must run once on any machine running the bridge. No production impact if models are already cached; setup is the safety net against a cold cache + offline mode becoming a silent failure at voice time.

## Synthesis timeout scaling (PR #55)

**Problem:** Synthesis was a flat 20s timeout. A 9,696-character reply takes ~30s to synthesize with Kokoro (measured: 9800 chars → 25s, 9696 chars prose → 30s), so `execFile` killed the child at 20s and the reply fell back to text silently.

**Solution:** `synthesizeTimeoutMs(textLength)` computes a **length-scaled budget**:
- 30s floor (cold model load is ~6s, with headroom)
- Plus 10ms/char (roughly 3x the measured 3.1ms/char worst case to be conservative)
- Capped at 300s so a runaway reply can't hang the bridge forever

Scaling was chosen over a bigger constant deliberately — a flat constant just relocates the cliff; scaling removes it entirely (up to a reasonable cap).

The function is exported so callers and tests assert the budget for a given length rather than pinning a constant. Tests use stubs; the real end-to-end eval with a 9,696-char reply (frozen eval `e1_...`) runs against the real venv and is run separately from `npm test` (grep guard forbids `bridge/speech.test.ts` from spawning real subprocesses).

**Additional fix (same PR):** `mlx-audio`'s `generate_audio` catches every exception, prints the reason to stdout (not stderr), then returns normally without writing a file. `synthesize()` originally read only stderr, so the reason was discarded. Now captures child stdout and includes it in the thrown error, so synthesis failure logs the actual diagnostic instead of a bare progress bar.

## Command-line argument surface

All three `.py` scripts accept audio/text paths as command-line arguments, keeping the contract simple:

```bash
python transcribe.py input.ogg                    # → transcribed text to stdout
python synthesize.py "text reply" output.wav      # → output.wav written
ffmpeg -i input.wav -c:a libopus output.ogg       # (no Python wrapper)
```

Text is passed as a positional arg to `synthesize.py`, which means `execFile` must shell-escape it. If a voice reply contains a character that breaks shell quoting, the error will manifest as a synthesis failure. This is a known seam — it's internal to the bridge and never exposed to user input directly (the reply text is Rachel's own model output, always serialised safely before passing to the command line). The command is constructed in `bridge/speech.ts`'s `synthesize()` function.

## Process lifecycle and cleanup

Each `.wav` and `.ogg` intermediate is written to `~/.rachel/tmp/<timestamp>.<ext>`. After the send completes (success or failure), there's a `finally` block that best-effort unlinks both files — an unlink failure is logged but not fatal. The `proactive/sweep.ts` family `tmp-sweep` (PR #71) runs every 30 minutes and deletes files >1 hour old, as a safety net for any intermediates that escape the per-turn `finally` blocks (e.g. a process killed mid-synthesis).

## Dependency: Python venv setup

The speech directory (`scripts/speech/`) contains:
- `setup.sh` — bootstraps `~/.rachel/venvs/speech` with mlx-whisper, mlx-audio, and ffmpeg (optional, can be pre-installed)
- `transcribe.py` — STT subprocess
- `synthesize.py` — TTS subprocess

`scripts/install.sh` (the one-command installer) **does not** run setup.sh automatically — that's a separate step because the venv setup involves real model downloads and Python package compilation, which can take 5–10 minutes depending on the host. The bridge will fail clearly at voice time if the venv doesn't exist, rather than silently skipping voice; this makes the missing setup detectable.

## Architecture decisions recorded elsewhere

- **Why local STT/TTS instead of cloud:** no API key dependency, lower latency, operator's audio never leaves the machine.
- **Why no TTS-specific duration cap before PR #55:** unforced error; a pre-discovery guess should have been tested against real reply lengths.
- **Why voice-origin turns always answer in voice (PR #42):** removed an invented 1000-char cap that was silently turning long replies back to text with no indication why; Gary confirmed real >1000-char voice replies work.
- **Why unsplit long voice replies (PR #42 residual):** no multi-message fallback for replies that synthesise to oversized OGG; still theoretical rather than observed.
- **Why `HF_HUB_OFFLINE` instead of snapshot paths:** decision recorded in [[sources/2026-07-24-rca-telebot-hardening]] (RCA items 1-4).

## Related pages

- [[capabilities/telegram-frontend]] — the full Telegram front-end architecture, including voice integration
- [[sources/2026-07-22-pr55-voice-synthesis-timeout]] — earlier fix for the synthesis timeout before scaling was added
- [[investigations/2026-07-18-telegram-voice-stt-tts-spec-gaps]] — pre-build spec gaps discovery
- [[sources/2026-07-24-rca-telebot-hardening]] — RCA items 1-4 (this architecture's root cause and fix list)
