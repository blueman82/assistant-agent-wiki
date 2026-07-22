# Assistant-Agent Wiki

Gary's AI assistant (Rachel) knowledge base. Read this first when answering queries.
Schema and workflows: see `AGENTS.md` in the project directory (`~/Github/assistant-agent/AGENTS.md`).

**Drop zone**: put files to ingest in `raw/` — then say "ingest raw/" to Rachel.

## Architecture

- [[architecture/overview]] — plumbing/brain split, SDK wiring, session continuity (bridge-restart persistence exception since PR #51)
- [[architecture/mcp-integrations]] — Gmail, Google Calendar, Slack, Chrome extension, mcp-exec

## Capabilities

- [[capabilities/tasks]] — task schema (two frontmatter shapes: Gary-tasks vs. loop-launcher), path, CRUD behaviour; `tasks/*.md` gitignored as of PR #43 (personal files stay on disk, untracked; only `EXAMPLE-task.md` + the 3 launchd plists remain tracked — `inbox-brief.md`/`proactive-calendar.md` were untracked by `a505042`, so a fresh clone lacks two runtime files `install.sh` still installs jobs for); also the single destination for actionable backlog ingested from `raw/` across all repos, via `AGENTS.md`'s Step 0 triage rule (`Repo: <path>` line for non-`assistant-agent` items, no per-repo task storage)
- [[capabilities/slack]] — personal Slack read + send, confirm-before-send, search consent rules
- [[capabilities/email]] — Not yet documented.
- [[capabilities/calendar]] — Not yet documented.
- [[calendar-log]] — Persistent index of upcoming events (7-14 days) and recent completions (30 days), auto-refreshed by the proactive-calendar task 4x/day.
- [[capabilities/send-gate]] — deterministic PreToolUse hook enforcing confirm-before-send on Slack/Calendar
- [[capabilities/ask-user-question-hook]] — PreToolUse hook denying AskUserQuestion tool, forcing conversational asking; E3 eval 4/4 pass (2026-07-19)
- [[capabilities/telegram-frontend]] — Telegram chat front-end onto Rachel; owns the getUpdates loop, single-user only, strips markdown from replies (no parse_mode); receives photos, image documents (PR #17), and PDF documents (PR #54, `[document: /path]` tag, distinct from `[image: /path]`); local voice in/out via mlx-whisper/Kokoro — voice-origin turns always answer in voice regardless of length, text only on synthesis failure (Tasks 4, 6-9, PR #42 dropped an unsourced 1000-char cap; commit 6f409be added a character-count log line + Telegram voice-note caption; PR #44 added a mirrored inbound-success log line; PR #55 made the synthesis timeout scale with text length after a flat 20s budget killed a 9696-char reply mid-synthesis); self-monitors via a health state machine with 409 backoff, fetch timeout, and startup alert (PRs #21 + #22); writes a per-poll heartbeat and routes its own alerts through the proactive chokepoint, with the sweep watching it from outside — liveness boundary closed (PR #26)
- [[capabilities/inbox-brief]] — recommend-only Gmail sweep, six-tier taxonomy; Urgent/Action-required threads pushed individually through the proactive chokepoint, the rest as a batch brief via `notify.ts`; scheduled 4x/day + dashboard button + on-request (PRs #23, #27, #28)
- [[capabilities/proactive-layer]] — Rachel's proactive layer: the `push.ts` chokepoint (dedup, quiet hours 22:30–08:00, 10/day budget), the 30-min deterministic sweep (PR-red, bridge liveness, calendar <2h escalation), headless one-shots with `RACHEL_ALLOWED_TOOLS` narrowing, and the security invariants (PRs #24–#26, #28)
- [[capabilities/installation]] — one-command installer: stamps + installs + verifies all four Rachel launchd services, config bootstrap-if-absent, truthful PASS/FAIL contract, launchd teardown-race handling (PRs #30-#32)
- [[capabilities/memory]] — persistent file-based fact store at `~/.rachel/memory/`, deterministically injected into the system prompt every turn via `composeSystemPrompt`; write/recall/update-over-duplicate/delete-when-wrong/self-maintenance contract; 32 KiB size guard with UTF-8-safe truncation (PRs #49-#52)

## Patterns

- [[patterns/extending-system-md]] — how to evolve the brain without touching TypeScript

## Investigations

- [[investigations/2026-07-16-model-effort-switching-assumptions]] — plan for runtime `/model`/`/effort` switching: the model-constant-vs-per-turn asymmetry, why "shared in-memory state" is per-process not global, and the `/status` staleness bug it would introduce
- [[investigations/2026-07-18-telegram-voice-stt-tts-spec-gaps]] — cross-check of the Telegram voice STT+TTS spec against bridge architecture: FIFO shape change confirmed, and 7 assumed-but-unenforced gaps (drainFifo reply branch, no markdown-stripping before TTS, no length cap on synthesized output, no spoken-register signal to the model, no install.sh preflight for the new Python/ffmpeg deps, a half-accurate execFile-timeout precedent, and compounding the already-filed temp-file cleanup debt)
- [[investigations/2026-07-21-repo-depersonalisation]] — what was untracked from the public repo and why: personal task files, `.claude/`, `evals/`, `workflow.config.yaml`; the system prompt split into a generic tracked `system.md` plus a gitignored `system.local.md` resolved by `proactive/systemPrompt.ts`; and the one removal that had to be **reverted** — the launchd plists are install machinery, not personal content, and untracking them failed 26 tests on a clean checkout. Carries the general lesson: "is this file personal?" and "does the repo need this file?" are different questions, and only a clean-checkout run settles the second.
- [[investigations/2026-07-21-rejected-shared-session-thread]] — a single shared session thread across terminal and Telegram was considered and rejected: concurrent-resume forks the SDK's parentUuid chain, compaction makes a persisted thread non-durable anyway (the decisive reason), and headless one-shots would pollute the operator's personal thread. Shipped instead: a separate memory store (durable, survives compaction) + narrowly-scoped bridge-only session persistence (Telegram continuity only)

## Sources

- [[sources/karpathy-llm-wiki]] — the LLM wiki pattern this knowledge base is based on
- [[sources/2026-06-29-wire-in-slack]] — wiring personal Slack into Rachel (replace-not-add decision)
- [[sources/2026-07-08-telegram-reply-formatting]] — Telegram reply markdown-stripping fix and the parse_mode rejection rationale
- [[sources/2026-07-08-rachel-rebrand]] — secretary → Rachel rebrand (naming only, no behaviour change)
- [[sources/2026-07-08-telegram-emit-channel]] — typed emit channel (text/tool/meta) keeps tool echoes and the done footer out of Telegram replies
- [[sources/2026-07-22-pr55-voice-synthesis-timeout]] — voice replies failed on long text: a flat 20s synthesis timeout killed the child; length-scaled budget + stdout capture in thrown errors
- [[sources/2026-07-08-telegram-image-reception]] — PR #17: Rachel can now receive photos and image documents via Telegram; "PDF scope dropped" decision later superseded by PR #54
- [[sources/2026-07-22-telegram-pdf-ingestion]] — PR #54: PDF document ingestion via Telegram (`[document: /path]` tag) plus a post-review dot-less-filename extension bugfix
- [[sources/2026-07-14-terminal-tool-echo-truncation]] — removed the 100-char cap on terminal tool-use echoes in rachel.ts (terminal-only; Telegram unaffected)
- [[sources/2026-07-14-bridge-self-monitoring]] — PRs #21 + #22: bridge health state machine (409 backoff, threshold-5 exit), fetch timeout, and startup alert
- [[sources/2026-07-14-inbox-brief]] — PR #23: recommend-only Gmail sweep + `bridge/notify.ts` standalone Telegram sender for headless one-shot runs
- [[sources/2026-07-15-proactive-layer]] — PRs #24–#26 + #28: push chokepoint, deterministic sweep, bridge heartbeat + closed liveness boundary, one-shot wiring with tool narrowing; deployed live 2026-07-15
- [[sources/2026-07-15-installer]] — PRs #30-#32: one-package installer, push.ts TDZ hotfix, live-found launchd teardown race; clean-slate cycle verified live
- [[sources/2026-07-20-persistent-calendar-index]] — PR #48: persistent calendar index (upcoming 7-14d + recent 30d) auto-refreshed 4x/day; solves cross-session calendar awareness
- [[sources/2026-07-21-cross-platform-persistent-memory]] — PRs #49-#52: memory store contract + deterministic prompt injection + 32 KiB UTF-8-safe size guard, plus bridge-only session persistence and the one-writer invariant runtime hole found and fixed post-merge
