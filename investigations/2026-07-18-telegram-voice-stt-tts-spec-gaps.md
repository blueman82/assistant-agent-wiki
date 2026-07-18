---
title: "Telegram voice in/out (STT+TTS) spec — architecture cross-refs and unenforced assumptions"
type: investigation
created: 2026-07-18
last_updated: 2026-07-18
sources: ["docs/coderails/specs/telegram-voice-stt-tts.md", "bridge/telegram-bridge.ts", "bridge/api.ts", "proactive/sweep.ts", "prompts/system.md", "scripts/install.sh", "capabilities/telegram-frontend.md", "capabilities/installation.md"]
tags: [investigation, telegram, voice, stt, tts, bridge, spec-review]
---

## Question

Gary asked for a wiki cross-check of `docs/coderails/specs/telegram-voice-stt-tts.md` (approved under crack-on) against existing Telegram bridge architecture, FIFO/queue semantics, and any prior voice/audio decisions — plus whether a prior agentic loop's retro (session `182d03e1`, 2026-07-15) recording "multi-channel/voice ... premature for single-user v1" as out-of-scope is documented anywhere in the wiki.

## Findings

**No prior wiki content on voice/audio.** Confirmed by a fresh repo-wide grep for `mlx|Kokoro|whisper|opus|ogg` (case-insensitive) across the vault: the only matches are substring false positives (`logged` contains `ogg`, etc.) — same null result Gary had already found. This spec is the first voice/audio design in the project.

**The `182d03e1` "voice out of scope" retro is not a wiki page — it's a Claude Code session transcript ID**, found under `~/.claude/projects/-Users-harrison-Github-assistant-agent/182d03e1-.../`, not a git commit and not ingested into the wiki. There is nothing in `index.md` or `log.md` recording that non-goal, so there is no stale wiki claim to correct or supersede — the crack-on-approved spec simply proceeds against a clean slate. Nothing to reconcile.

**FIFO shape change is real and matches the spec exactly.** Current `bridge/telegram-bridge.ts:419` declares `const fifo: string[] = []`; entries are pushed as bare strings at the image branch (`fifo.push(input)`, line 609) and the text branch (`fifo.push(text)`, line 614), then drained in `drainFifo` (`const text = fifo.shift()!`, line 635) straight into `runTurn`. The spec's planned shape (`{ text: string; voice: boolean }`) is a minimal, additive change consistent with this — but see the drainFifo gap below, which the spec doesn't spell out at the same level of concreteness it gives the inbound side.

**Bridge ownership/routing model the new voice paths must respect.** Per [[capabilities/telegram-frontend]]: `bridge/telegram-bridge.ts` is the sole `getUpdates` consumer (Telegram allows exactly one per token); ordinary messages go FIFO → `runTurn`, `callback_query` taps go straight to the imported `telegramSurface` ahead of the FIFO. A `msg.voice` branch slots into `handleMessage` exactly like the existing `msg.photo || msg.document` branch (same chat-id auth gate at the top of `handleMessage`, `bridge/telegram-bridge.ts:513`) — no new auth surface, single-user constraint inherited for free.

## Constraints the spec assumes but doesn't enforce in code

1. **`drainFifo`'s reply branch isn't a concrete task item.** The spec's Components section details the *inbound* `msg.voice` branch precisely, but the *outbound* decision point — `drainFifo` (`bridge/telegram-bridge.ts:630-664`) currently always calls `reply()` (=`sendChunked`, line 472-474) after joining the emit buffer — is only described at the Architecture-section level ("once runTurn's reply text is ready, synthesize... instead of sendChunked()"). Nothing pins how the shifted FIFO entry's `voice` flag reaches that call site, nor what happens on the existing `buffer.join(...) || "(no output)"` fallback (line 662-663) for a voice-origin turn: synthesize the literal string "(no output)" as speech, or force a text fallback even though the turn was voice-in? Undecided.

