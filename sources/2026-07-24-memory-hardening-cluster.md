---
title: "Memory system hardening: truncation fix, periodic lint, write gate, write lock (PRs #62-#65)"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources: ["proactive/memoryIndex.ts", "proactive/memoryLint.ts", "proactive/sweep.ts", "gate/memoryGate.ts", "gate/sendGate.ts", "rachel.ts", "prompts/system.md", "proactive/memoryLock.ts", "proactive/memoryAppend.ts", "tasks/inbox-brief-launchd.plist"]
tags: [memory, security, concurrency, write-gate, lint, symlink, path-traversal, hardening]
---

## Origin

An audit found Rachel's memory contract ([[capabilities/memory]], `prompts/system.md`'s `## Memory` section) was enforced by **nothing in code**. Empirical proof (verified): the first-ever fact file, `~/.rachel/memory/units-preference.md`, has no frontmatter at all — pure prompt-only compliance had already failed in the wild — and a second violating file appeared **during this loop**. Gary's steer: "prompt-only enforcement doesn't work as it outlines so that means a hook is the best offence/defense." That line is the design thesis for the whole cluster: move enforcement from prose the model might follow to code that runs regardless.

## Status as of 2026-07-24 (verified via `gh pr view` + `git merge-base --is-ancestor`)

| PR | Title | State | On `main`? |
|----|-------|-------|-----------|
| #62 | memory index tail keep | MERGED (`0ee4c28`) | yes |
| #63 | periodic memory store lint | MERGED (`302dc93`) | yes |
| #64 | memory write gate | MERGED (`bc708f7`, current `main` HEAD) | yes |
| #65 | memory write lock | **OPEN** | **no** — branch `feat/memory-write-lock`, tip `1c09054` |

This is **not** a completed 4-PR cluster. Three of four are live; #65 is implemented and its own tests pass, but it has not merged and none of its guarantees are active on any deployed surface yet. Anything below describing `memoryLock.ts`/`memoryAppend.ts` describes code that exists on a branch, not shipped behaviour — flagged inline, not just here. 521 tests pass on `main` (PRs #62-#64 included); that count does **not** include #65's tests, which run only on its branch.

## PR #62 — truncation direction fix (`0ee4c28`, merged)

`proactive/memoryIndex.ts` caps the injected index at 32 KiB ([[capabilities/memory]]). The prior truncation kept a **head** slice, but `MEMORY.md` is append-ordered — new pointer lines land at the end — so a head-keep silently evicted the **newest** memories and kept the oldest. Now keeps the **tail**:

- Preserves the leading `# Memory Index` heading when present, so a tail slice doesn't drop the file's title.
- Advances **forward** over UTF-8 continuation bytes at the cut point (the head-keep path advanced backward — the opposite operation, not a mirror, because a tail slice must start on a boundary rather than end on one).
- Falls back to the character-boundary cut when a newline-snap would leave an empty (or whitespace-only) tail. This closes a real total-wipe case: a single oversized line, or one whose only newline sits right at end-of-file, could otherwise push the line-snap past `buf.length`, returning the truncation marker and **zero** memory content — worse than the bug it was meant to fix.
- Marker text, current as written: `[MEMORY.md truncated — older entries were dropped, keeping the most recent ${MAX_INDEX_BYTES} bytes; consolidate the index (see prompts/system.md's Memory contract).]` — [[capabilities/memory]] previously quoted a different, pre-#62 marker string; corrected there.

## PR #63 — periodic detection (`302dc93`, merged)

New `proactive/memoryLint.ts`: `lintMemoryStore(dir)` (directory scan) plus exported pure `validateFrontmatter(content, filename)` — the single schema validator, reused unmodified by PR #64's write-time hook rather than a second implementation. Required fields: `name`, `description`, `type` (must be one of `preference | decision | ongoing | reference`), plus a `name`-matches-filename-slug check. A new `date` field (ISO 8601) was added to the Memory contract in `prompts/system.md` by this PR.

