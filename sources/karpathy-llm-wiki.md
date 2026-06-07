---
title: "Karpathy — LLM Wiki Pattern"
type: source
created: 2026-06-07
last_updated: 2026-06-07
sources: ["https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f"]
tags: [source, pattern, reference]
---

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