2. **No markdown-stripping step before TTS.** `bridge/api.ts`'s `stripMarkdown` is currently applied only inside `sendChunked` (`api.ts:121`), as the deterministic backstop behind `prompts/system.md`'s "Plain text replies" rule. The spec's synthesize step reads `runTurn`'s raw reply text with no mention of running it through `stripMarkdown` (or an equivalent) first — a stray `**bold**`/`` `backtick` `` the model still emits would be read aloud literally by Kokoro instead of being silently stripped, same failure mode the text path already solved once.

3. **No length/duration bound on synthesized output.** Text replies have an enforced chunking boundary (`chunkText`, `api.ts:99-118`, 4096-char Telegram limit, tested and documented). The spec has no equivalent for voice — no cap on how long a reply may run before TTS, and no split-into-multiple-voice-messages story for a long reply (e.g. an inbox brief or task summary spoken in full). `sendVoice` itself also isn't checked against Telegram's own voice-message size ceiling.

4. **Rachel has no signal that a turn is voice-origin, and no spoken-register guidance.** The spec's inbound design deliberately pushes plain transcribed text to the FIFO (not a tagged prefix like the image protocol's `[image: path]`, which `prompts/system.md:72` instructs the model to parse) — so the model itself never learns a turn will be spoken back. Yet `prompts/system.md:185-187`'s ground rules currently say **"Bullet points over paragraphs"** and **"Simple hyphen bullets are fine"** — exactly the format TTS renders worst (no pause structure, bullets read as disconnected fragments). The spec doesn't touch `prompts/system.md` at all, so a voice-origin reply is composed under rules tuned for a text surface, with no seam to do otherwise.

5. **No preflight/install-time verification of the new external dependencies**, unlike the project's established pattern. Per [[capabilities/installation]], `scripts/install.sh` already fails loud on a missing prerequisite (Telegram credentials, node/node_modules). The spec's Environment section requires a dedicated `~/.rachel/venvs/speech` Python 3.12 venv (host currently defaults to 3.13), `mlx-whisper`/`mlx-audio`/`misaki`, and a working `ffmpeg` — but explicitly punts verification to "first implementation step" with no mention of wiring a check into `install.sh`'s existing preflight/verification phases. A redeploy or fresh machine could silently lack voice support with no PASS/FAIL signal, unlike every other dependency the installer already guards.

6. **The `execFile`-with-timeout precedent is only half-real.** The spec cites two prior patterns for `bridge/speech.ts`'s planned `execFile`-with-timeout, injectable-exec-function design: "`proactive/sweep.ts`'s existing execFile-with-timeout call" — confirmed accurate (`sweep.ts:8,48-62`, `defaultExecFn` passes `timeoutMs` straight to `execFile`'s `timeout` option) — and "`telegram-bridge.ts`'s `isPidAlive` check" — **not accurate**: `isPidAlive` (`bridge/telegram-bridge.ts:144-155`) shells out via synchronous `execSync` with no timeout at all. It's a real injectable seam (`isPidAliveFn`), just not an example of the timeout pattern the spec is citing it for. Low-stakes, but an implementer copying "the isPidAlive pattern" literally would drop the timeout the spec itself requires (30s/20s).

7. **New temp-file growth compounds a previously filed, still-open debt item.** [[capabilities/telegram-frontend]] already records: "Temp files in `~/.rachel/tmp/` are never cleaned up — the directory grows over time. No cleanup is scheduled" (from the PR #17 image-reception work). The voice spec adds at least one more inbound temp file per voice note (`.ogg`) plus at least one outbound intermediate (synthesized WAV before ffmpeg conversion, then the converted OGG) per voice-origin reply — multiplying the existing unbounded-growth debt without acknowledging or resolving it.

## Relationships

- [[capabilities/telegram-frontend]] — bridge ownership/routing model, reply-content contract, temp-file debt this spec extends
- [[capabilities/installation]] — the fail-loud preflight/verification pattern the spec's new Python/ffmpeg dependencies aren't wired into
- [[architecture/overview]] — plumbing/brain split; this spec touches only plumbing (`bridge/`), never `prompts/system.md`, which is where gap 4 (spoken-register awareness) would need to land
- [[sources/2026-07-08-telegram-image-reception]] — the `[image: path]`-prefix protocol the voice design deliberately does *not* mirror (transcribed text carries no origin tag to the model)