**Severity split is deliberate, not an oversight**: missing `name`/`description`/`type`/name-mismatch = `error`; missing `date` = `warning` only. Real files predate the `date` field (the audit's own `units-preference.md` example), so treating its absence as an error would make the lint permanently red from the day it shipped.

Wired as a 6th family in `proactive/sweep.ts`'s 30-minute deterministic tick: `checkMemoryLint` runs `lintMemoryStore(dirname(resolveMemoryPath()))`, dedup-keyed on a hash of the **violation set** (`memoryLintStateHash`) so the same standing violations don't re-alert every tick — only a change in what's wrong pushes again.

## PR #64 — write-time prevention (`bc708f7`, merged, current `main` HEAD)

The security-relevant PR. New `gate/memoryGate.ts`, wired as a **third** `PreToolUse` callback in `rachel.ts`'s hooks array (alongside the existing send gate). Two independent checks:

1. **Untrusted-context lockout.** Gated on `RACHEL_UNTRUSTED_CONTENT`, an env var set **only** in `tasks/inbox-brief-launchd.plist` (verified: `grep` on the plist shows the key/value pair plus an explanatory comment block). When set, any `Write`/`Edit` whose `file_path` resolves inside `~/.rachel/memory/`, or any `Bash` command whose string contains the memory dir path (literal or `~/.rachel/memory` shorthand), is denied outright — `"This run is processing untrusted content ... memory writes are disabled."`
2. **Frontmatter validation.** Any `Write` landing inside the memory dir (excluding `MEMORY.md` itself, the index) is checked with PR #63's `validateFrontmatter`. `error`-level findings deny the write with the specific reasons joined; `warning`-level findings (i.e. missing `date` alone) pass through.

### Threat model

Rachel's memory index is injected **raw** into her system prompt every turn ([[capabilities/memory]]'s deterministic-injection section). The inbox-brief one-shot processes hostile email while holding `Write`. A hostile email that persuades the model to "remember" attacker-supplied text is a **persistent system-prompt compromise on every future turn** — not a one-off bad response, an ongoing one. This is why the lockout is unconditional under `RACHEL_UNTRUSTED_CONTENT` rather than merely warned-about: the frontmatter check alone would not stop well-formed attacker content, only malformed content.

### Three security-fix rounds — the actual shape of how this hardened

Each round was found only after the previous one looked complete and had already passed review/tests. This progression is the durable lesson, not just the changelog:

