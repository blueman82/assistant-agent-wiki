# AGENTS.md — vault contract

wiki-vault: true

This vault's own copy of the rules that govern it. It exists so the contract
travels with the content: this wiki is a separate git repo from
`assistant-agent`, so an agent editing a page here has no path up to that
repo's `AGENTS.md`. Anything that must be enforceable — by a human reading it
or by a hook parsing it — has to live at this root.

The authoring workflow (ingest, query, lint) is documented in
`assistant-agent/AGENTS.md`. This file carries the structural contract only.

## Page types

Every page lives in exactly one of these directories. Nothing else is a page
type. A page that fits none of them means either the taxonomy needs extending
here first, or the page belongs somewhere other than the wiki.

| Directory | Purpose |
|-----------|---------|
| `architecture/` | How the system is built — components, wiring, data flow |
| `capabilities/` | What Rachel can do — one page per tool surface |
| `patterns/` | Reusable approaches — how to extend, configure, evolve |
| `investigations/` | Filed-back answers to queries: a question posed, alternatives weighed, a conclusion recorded |
| `sources/` | Ingested references — docs, gists, PRs |
| `templates/` | Page skeletons — copy, don't edit |
| `personal/` | Gary's own personal-life reference facts (memberships, appointments, contacts) — not about Rachel's engineering, but durable facts Rachel should recall on request |

`raw/` is not a page type. It holds immutable ingested input, never edited in
place.

Root-level files are not pages and are not governed by the table above:
`index.md` (navigation, read first), `log.md` (append-only history),
`calendar-log.md`, and this file.

## Why this is enforced, not advised

On 2026-07-21 an agent wrote a page into `decisions/` — not a page type. It
was following an existing off-taxonomy precedent rather than inventing one,
which is the point: once a stray directory exists, later agents treat it as
sanctioned. Both pages were moved into `investigations/`, which is where a
"decision" page belongs — each poses a question, weighs alternatives, and
records a conclusion.

The cost of that drift was not the directory itself. Six files carried
`[[decisions/...]]` wikilinks, so the correction was a move plus a rewrite of
every inbound link, and a bare `git mv` would have traded a taxonomy violation
for six dead links.

The table above is therefore machine-read by a `PreToolUse` hook that blocks
writes into unsanctioned directories. **To add a page type, edit this table
first** — the hook parses this file, so the table is the single source of
truth and enforcement follows it automatically. There is no second list to
keep in sync.
