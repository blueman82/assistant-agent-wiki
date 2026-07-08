# Assistant-Agent Wiki

Gary's AI secretary knowledge base. Read this first when answering queries.
Schema and workflows: see `AGENTS.md` in the project directory (`~/Github/assistant-agent/AGENTS.md`).

**Drop zone**: put files to ingest in `raw/` — then say "ingest raw/" to the secretary.

## Architecture

- [[architecture/overview]] — plumbing/brain split, SDK wiring, session continuity
- [[architecture/mcp-integrations]] — Gmail, Google Calendar, Slack, Chrome extension, mcp-exec

## Capabilities

- [[capabilities/tasks]] — task schema, path, frontmatter contract, CRUD behaviour
- [[capabilities/slack]] — personal Slack read + send, confirm-before-send, search consent rules
- [[capabilities/email]] — Not yet documented.
- [[capabilities/calendar]] — Not yet documented.
- [[capabilities/send-gate]] — deterministic PreToolUse hook enforcing confirm-before-send on Slack/Calendar
- [[capabilities/telegram-frontend]] — Telegram chat front-end onto the secretary; owns the getUpdates loop, single-user only, strips markdown from replies (no parse_mode)

## Patterns

- [[patterns/extending-system-md]] — how to evolve the brain without touching TypeScript

## Investigations

*(empty — file answers back here as they compound)*

## Sources

- [[sources/karpathy-llm-wiki]] — the LLM wiki pattern this knowledge base is based on
- [[sources/2026-06-29-wire-in-slack]] — wiring personal Slack into the secretary (replace-not-add decision)
- [[sources/2026-07-08-telegram-reply-formatting]] — Telegram reply markdown-stripping fix and the parse_mode rejection rationale
