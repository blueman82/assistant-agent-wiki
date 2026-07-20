---
title: Installation — the one-package installer
type: capability
created: 2026-07-15
last_updated: 2026-07-15
sources: [scripts/install.sh]
tags: [capability, installation, deployment, launchd, installer]
---

# Installation — the one-package installer

One command takes a machine from a bare checkout to fully-deployed Rachel:

```
./scripts/install.sh
```

Shipped in PRs #30–#32 (merged 2026-07-15, live-verified the same evening by an independent clean-slate cycle — services unloaded, plists deleted, one installer run, everything back and polling). Replaces every manual copy-the-plist/`launchctl load` instruction that previously lived in CLAUDE.md and this wiki.

## What it does, in order

1. **Preflight** — `node`/`node_modules` present; Telegram config valid via a `node -e` check that mirrors `loadTelegramConfig` exactly (env pair `RACHEL_TELEGRAM_TOKEN`/`RACHEL_TELEGRAM_CHAT_ID`, or `~/.rachel/telegram.json`). Missing config fails loud with instructions for both routes; the installer never writes credentials itself.
2. **Files** — stamps `__REPO_PATH__` into all four launchd templates (labels read from each template at runtime via `plutil -extract`), lint-checks the stamped output on a temp file, then atomically renames into `~/Library/LaunchAgents`. Bootstraps `~/.rachel/proactive/config.json` with the documented defaults if absent — an existing file is NEVER overwritten (atomic temp+mv either way).
3. **Services** — for each of `com.rachel.telegram-bridge`, `com.rachel.inbox-brief`, `com.rachel.proactive-sweep`, `com.rachel.proactive-calendar`: `bootout` (exit 3 tolerated on a fresh machine), then **wait out the teardown** (see the race below), then `bootstrap` with one bounded retry on exit 5. A failing service is accounted and skipped, never a mid-run abort — the loop completes and the summary names every casualty.
4. **Verification** — truthful PASS/FAIL summary: all four services loaded, bridge running, proactive config parses, and the bridge heartbeat file freshly written after the old process's teardown completed (up to 120s wait, covering the 409-backoff window). Exit 0 only when everything passed; any failure names the check and exits 1.

`--dry-run` prints the complete plan (every file write, every launchctl call) with zero side effects, then reports any preflight problems.

Test seams (used by the installer's own 25-test suite, never needed in normal use): `INSTALL_HOME`, `INSTALL_LAUNCH_AGENTS_DIR`, `INSTALL_LAUNCHCTL`, `INSTALL_HEARTBEAT_WAIT_SECS`, `INSTALL_TEARDOWN_WAIT_SECS`.

## The launchd teardown race — a reusable macOS ops gotcha

`launchctl bootout` is **asynchronous**: it returns while the old job is still draining (the bridge holds a 30s `getUpdates` long-poll). An immediate `bootstrap` of the same label then fails with exit 5 "Input/output error". This bit the first live install run — which failed *honestly* (exit 1, all three casualties named; a manual bootstrap seconds later succeeded, proving the race). PR #32's fix: after bootout, poll `launchctl print` until the label is gone (30s bound) before bootstrapping, with one 2s retry as a backstop. The heartbeat-freshness epoch is captured only after teardown completes, so a dying process's final heartbeat write can never satisfy verification for its replacement.

If you script launchd service replacement anywhere else: never `bootstrap` immediately after `bootout` returns — wait for the label to actually disappear.

## Related

- [[capabilities/telegram-frontend]] — the bridge the installer deploys
- [[capabilities/proactive-layer]] — the sweep/one-shot services and their config
- [[sources/2026-07-15-installer]] — cluster source page (PRs #30–#32, live E9 verification)
