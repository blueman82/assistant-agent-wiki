# Assistant-Agent Wiki

Gary's AI secretary knowledge base. Read this first when answering queries.
Schema and workflows: see `AGENTS.md` in the project directory (`~/Github/assistant-agent/AGENTS.md`).

**Drop zone**: put files to ingest in `raw/` — then say "ingest raw/" to the secretary.

## Architecture

- [[architecture/overview]] — plumbing/brain split, SDK wiring, session continuity
- [[architecture/mcp-integrations]] — Gmail, Google Calendar, Chrome extension, mcp-exec

## Capabilities

- [[capabilities/tasks]] — task schema, path, frontmatter contract, CRUD behaviour
- [[capabilities/slack]] — personal Slack read + send, confirm-before-send, search consent rules
- [[capabilities/email]] — Not yet documented.
- [[capabilities/calendar]] — Not yet documented.

## Patterns

- [[patterns/extending-system-md]] — how to evolve the brain without touching TypeScript

## Investigations

*(empty — file answers back here as they compound)*

## Sources

- [[sources/karpathy-llm-wiki]] — the LLM wiki pattern this knowledge base is based on
