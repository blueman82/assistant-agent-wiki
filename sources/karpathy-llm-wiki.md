---
title: "Karpathy — LLM Wiki Pattern"
type: source
created: 2026-06-07
last_updated: 2026-06-07
sources: ["https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f"]
tags: [source, pattern, reference]
---

> **Staleness disposition — settled 2026-07-24, do not re-flag.** This page is the vault's oldest by
> `last_updated` and has now been surfaced by four consecutive lints. **It needs no action, and an
> old date here is correct rather than a defect.** Two reasons: (1) it documents a fixed external
> reference — a published gist that does not change with this codebase, so `last_updated` tracking
> the repo would be meaningless; (2) it does not describe active code, which is the condition
> `AGENTS.md`'s staleness rule actually tests ("flag pages not updated in 90+ days **if they cover
> active code**"). At 47 days it is inside this vault's 90-day threshold anyway; it only trips the
> generic lint skill's stricter 30-day default. **Where the two rules disagree, this vault's
> `AGENTS.md` wins** — it is the schema of record for this wiki. `last_updated` is deliberately left
> at its true value; bumping it to silence a lint would falsify the one field the check reads.

## Summary

Andrej Karpathy's pattern for a persistent, compounding LLM knowledge base. The LLM incrementally builds and maintains structured markdown — the human browses it in Obsidian.

## Three layers

1. **Raw sources** (`raw/`) — immutable input: articles, papers, PDFs, code. LLM reads, never modifies.
2. **Wiki** — LLM-maintained markdown with wikilinks. Claude owns this layer entirely.
3. **Schema** (AGENTS.md) — tells the LLM how the wiki is structured and what workflows to run.

## Key workflows

- **Ingest**: process new raw sources, update 10–15 existing wiki pages (not create isolated summaries)
- **Query**: answer questions by reading the wiki; file good answers back as new `investigations/` pages
- **Lint**: periodic pass to fix contradictions, stale claims, orphaned pages

## What this wiki applies from the pattern

- Three-layer structure (raw → wiki ← schema)
- `index.md` read-first, `log.md` append-only
- `investigations/` for filed-back answers
- `sources/` for ingested references
- AGENTS.md in project dir (not vault)

## What this wiki omits

- Vector DB / provenance tracking — not needed at this scale (sub-100 sources)
