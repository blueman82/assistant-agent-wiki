---
title: Ad-hoc Background Escalation
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources: ["prompts/system.md", "bridge/telegram-bridge.ts", "bridge/telegram-bridge.test.ts"]
tags: [telegram, bridge, loop-launcher, backgrounding, turn-timeout, prompt-injection]
---

## Summary

PR #56 (`0ee4936`, merged 2026-07-22) gave Rachel a way to move a spontaneous, long-running
request out of the Telegram bridge's inline turn and into a detached background loop, without
a pre-written `tasks/launch-*.md` file. The pre-existing loop launcher only serves task files
Gary wrote in advance; this closes the gap for requests made mid-conversation.

## Problem

`bridge/telegram-bridge.ts`'s `DEFAULT_TURN_TIMEOUT_MS` aborts any single turn past 10 minutes.
The timeout exists because `drainFifo` is single-flight — one hung turn would wedge the whole
message queue silently otherwise. It had fired 9 times in ~1.5 days. A long spontaneous request
had no escape: raising the timeout was the obvious lever, and it was rejected.

## Key Decisions

| Decision | Why |
|----------|-----|
| Raise the timeout (option A) | **Rejected.** The bridge log carries no per-line durations, so no specific bump number was defensible from evidence. The timeout is also the wedge detector for a hung upstream call — every added minute is added wedge time before a hang is even noticed. |
| Explicit "background this" escalation only (option B) | **Chosen.** Gary (or Rachel, via auto-suggest) explicitly moves the work out-of-band; the 10-minute ceiling stays untouched as the wedge detector. |
| Both A and B (option C) | Not needed once B covers the actual gap. |
| Auto-detection (Rachel silently decides to background) | **Rejected** — a silent wrong guess launches a `bypassPermissions` agent with no operator awareness. |
| Auto-suggest (Rachel proposes backgrounding, never spawns unasked) | **Ruled in** — fails visibly (a suggestion ignored is just a suggestion ignored), unlike auto-detect. |

## What Shipped

**`prompts/system.md`** — new "Ad-hoc backgrounding" section, immediately after the existing
loop-launcher section (see [[capabilities/tasks]] for that launcher's `launch-*.md` frontmatter
shape, which this reuses).

- **Triggers**: "background this", "run that as a loop", "don't do this inline", or accepting
  Rachel's own auto-suggest. Auto-suggest fires before starting work judged likely to run past
  10 minutes — one line, never a silent spawn.
- **Task-file synthesis**: `tasks/adhoc-YYYY-MM-DD-<slug>.md` — the `adhoc-` prefix keeps these
  out of the `launch-*.md` glob. Same launcher frontmatter (`title`, `slug`, `repo`,
  `permission_mode: bypassPermissions`, `status: launchable`) plus a `report:` path field.
  Body has three parts in order: (1) Gary's triggering words, verbatim, headed as authoritative
  — if the words aren't in session context (e.g. lost with an aborted turn), Rachel asks him to
  restate rather than paraphrasing from memory; (2) Rachel's own brief — goal, done-criteria,
  file paths, in-session facts; (3) a fixed constraints block, identical in every synthesised
  file.
- **Confirm-then-spawn**: Rachel replies with slug + one-paragraph brief and waits for "go",
  unless Gary says "background this, just go".
- **Concurrency**: reuses the loop launcher's `checkLaunchAllowed` check. Non-repo work gets a
  scratch dir (`~/.rachel/loops/<slug>-work/`) created before spawning, so the check passes by
  construction. Repo-mutating work spawns in-place if the check passes, or offers stopping the
  running loop / spawning in a fresh worktree if it's blocked.
- **Spawn line**: loop-launcher steps unchanged, plus `--disallowedTools` naming the 5 gated
  send tools (Slack send, the 4 Calendar mutators), the 3 `mcp-exec` tools (denied because
  mcp-exec can proxy other MCP servers), and `Bash(curl *)`/`Bash(wget *)`.
- **Post-spawn liveness check**: wait ~2s, `ps -p <pid> -o command=`, confirm it contains
  `claude` before reporting success — a pid is assigned on fork/exec regardless of whether the
  process then dies in its first second.
- **Quiet-hours/budget warning**: if spawning within ~2h of quiet hours (22:30 Europe/Dublin) or
  near the daily interrupt budget, Rachel names the log path as the interim source of truth in
  the same reply.
