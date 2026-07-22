---
title: "PR #55 — voice replies failed on long text: the flat 20s synthesis timeout"
type: source
origin: "assistant-agent PR #55 (merged 08f3022, 2026-07-22)"
date: 2026-07-22
tags: [source, telegram, voice, tts, bridge, timeout, rca]
---

## What happened

Gary sent a voice message. Rachel replied with **text**. The bridge had correctly
identified the turn as voice-origin and attempted synthesis; synthesis failed, so the
designed text fallback fired. Not a routing bug — the question was only why synthesis
failed.

## Root cause

`SYNTHESIZE_TIMEOUT_MS` was a flat `20_000`. Real synthesis of a long reply takes ~27-30s
on this host, so `execFile` killed the child at 20s.

Reproduced directly against the real venv with real Rachel-shaped text:

```
OLD flat 20s : FAILED at 20087ms -> synthesize failed (exit 1): ... (killed) signal=SIGTERM
NEW scaled   : SUCCESS in 27230ms (budget 110990ms)
```

Measured synthesis cost on this host:

| Text length | Duration |
|---|---|
| 210 chars | 6s (dominated by cold model load) |
| 9800 chars | 25s |
| 9696 chars (prose with paths, parens, quotes) | 30s |

## The fix

1. **`synthesizeTimeoutMs(textLength)`** replaces the flat constant — 30s floor for cold
   model load, 10ms/char (~3x the measured 3.1ms/char worst case), capped at 300s so a
   runaway reply can't hang the bridge. Exported so callers and tests assert the budget
   for a length rather than pinning a constant. **Scaling removes the cliff instead of
   relocating it**, which a bumped number would have done.
2. **`synthesize()` now includes child stdout in the thrown error.** mlx-audio's
   `generate_audio` catches every exception, prints the reason with a bare `print()` —
   *stdout* — and returns normally without writing a file. Reading stderr alone discarded
   the only diagnostic upstream provides.

`scripts/speech/synthesize.py` was deliberately **not** changed: stubbing `generate_audio`
to return without writing shows it already exits 1 and already names the no-output
condition on stderr. The reason is simply on stdout, which was ours to capture.

## The diagnostic trap — worth internalising

The failure looked like a Python-side `exit 1`, not a timeout. The log line read:

```
voice reply synthesis failed, falling back to text: synthesize failed (exit 1): Fetching 56 files: 100%...
```

A harmless model-download progress bar, no kill signal. That reading was **wrong**.

The bridge writes the child's whole captured stderr into one log entry — here **84 lines /
11207 chars** — and `(killed) signal=SIGTERM` is its *final* line. `awk NR==100` returns
only the first line of that entry. A follow-up search that capped its window at 4000 chars
missed it too.

> ⚠️ A log entry embedding a subprocess's stderr is one **record** spanning many lines, not
> one line. Line-oriented tools (`awk`, `grep`, `sed -n`, `head`) read a fragment and can
> invert a conclusion. Read to the next entry delimiter, uncapped — or better, reproduce at
> the boundary, which settled this in one run after log-reading had misled twice.

## Relationship to prior wiki content

This closes, from the other direction, a gap
[[investigations/2026-07-18-telegram-voice-stt-tts-spec-gaps]] recorded at spec time:

- **Its item 3** ("No length/duration bound on synthesized output") was resolved in PR #42
  by dropping the unsourced 1000-char cap, with the note that removing the cap made
  oversized replies reachable "pending a real long-reply round-trip on the host". This
  incident **was** that round-trip. The failure wasn't the OGG size ceiling it predicted —
  it was the synthesis *time* budget, which no one had length-scaled.
- **Its item 6** flagged that the spec's cited `execFile`-with-timeout precedent was only
  half-real. The timeout did get implemented, correctly — the defect was that it was a
  constant while the cost it bounds is a function of input length.

## Verification

- 8 new tests, each watched fail first (TDD red→green); 448/448 suite green
- Frozen tier-1 evals, all 5 scripted P0s pass with negative controls in separate files
- Drive-the-artifact eval: the real pipeline (venv python + mlx-audio + ffmpeg) turns a
  9696-char reply into a playable OGG/Opus (2,304,783 bytes) in 31.6s
- **Deployed**: launchd bridge booted out and bootstrapped onto merged code (pid 35517 →
  95892), heartbeat fresh. Merging is not deploying for a long-lived service.

Eval artifact: `https://github.com/blueman82/assistant-agent/pull/55#issuecomment-5049053383`
