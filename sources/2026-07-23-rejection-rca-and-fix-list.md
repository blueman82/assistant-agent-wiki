---
title: "RCA 2026-07-23 — Ghost Tool Rejections, No Report-Back, HF Hub Stalls"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources: ["docs/coderails/specs/rca-2026-07-23-fix-list.md", "bridge/telegram-bridge.ts", "bridge/speech.ts", "scripts/speech/transcribe.py", "rachel.ts", "prompts/system.md", ".rachel/telegram-bridge.log"]
tags: [rca, investigation, telegram, bridge, turn-timeout, permissions, bypass-permissions, voice, stt, huggingface, deploy-discipline, wake-channel]
---

## Why this page exists

The source document (`docs/coderails/specs/rca-2026-07-23-fix-list.md`) lives in a **gitignored**
directory. This wiki page is the only durable home for it — a clean checkout of `assistant-agent`
contains neither the RCA nor the spec it produced.

Triggered by Gary, 2026-07-23 evening, verbatim: *"debug why you are having tool problems, why you
are not proactively messaging me back when you say you are. Don't stop until you have a final RCA
backed up with evidence and verified."* Every claim in the source doc was re-derived from primary
sources — SDK session JSONL under `~/.claude/projects/-Users-harrison-Github-assistant-agent/`,
`.rachel/telegram-bridge.log`, repo code, git history, and live reproduction on the host.

