---
title: "Repo depersonalisation — what was untracked, what was kept, and why"
type: investigation
created: 2026-07-21
last_updated: 2026-07-21
status: implemented
sources:
  - .gitignore
  - prompts/system.md
  - proactive/systemPrompt.ts
  - scripts/install.sh
  - scripts/install.test.ts
tags: [repo-hygiene, privacy, depersonalisation, gitignore, system-prompt, install]
---

# Repo depersonalisation — 2026-07-21

`blueman82/assistant-agent` is public. Several tracked files carried the
operator's name, personal email address, absolute home paths, or private working
artefacts. This records what was removed, what was deliberately kept, and the one
removal that had to be reverted.

Governing principle, as stated by the owner: **this repo runs on one machine.
Nothing personal ships. Anything needed at runtime stays on disk, untracked.**

## Removed from the repo

| Path | What it was | Why it went |
|---|---|---|
| `tasks/inbox-brief.md` | Operator's own task definition | Personal content; read at runtime by the launchd job |
| `tasks/proactive-calendar.md` | Operator's own task definition | Same |
| `.claude/settings.local.json` | Local agent permissions | Machine-specific |
| `.claude/test_command` | Commit test gate command | Machine-specific; the coderails `test_gate` hook reads it from disk |
| `.claude/evals-*.json` (4 files) | Recorded eval artefacts | Private working artefacts |
| `evals/e3-ask-before-acting.md` + results | Behavioural eval spec and the operator's recorded results | Private; referenced by no code |
| `workflow.config.yaml` | coderails workflow config | Carries an absolute `wiki_path` under the operator's home |
| `.DS_Store`, `.obsidian/` | macOS and Obsidian cruft | Never should have been trackable |

Every one of these **remains on disk** and is still read at runtime. Only the
git tracking was removed.

### Why `workflow.config.yaml` was untracked rather than genericised

Setting `wiki_path: null` is not a neutral placeholder — it is a live instruction
meaning *skip the wiki phases entirely* (`commands/prep.md:13`,
`commands/workflow.md:12`). Genericising the file would have silently disabled
the wiki skills on the machine where the vault actually exists. Untracking keeps
it working locally and out of the repo.

## Reverted: the launchd plists

`tasks/*-launchd.plist` were untracked in the same sweep, then **put back**.

A fresh worktree — a clean checkout of exactly what the repo ships — failed its
baseline: **378 passed / 26 failed**, every failure in `scripts/install.test.ts`,
25 of them `template missing: .../tasks/inbox-brief-launchd.plist`. The main
checkout had kept passing only because the untracked files were still sitting on
its disk.

The plists turned out to be **install machinery, not personal content**. They
contain zero absolute paths: `scripts/install.sh` stamps `__REPO_PATH__` at
install time (`install.sh:200`) and dies loudly if the placeholder survives
(`install.sh:204-206`). The only operator reference was three comment lines
reading "Gary's waking hours (Ireland time)", now "the operator's waking hours
(local time)".

They are re-tracked with an explicit negation so the intent is legible rather
than implicit:

```
tasks/*
!tasks/.gitkeep
!tasks/EXAMPLE-task.md
!tasks/*-launchd.plist
```

> The lesson worth keeping: "is this file personal?" and "does the repo need this
> file?" are **different questions**. The plists looked personal because of where
> they lived and what they scheduled, but nothing in them identified anyone. A
> clean-checkout test run is what settled it — reasoning about the file's
> *location* would not have.

## The system prompt

`prompts/system.md` named the operator **38 times**: full name, personal Gmail
address, "based in Ireland", and absolute `/Users/harrison` paths. It ships in
the repo.

It could not simply be untracked — `rachel.ts` exits 2 without it, so a clone
would be non-functional. Instead the file was split:

- **`prompts/system.md`** (tracked, generic) — "the operator" throughout, no
  address, `<repo>` and `<absolute-home>` placeholders. All 191 lines and every
  behavioural rule preserved; this was a substitution, not a rewrite.
- **`prompts/system.local.md`** (gitignored) — the operator's real prompt,
  unchanged.

`proactive/systemPrompt.ts` resolves which to load, first hit wins:

1. `$RACHEL_SYSTEM_PROMPT` — explicit override, any path
2. `prompts/system.local.md` — the operator's own, if present
3. `prompts/system.md` — the generic tracked fallback

Extracted into `proactive/` rather than left inline in `rachel.ts` so it is
testable without executing `rachel.ts`'s top-level side effects, and so the
existing test glob picks it up.

Two edge cases decided deliberately:

- A **blank** `RACHEL_SYSTEM_PROMPT` is treated as unset, not as a path to `""`.
  An empty env var is far more likely an unset-variable accident in a launchd
  plist than a deliberate request. Falling through beats failing to start.
- The explicit override **short-circuits before any existence probe**, so a bad
  path surfaces as `rachel.ts`'s own missing-prompt error naming that path,
  rather than silently falling back to a different prompt.

Five tests cover all three branches plus both edge cases, and are
mutation-proved: ignoring the local override fails 1, dropping the blank-value
guard fails 1. **416/416 green**, `tsc --noEmit` clean.

Verified live afterwards: Rachel still answers "I work for Gary Harrison" —
identity intact from the local prompt, while the repo ships a generic one.

## Verification

- Remote `prompts/system.md`: **0** personal references
- Remote tracked `tasks/`: only `.gitkeep`, `EXAMPLE-task.md`, three plists
- No `.claude/`, `evals/`, or `workflow.config.yaml` on the remote
- Every ignore rule checked **both** directions — the personal files ignored,
  and `EXAMPLE-task.md`, `rachel.ts`, `package.json`, `bin/rachel` not

## Open

`scripts/install.sh` still installs `com.rachel.inbox-brief` and
`com.rachel.proactive-calendar`, whose `.md` task files no longer ship. On this
machine they exist on disk so nothing breaks. A genuine fresh clone would install
those jobs and find their task files missing. Not reconciled — the owner's
position is that this repo is single-machine, so the fresh-clone path is not a
supported scenario.

See [[capabilities/installation]], [[capabilities/tasks]].
