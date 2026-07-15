# Assistant-Agent Wiki

Gary's AI assistant (Rachel) knowledge base. Read this first when answering queries.
Schema and workflows: see `AGENTS.md` in the project directory (`~/Github/assistant-agent/AGENTS.md`).

**Drop zone**: put files to ingest in `raw/` — then say "ingest raw/" to Rachel.

## Architecture

- [[architecture/overview]] — plumbing/brain split, SDK wiring, session continuity
- [[architecture/mcp-integrations]] — Gmail, Google Calendar, Slack, Chrome extension, mcp-exec

## Capabilities

- [[capabilities/tasks]] — task schema, path, frontmatter contract, CRUD behaviour
- [[capabilities/slack]] — personal Slack read + send, confirm-before-send, search consent rules
- [[capabilities/email]] — Not yet documented.
- [[capabilities/calendar]] — Not yet documented.
- [[capabilities/send-gate]] — deterministic PreToolUse hook enforcing confirm-before-send on Slack/Calendar
- [[capabilities/telegram-frontend]] — Telegram chat front-end onto Rachel; owns the getUpdates loop, single-user only, strips markdown from replies (no parse_mode); receives photos and image documents (PR #17); self-monitors via a health state machine with 409 backoff, fetch timeout, and startup alert (PRs #21 + #22); writes a per-poll heartbeat and routes its own alerts through the proactive chokepoint, with the sweep watching it from outside — liveness boundary closed (PR #26)
- [[capabilities/inbox-brief]] — recommend-only Gmail sweep, six-tier taxonomy; Urgent/Action-required threads pushed individually through the proactive chokepoint, the rest as a batch brief via `notify.ts`; scheduled 4x/day + dashboard button + on-request (PRs #23, #27, #28)
- [[capabilities/proactive-layer]] — Rachel's proactive layer: the `push.ts` chokepoint (dedup, quiet hours 22:30–08:00, 10/day budget), the 30-min deterministic sweep (PR-red, bridge liveness, calendar <2h escalation), headless one-shots with `RACHEL_ALLOWED_TOOLS` narrowing, and the security invariants (PRs #24–#26, #28)

## Patterns

- [[patterns/extending-system-md]] — how to evolve the brain without touching TypeScript

## Investigations

*(empty — file answers back here as they compound)*

## Sources

- [[sources/karpathy-llm-wiki]] — the LLM wiki pattern this knowledge base is based on
- [[sources/2026-06-29-wire-in-slack]] — wiring personal Slack into Rachel (replace-not-add decision)
- [[sources/2026-07-08-telegram-reply-formatting]] — Telegram reply markdown-stripping fix and the parse_mode rejection rationale
- [[sources/2026-07-08-rachel-rebrand]] — secretary → Rachel rebrand (naming only, no behaviour change)
- [[sources/2026-07-08-telegram-emit-channel]] — typed emit channel (text/tool/meta) keeps tool echoes and the done footer out of Telegram replies
- [[sources/2026-07-08-telegram-image-reception]] — PR #17: Rachel can now receive photos and image documents via Telegram
- [[sources/2026-07-14-terminal-tool-echo-truncation]] — removed the 100-char cap on terminal tool-use echoes in rachel.ts (terminal-only; Telegram unaffected)
- [[sources/2026-07-14-bridge-self-monitoring]] — PRs #21 + #22: bridge health state machine (409 backoff, threshold-5 exit), fetch timeout, and startup alert
- [[sources/2026-07-14-inbox-brief]] — PR #23: recommend-only Gmail sweep + `bridge/notify.ts` standalone Telegram sender for headless one-shot runs
- [[sources/2026-07-15-proactive-layer]] — PRs #24–#26 + #28: push chokepoint, deterministic sweep, bridge heartbeat + closed liveness boundary, one-shot wiring with tool narrowing; deployed live 2026-07-15