**Status: the RCA itself is complete and verified.** Its fix list is in three different states —
see [[#Fix list — three states, not two]]. Do not read the fix list as a description of shipped
behaviour.

## Not the same RCA as PR #55

There are now two voice-pipeline RCAs a week apart, both landing on `bridge/speech.ts`, both
producing `SIGTERM` in the bridge log. **They are different bugs and must not be conflated:**

| | [[sources/2026-07-22-pr55-voice-synthesis-timeout]] | This RCA (finding 3) |
|---|---|---|
| Direction | **Outbound** — TTS synthesis | **Inbound** — STT transcription |
| Function | `synthesize()` (mlx-audio/Kokoro) | `transcribe()` (mlx-whisper) |
| Root cause | Flat 20s budget vs ~30s real synthesis of long text | A HuggingFace Hub freshness check on every call, hanging when the endpoint is unreachable |
| Scales with | Reply **length** | Nothing — it's a network condition, not compute |
| Status | **Fixed and merged** (PR #55, length-scaled budget) | **Decided, not built** (fix-list items 1–4) |

The shared symptom (`signal=SIGTERM` from `execFile`'s timeout) is what makes them look identical
in the log. The tell is the direction: `transcribe failed` vs `synthesize failed`.

## Headline findings

### 1. The "user doesn't want…" rejections were machine-generated — no human denied anything

Four rejection events that evening, from **two distinct mechanisms**. Neither is a person.

**Mechanism A — turn-death ghosts.** `"The user doesn't want to proceed with this tool use"`
co-occurring within the same sub-second as `[Request interrupted by user for tool use]`, with the
turn ending there. This is the harness's wording for an aborted turn's in-flight tool call. The
bridge aborts every turn at 10 minutes (`DEFAULT_TURN_TIMEOUT_MS`). Cleanest proof: Gary's message
arrived 22:20:19, the abort landed 22:30:19.589 — **600 seconds to the second**. Retries appeared
to "work" only because each new turn got a fresh 10-minute budget.

**Mechanism B — permission prompts with nobody to answer them.** `"The user doesn't want to take
this action right now. STOP…"` with the turn *continuing* afterwards. That trailing continuation is
the discriminator: it's the permission layer denying one call, not an abort. Under
`permissionMode: "auto"` the SDK raises a prompt for anything not on the auto-approval list;
headless, nobody can answer, so it auto-denies with exactly that string. **This also killed the
wiki-ingest `Agent` spawns during the PDF loop** — `Agent` is not in `DEFAULT_ALLOWED_TOOLS`, and
that constant is an *auto-approval* list, not a tool-*availability* list. The distinction is the
whole bug.

**How to tell them apart, for the next reader:** check elapsed time. Mechanism A lands at exactly
600s from turn start and the turn dies. Mechanism B lands anywhere and the turn keeps going.

**Aggravating factor — merged is not deployed, second occurrence.** [[sources/2026-07-23-pr59-bypass-permissions]]
(PR #59, merged 16:55, commit `8831dc3`) *was* the fix for mechanism B, but the launchd bridge kept
running pre-#59 code until its 22:56 restart. **The entire painful conversation ran on stale code.**
This is what fix-list items 10 and 11 exist to prevent.

### 2. Rachel structurally cannot "report back later"

Not a bug — a structural property of the architecture, and the direct motivation for the wake
channel ([[sources/2026-07-24-streaming-relay-wake-channel]], Part B).

- `runTurn`'s only caller is `drainFifo`, fed **only** by inbound Telegram messages. No timer, no
  completion hook, nothing else can start a turn.
- Each turn is a fresh `claude` subprocess; harness background tasks die with it. Observed twice
  that evening: a profiling command backgrounded at 20:55, turn aborted 20:56, and the *next* turn
  (21:21 — which happened only because Gary messaged) received a "stopped" notification with an
  empty output file.
- An aborted turn's reply text is replaced by the generic cutoff notice, so any "I'll report back"
  written mid-turn **evaporates**.

The only pattern that survives today is the detached ad-hoc task spawn
([[sources/2026-07-22-adhoc-background-escalation]]). Rachel's own memory now carries this rule at
`~/.rachel/memory/turn-lifecycle-background-work.md` (see [[capabilities/memory]]).

### 3. Voice transcription failures are a HuggingFace network check, not slow compute

`scripts/speech/transcribe.py` calls
`mlx_whisper.transcribe(path_or_hf_repo="mlx-community/whisper-small.en-mlx")`, which performs a
Hub **freshness check on every call even with the model fully cached** — under a hard 30s
`execFile` budget (`bridge/speech.ts:16`). Reproduced on real audio:

| Condition | Result |
|---|---|
| Normal network | 12.2s, stderr shows `Fetching 4 files` |
| `HF_HUB_OFFLINE=1` | **4.3s**, no network touch |
| HF endpoint unreachable | **Hangs past 45s before touching audio** — the 30s budget SIGTERMs it |

That third row matches the five `transcribe failed … signal=SIGTERM` entries in the bridge log,
clustered with Telegram-side `Bad Gateway`/429/fetch-timeout errors from the same host — i.e. the
host's network was the common cause of both.

**A competing theory was disproved, not just doubted:** "slow Python import" fails, because import
completes in ~5s even with the network blackholed.

### 4. PR #56 (ad-hoc backgrounding) is cleared

Worth recording because the timing made it the obvious suspect. PR #56 touched only the bridge
cutoff message, its tests, and `system.md`. The 10-minute deadline (**PR #46**) and the 30s
transcribe budget both date from 2026-07-18, *before* it. The rejection strings originate in the
harness, not the repo. The correlation was real but causally empty: #56's new cutoff text said
"background it", Rachel began backgrounding *in-turn* (which cannot survive — see finding 2), and
that week's long investigation turns started tripping the deadline more often.

### 5. Contributing environment noise

Gary's global `pre_tool_use_with_tts.py` hook failed on **every one** of Rachel's Bash calls
(`env: uv: No such file or directory`, exit 127, ~1s each, 30 occurrences that evening), and
`UserPromptSubmit` hooks were timing out at 5s per prompt.

Resolved 2026-07-24: the hook was found to be **non-functional in all its guard duties** — it used
`sys.exit(1)`, which does not block a tool call — and was deleted entirely along with its orphan
files. `destructive_bash_gate.sh` and `test_gate.sh` remain and are **blessed** for Rachel's
environment (fix-list item 13: she gets the same hooks as any Claude session).

## Fix list — three states, not two

The numbering below is **canonical**: [[sources/2026-07-24-streaming-relay-wake-channel]] references
items 10, 11 and 17 by these numbers.

### Closed — shipped, no action outstanding

- **Mechanism B** — PR #59 `bypassPermissions`, live since the 22:56 bridge restart. See
  [[sources/2026-07-23-pr59-bypass-permissions]]. (That page's own open question — the mode was set
  without the documented-as-required `allowDangerouslySkipPermissions: true` — is **not** resolved
  by this RCA and remains open.)
- **tts hook** — settings entry, script, `current_activity.tmp`, and both `pre_tool_use.json` logs
  deleted; no references remain.
- **UserPromptSubmit discards** — Gary raised the hook timeout 5s → 10s.
- **Turn-lifecycle rule** — written to Rachel's own memory store.
- **PR #61** (speech.ts docs) — merged.

### Specced and approved, not built

Items **10, 11, 17** — the whole of [[sources/2026-07-24-streaming-relay-wake-channel]]. Three PRs
and a live drill remain.

- **10. Mid-turn progress relay** (Gary's own addition). His complaint verbatim: replies take so
  long because Rachel waits until text generation is done, forcing him to read reams of text with
  no sense of progress. He asked for stream events relayed to the human, mindful of Telegram API
  limits, with jitter/backoff. → Spec Part A.
- **11. Stale-process family (auto-remediating).** Compare bridge process start time against the
  newest relevant commit on main. **Per Gary's explicit instruction this must not ask for
  approval**: fix it — restart the bridge (bootout → poll-until-gone → bootstrap) — then tell him
  it was done. → Spec Part B, producer 2.
- **17. Completion→wake channel** — the real "Rachel comes back to you" capability, and the answer
  to finding 2. → Spec Part B.

### Decided but not built

Real decisions with no implementation yet. **Nothing below describes current behaviour.**

**A. Voice pipeline** (all from finding 3)
1. `HF_HUB_OFFLINE=1` in `transcribe()`'s child env (`bridge/speech.ts:84`). Tiny.
2. Same for `synthesize()` — it performs its own hub check (`Fetching 56 files` observed). Tiny.
3. `scripts/speech/setup.sh` pre-fetches and verifies both models so offline mode can never hit a
   cold cache. Small. *(Alternative to 1–3: pass a local snapshot path instead of the HF repo ref —
   pick one approach, not both.)*
4. **Transcribe budget 30s → 2 minutes.** Gary's decision, explicitly overruling the RCA's original
   "no change" recommendation: a human may legitimately send a longer voice note.

**B. Ghost-rejection hardening** (all from finding 1)

5. `prompts/system.md`: document both machine strings — abort injection vs unanswerable permission
   prompt; check elapsed time before attributing; **never attribute a decline to Gary**; never
   promise report-backs from in-turn background work.
6. **Abort inoculation** — after a deadline abort, `drainFifo` prefixes the next turn's prompt with
   a note that the previous turn was auto-aborted and its rejection strings are artifacts.
7. **Turn budgeting** in `system.md` — split investigations across turns; go detached past ~8
   minutes. **The 10-minute deadline itself stays** (decided, not revisited).
8. Log turn start plus queue depth — only completion/abort are logged today, which is exactly why
   this RCA needed JSONL cross-referencing.
9. Truncate synthesis-failure log detail — one failure wrote a full 9,696-char **private reply**
   into the log.

**D. Permission surface**

12. **`.env` protection** — nothing guards it now; the deleted hook's block never worked (finding
    5). Fold a deny branch into `destructive_bash_gate.sh`, which already has the correct
    JSON-decision plumbing. **Decided: yes.** Tiny.
13. **Hooks in Rachel's environment — decided: bless them.** No change.
14. **`bashPatterns.ts` coverage** — this is the *only* send enforcement left in bridge turns under
    `bypassPermissions`. Known holes: `curl --data` POSTs without `-X POST` evade the calendar
    detector; Telegram `sendVoice`/`sendDocument`, Slack `chat.update`/`chat.delete`, and Gmail
    `drafts.send` are absent entirely. Medium, with tests. See [[capabilities/send-gate]].
15. **Agent-tool policy under bypass — decided: allow for now.** Gary may later return to auto mode
    with a defined approval-toolset list.

**E. Hygiene**

16. **`~/.rachel/tmp` sweep** — 750 voice artifacts, plus the PR #17 image-leak debt already tracked
    on [[capabilities/telegram-frontend]]. The bridge's own voice cleanup is *correct*; the debris
    comes from direct test invocations and mid-synthesis kills, so a startup or daily sweep is the
    robust fix rather than more `finally` blocks. **Decided: agreed.** Small.
18. **Ops one-off — triaged, closed.** Stray `claude --dangerously-skip-permissions` processes were
    all interactive terminal sessions under `-zsh` on TTYs (coderails and assistant-agent
    worktrees), not dead Rachel jobs or runaway routines.
19. **Harness-originated `"Continue from where you left off."` stub**, seen once at 21:20:45. The
    string exists nowhere in the repo, so it originates in the harness. **Monitor only** — no
    observed harm.

## Lessons that outlive the fix list

1. **A tool-result string is not a person.** Both mechanisms in finding 1 produce text naming "the
   user". Neither involved Gary. Cross-session recurrence of a "the user declined" string is a tell
   that it's infrastructure, not a human — report it as a decline, never as "you rejected this".
2. **Merged is not deployed** (second occurrence). A long-running launchd service keeps running the
   old code until something restarts it. Six hours of a fixed bug still biting.
3. **An auto-approval list is not an availability list.** `DEFAULT_ALLOWED_TOOLS` gates prompting,
   not existence — a tool absent from it is still callable, it just deadlocks headless.
4. **Two bugs in one file with one symptom are still two bugs.** The `SIGTERM` in the log meant one
   thing on 2026-07-22 and a different thing on 2026-07-23.

## Related pages

- [[sources/2026-07-24-streaming-relay-wake-channel]] — the spec built from items 10, 11 and 17
- [[sources/2026-07-22-pr55-voice-synthesis-timeout]] — the adjacent, **different** voice RCA (TTS
  synthesis, not STT transcription)
- [[sources/2026-07-23-pr59-bypass-permissions]] — the mechanism-B fix, and the deploy lag that
  neutered it for six hours
- [[sources/2026-07-23-bridge-log-timestamps]] — PR #60, motivated by the *previous* day's RCA
  having to reconstruct timing from a side channel; this RCA still needed JSONL cross-referencing
  (fix-list item 8)
- [[sources/2026-07-22-adhoc-background-escalation]] — the only surviving detached-work pattern
  (finding 2), and the PR cleared by finding 4
- [[capabilities/telegram-frontend]] — the 10-minute turn deadline behind mechanism A
- [[capabilities/send-gate]] — `bashPatterns.ts`, the sole remaining send enforcement under bypass
  (item 14)
- [[capabilities/memory]] — where the turn-lifecycle rule was written