1. **Round 1 — substring check.** The first implementation used `filePath.includes(MEMORY_DIR)` — a raw string test on a model-supplied path. Bypassed by `.` path segments, doubled separators, and case variants (this filesystem is case-insensitive, confirmed empirically — folding both sides to lowercase was required).
2. **Round 2 — lexical resolve only.** Fixed with `path.resolve()`, which normalises `.`/doubled-separator cases correctly — but `resolve()` is **purely lexical**; it does not follow symlinks. A symlink whose target is the real memory dir lexically resolves to a path *outside* `MEMORY_DIR` while the filesystem's actual write target is *inside* it — still a bypass. Fixed by switching to `realpathSync`, which does resolve symlinks — but the target file doesn't exist yet when a `Write` is gated (it's about to be created), so `realpathSync` on the literal path throws `ENOENT`. `resolveReal()` (verified, `gate/memoryGate.ts`) resolves the nearest *existing* ancestor instead and rejoins the basename.
3. **Round 3 — swallowed non-ENOENT.** `resolveReal`'s catch blocks were initially unconditional, swallowing **every** errno (not just `ENOENT`) into a lexical fallback. An unreadable ancestor directory concealing a memory-dir symlink would then return a wrong-but-valid lexical path that failed the `startsWith` check — i.e. was **allowed**. Fixed: only `ENOENT` is swallowed; any other errno (`EACCES`, `ELOOP`, `ENOTDIR`, ...) is rethrown, on the reasoning that an unresolvable ancestor may be actively hiding a symlink rather than merely absent. The two call sites then diverge deliberately on what a throw means: `isInsideMemoryDirOrThrow` (untrusted-context lockout) lets the throw propagate to the hook's outer catch, which denies by default; `isInsideMemoryDirPermissive` (ordinary frontmatter check, trusted context) catches it and falls back to the pre-symlink-fix lexical comparison, so a resolution failure during Rachel's normal CLI/bridge use doesn't newly start denying legitimate writes.

The theme across all three rounds: **the gap between what a guard inspects (a string) and what the OS actually does (symlink-following path resolution) is where the vulnerability lived, every time.** A string check that agrees with the filesystem's own answer on the happy path can still disagree on the adversarial one.

### Known limitation — explicitly not closed by this gate `(verified)`

`rachel.ts:219` sets `permissionMode: "bypassPermissions"`; `rachel.ts:222` sets `allowedTools` (via `resolveAllowedTools`) but the SDK's separate `tools` option is never set anywhere in the file. `mcp__mcp-exec__execute_code_with_wrappers` — arbitrary code execution — therefore remains reachable in the untrusted one-shot, and **no tool-name-based `PreToolUse` gate can cover it**, because the hook only inspects `Write`/`Edit`/`Bash` tool calls by name; a hostile email could in principle drive memory-store mutation through `mcp-exec` entirely outside the three patterns this hook matches. This is pre-existing and out of scope for PRs #62-#65. PR #64 narrows the injection surface substantially (the ordinary Write/Edit/Bash paths a model would reach for first); **it does not close it.** Do not read this page, or [[capabilities/memory]], as though the hole is sealed.

## PR #65 — concurrency (`1c09054`, **OPEN, unmerged**, branch `feat/memory-write-lock`)

Not yet on `main`. Described here for completeness of the cluster's design, with every claim scoped to "exists on the branch."

New `proactive/memoryLock.ts` — an `O_EXCL` lockfile mutex (`openSync(path, "wx")`, atomically fails if the file exists) — plus `proactive/memoryAppend.ts`, a CLI (`memoryAppend.ts <title> <file> <hook>`) that performs a locked read-modify-write append of one pointer line to `MEMORY.md`.

**Why**: `MEMORY.md` is read-whole-file, append-a-line, write-back across at least three concurrent writer classes — the interactive CLI, the Telegram bridge, and roughly 8 headless one-shots a day (inbox-brief + proactive-calendar). The repo's house idiom for state files (`push.ts`, `sessionPersist.ts`) is atomic temp-file+rename, which gives **durability** (no reader ever sees a half-written file) but not **mutual exclusion** — two concurrent writers can both read the old index, both append their own line, both rename, and the second rename silently discards the first writer's line forever. `memoryLock.ts` is the actual mutex that idiom doesn't provide.

**Staleness handling**: a lock is stale (safe to break) if the holder's pid is no longer alive (`kill -0` probe, same idiom as the bridge's `isPidAlive`) OR the lock is older than `staleMs` regardless of pid liveness — a live-but-wedged holder would otherwise block the store forever. `withMemoryLock` polls at `pollMs` until `timeoutMs`, then throws loud rather than proceeding unlocked — a lost pointer line is a permanently destroyed memory, not a safe-direction loss like `push.ts`'s budget counter.