- **Mid-turn escalation**: an in-flight turn can't be transplanted into the background (live SDK
  stream, no way to serialise half-finished work). Protocol is `/stop`, then "background that" —
  `/stop` aborts immediately, the follow-up starts a fresh turn that synthesises the task file as
  usual. Work already done inline is lost except what reached disk or the transcript.
- **Aftermath**: on a watchdog exit/quiet ping, Rachel doesn't act until Gary asks; then she
  reads the report file, relays it, and flips the task file's `status` to `done`.

**`bridge/telegram-bridge.ts`** (+13/-1) — the timeout cut-off message now offers backgrounding
as an alternative to re-asking. Added `turnStartedMs = Date.now()` and a three-way branch on
turn outcome: timed-out (existing message, now with the backgrounding offer appended), errored
(`turnErrored` flag set in the catch block; logs `turn failed after <ms>ms`, excluded from the
completed-duration log so a crash doesn't contaminate that data), or completed normally (logs
`turn completed in <ms>ms`). See [[capabilities/telegram-frontend]] for the constant this
instruments.

**`bridge/telegram-bridge.test.ts`** (+124) — three new tests: happy-path duration log,
errored-turn excluded from the completed-log path, timed-out-turn excluded from the
completed-log path.

## Findings With Lasting Design Significance

Five Critical review findings were fixed pre-merge. Recorded here because they constrain how
this feature (and anything similar) can safely evolve:

1. **Frontmatter never reaches a spawned agent.** The launcher contract passes only the task
   file *body* to `claude -p` (`system.md:117`) — a `report:` path in frontmatter is invisible
   to the spawned process. This is why the constraints block spells out the literal report path
   in the body rather than relying on frontmatter. Any future task-file convention must put
   agent-facing values in the body, not frontmatter.
2. **A detached `claude -p` does not load the send gate.** `gate/sendGate.ts` is a `PreToolUse`
   hook wired inside `rachel.ts`'s own `query()` (`rachel.ts:224-232`), so
   `gate/bashPatterns.ts`'s `matchesBashSendPattern` (reachable only from `sendGate.ts:87`) is
   unreachable too. Background agents have unrestricted Bash with no send-pattern check;
   `--disallowedTools` only closes the MCP surface, not arbitrary Bash. This hole is
   **pre-existing** — the loop launcher's own `launch-*.md` spawns already run
   `bypassPermissions` with zero tool denials — but the new risk is that ad-hoc bodies are
   synthesised from session context that may contain untrusted text (an email, a web page), so
   prompt injection can reach a `bypassPermissions` agent. Mitigated with a content fence around
   quoted/pasted material, `Bash(curl *)`/`Bash(wget *)` denials, and an explicit note that the
   constraints-block prose is advisory, not enforced. See [[capabilities/send-gate]].
3. **Loop-exit pings are `severity: "normal"`** (`telegram-bridge.ts:829`), so quiet hours
   (22:30–08:00 Europe/Dublin) and the daily interrupt budget both defer them (see
   [[capabilities/proactive-layer]]) — a background job crashing at 22:35 may not be reported
   until the morning digest. Hence the spawn-time quiet-hours/budget warning above.
4. **A pid proves a process was created, not that it survived.** A bad flag, auth failure, or
   bad arg can crash the process in its first second; at fork/exec time that looks identical to
   a healthy launch. Hence the post-spawn `ps -p` re-check before reporting success.
5. **`nowFn()` is for timestamps, `Date.now()` is for elapsed time.** The bridge test suite's
   clock seams (`DAYTIME`/`QUIET_TIME`) are frozen constants — routing turn duration through
   `nowFn()` would report 0ms for every turn and make the new instrument useless. Recorded
   because it looks like an inconsistency a future reader might "correct" otherwise; it isn't.

## Related Pages

- [[capabilities/telegram-frontend]] — the 10-minute turn timeout this escapes, and the new
  duration-logging/cut-off-message changes
- [[capabilities/tasks]] — the `launch-*.md` frontmatter shape reused (with the `adhoc-` prefix
  and `report:` field variant) and the frontmatter-not-passed-to-spawn finding
- [[capabilities/send-gate]] — why a detached `claude -p` doesn't load the gate, and what
  `--disallowedTools` does and doesn't close
- [[capabilities/proactive-layer]] — quiet hours and the daily interrupt budget that can defer a
  loop-exit ping

## PR Details

**PR #56**: adhoc background escalation
- Modified: `prompts/system.md` (+29), `bridge/telegram-bridge.ts` (+13/-1)
- Added: `bridge/telegram-bridge.test.ts` (+124, three new tests)
- Merged: 2026-07-22 (commit `0ee4936`)
