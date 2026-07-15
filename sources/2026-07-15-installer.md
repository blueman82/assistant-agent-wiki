---
tags: [source, installer, deployment, hotfix, launchd]
---

# 2026-07-15 — One-package installer + TDZ hotfix (PRs #30–#32)

Loop 2 of the two-loop proactive envelope (Loop 1: [[sources/2026-07-15-proactive-layer]]). All merged to main and live-verified 2026-07-15 evening.

## PR #30 — Hotfix: push.ts CLI TDZ crash when config.json exists

Found by an independent loop-eval verifier running live probes: `proactive/push.ts`'s top-level `await cliMain()` guard sat above `const HM_RE` — the CLI crashed with a temporal-dead-zone ReferenceError on every run *once a config.json existed*. Latent until then (config-absent runs early-returned); the Loop-2 installer writes config.json, so it would have broken every one-shot on install day. Fix: guard moved to end-of-module (the notify.ts/sweep.ts idiom); class-wide audit confirmed push.ts was the sole instance. Red-green with the exact reported stack.

## PR #31 — scripts/install.sh

The installer itself (see [[capabilities/installation]] for the contract). Gate found two criticals pre-merge, both in the false-PASS class: an unchecked non-atomic config write (a truncated file would then be protected forever by never-overwrite) and heartbeat freshness anchored at script start (the *outgoing* bridge's dying heartbeat satisfied verification). Both fixed with red-green tests before merge; 6/6 frozen PR evals GO, re-graded in full at the fixed head.

## PR #32 — the live-found launchd race

First real install run failed honestly: bridge `bootstrap` exit 5 EIO immediately after `bootout`, because bootout returns while the old job drains. The failure accounting worked exactly as designed (exit 1, every casualty named, other services unaffected). Fix: post-bootout teardown-wait + single retry + post-teardown heartbeat epoch; 4 red-green tests against the actual unfixed installer via an upgraded launchctl shim modelling the linger.

## Live verification (loop eval E9)

A blind grader ran its own clean-slate cycle: unloaded three services, deleted their plists, ran the negative control (liveness check fails pre-install), then one real `./scripts/install.sh` — exit 0, genuine fresh-install branches, new bridge PID polling with an advancing heartbeat, sweep `runs = 1 / exit 0` in launchd's own accounting, and the user's config byte-identical (md5-pinned). Loop-scope evals finished 9/9; `grade-loop` returned GO.