**Three review findings, all fixed on the branch** (the loop's own recurring pattern: green tests, then a real gap found only in review):

1. **Critical — injection.** The CLI initially interpolated `title`/`file`/`hook` args directly into the fixed-format pointer line (`- [Title](file.md) — hook`). A newline in `title` would write a second, independent line that parses as a legitimate-looking pointer to an arbitrary file — the write-side twin of PR #63's bracketed-title parser risk. `validatePointerArgs` (exported, pure) now rejects `\n`, `\r`, and `[ ] ( )` in `title`/`hook`, and enforces `file` as a bare `*.md` filename with no path separators (closing directory traversal) and none of those characters either.
2. **Critical — first-run hang.** On a fresh install with no `~/.rachel/memory/` directory yet, `mkdirSync` originally ran **inside** the locked callback — but the lockfile itself (`<path>.lock`) lives in that same missing directory, so `acquireMemoryLock`'s `openSync` failed with `ENOENT` before the callback ever ran. The very first memory write on a fresh install busy-polled the full 10s timeout, then failed with a misleading "lock timed out" that hid the real cause. Fixed: `mkdirSync(dirname(path), { recursive: true })` now runs **before** lock acquisition.
3. **Misclassified contention.** A permissions failure during stale-lock reclaim (`EACCES` on the `rmSync` that breaks a stale lock) was being swallowed the same way a normal stale-break `ENOENT` is, which then let the retry `openSync` observe `EEXIST` and busy-poll to a misleading "timed out" message — masking a real permissions problem as ordinary contention. Fixed by rethrowing any non-`ENOENT` errno rather than swallowing it, matching the "only the expected condition is silent" idiom already used in `memoryIndex.ts` and (independently, same session) `gate/memoryGate.ts`'s `resolveReal`.

**Known gap, stated in the branch's own code comments — not yet closed even once #65 merges**: `memoryAppend.ts`'s header states plainly that routing index writes through this CLI is a **prompt contract, not a code-enforced one**. `prompts/system.md` will instruct Rachel to invoke the CLI instead of a freehand `Write` to `MEMORY.md`, but nothing stops a freehand `Write` from bypassing the lock entirely — same trust class as the ad-hoc-backgrounding constraints block elsewhere in this repo. The branch's own comment names PR #64's write-gate hook as "a plausible future place to enforce this in code (block a raw Write to MEMORY.md, require this CLI instead) — not built here since that PR is a sibling, not a dependency." So even after #65 lands, **two unsealed doors remain**: the mcp-exec gap noted under PR #64, and this prompt-only lock-routing contract.

## Durable lessons (not changelog — apply beyond this cluster)

1. **A green suite is evidence about what was tested, never about what was safe.** Three separate times in this cluster a fully green suite (the run counts moved 21/21 → 511/511 → 520/520 as PRs landed) coexisted with a real, live defect, because nobody had yet written a test for that specific shape — the case is *findable in review*, not *automatically caught*.
2. **Two-layer enforcement, deliberately divided, on purpose — neither layer alone is sufficient.** PR #64's hook blocks at write time; PR #63's sweep lint detects periodically. The hook can't see `Edit` results that land via a route it doesn't watch, obfuscated `Bash`, manual edits to the filesystem, or a detached `claude -p` spawn (which loads **no hooks at all** — see [[capabilities/send-gate]]'s own "no PreToolUse hook in a detached spawn" note). The sweep leaves up to a 30-minute detection window. Each covers the other's blind spot; this is presented in the code comments as an intentional split, not a compromise.
3. **The gap between what a guard inspects and what the OS actually does is where path vulnerabilities live.** Every one of PR #64's three bypass rounds was a string-level check disagreeing with filesystem-level resolution (case-folding, lexical-vs-symlink resolution, swallowed errno hiding a real resolution failure). This generalises past this cluster: any security check phrased as a string comparison on a path is a candidate for the same class of bug.
4. **A known limitation, recorded honestly rather than left implicit**: see the PR #64 and PR #65 sections above for the two specific unsealed doors (`mcp-exec`, prompt-only lock routing). Both are pre-existing/deliberately-deferred, not silently missed — but the wiki must not read as though the hole is sealed just because a hardening PR landed nearby.

## Relationships

- [[capabilities/memory]] — the capability page this cluster hardens; updated for the tail-keep truncation direction, corrected marker text, the `date` field, and the two-layer enforcement description
- [[sources/2026-07-21-cross-platform-persistent-memory]] — the original PRs #49-#52 that built the memory store this cluster hardens; that page's truncation description (head-slice, old marker text) is superseded by PR #62, noted there
- [[capabilities/send-gate]] — the sibling `PreToolUse` hook `gate/memoryGate.ts` sits alongside; the "detached `claude -p` loads no hooks" limitation both gates share
- [[capabilities/proactive-layer]] — `proactive/sweep.ts`'s 30-minute tick, now 6 families including `memory-lint`
- [[capabilities/inbox-brief]] — the one-shot that sets `RACHEL_UNTRUSTED_CONTENT`, the concrete trigger for the untrusted-context lockout
