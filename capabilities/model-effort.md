---
title: "Model & Effort Switching"
type: capability
created: 2026-07-24
last_updated: 2026-07-24
sources: ["proactive/modelConfig.ts", "rachel.ts", "bridge/telegram-bridge.ts", "CLAUDE.md"]
tags: [capability, model-config, cli, telegram-commands]
---

## What it does

`proactive/modelConfig.ts` lets Gary switch which Claude model and reasoning-effort level Rachel runs at, at runtime, via `/model [name]` and `/effort [level]` — from the terminal REPL or Telegram, with no restart required. A switch takes effect on the very next turn: `rachel.ts`'s `runTurn` rebuilds its `options` object every turn and reads `getModel()`/`getEffort()` fresh each time, rather than capturing a boot-time constant.

Valid models: `claude-sonnet-5`, `claude-opus-4-8`, `claude-haiku-4-5`, `claude-fable-5` (aliases: `sonnet`, `opus`, `haiku`, `fable`, case-insensitive). Valid efforts: `low`, `medium`, `high`, `xhigh`, `max` — all five levels are valid for all four models (the SDK's own type docs mark `xhigh`/`max` as restricted to specific older Opus versions; that annotation is stale against current model docs and is deliberately not enforced here).

## How it works

- **Module-level mutable state, not a config file.** `currentModel`/`currentEffort` are plain `let` bindings in `proactive/modelConfig.ts`. `getModel()`/`getEffort()` read them; `setModel()`/`setEffort()` write them after validating against the whitelist.
- **Boot-time default resolution**: `RACHEL_MODEL` env var (default `claude-sonnet-5` if unset or invalid — an invalid value logs to stderr and falls back rather than crashing). Effort always boots to `high`, with no env override.
- **Setters never throw.** Unlike `proactive/allowedTools.ts` (which throws when an operator's env var narrows the tool list to zero), a bad `/model`/`/effort` value returns a discriminated `{ ok: false, message }` result instead — this module always has a safe fallback (the current value stays unchanged), so throwing here would turn an operator's typo into a crash under the Telegram bridge's `KeepAlive` supervision (a crash-loop, not a fail-loud).
- **`handleConfigCommand`** is the single dispatch point both the terminal REPL and the Telegram bridge call — one parsing/validation/rendering path, not two copies. No-argument form (`/model`, `/effort`) reports the current value plus valid options; with an argument, it validates and switches.
- **`parseArgvConfig`** handles the one-shot CLI path (`rachel /model opus "check my email"`) — walks argv token-by-token, applies every `/model`/`/effort` command it finds via `handleConfigCommand`, and rejoins whatever tokens remain (in original order) as the prompt. Config and prompt tokens compose rather than being mutually exclusive.
- **Wired into `rachel.ts`**: `getModel()`/`getEffort()` are read fresh into the `options` object passed to the SDK's `query()` call on every turn (`rachel.ts:216-217`).
- **The Telegram `/status` staleness risk flagged in [[investigations/2026-07-16-model-effort-switching-assumptions]] was closed in the shipped code**: `bridge/telegram-bridge.ts:560` reads the live `getModel()` getter, not `process.env["RACHEL_MODEL"]` directly — a `/model` switch is correctly reflected in a subsequent `/status` call.

## Constraints / gotchas

- **Per-process, not shared.** This state lives in one OS process's memory. The terminal REPL and the Telegram bridge are separate processes, each importing its own fresh copy of `proactive/modelConfig.ts` — a `/model` switch in one is invisible to the other, and invisible to any headless one-shot spawned separately. This is a deliberate design choice, not a gap: persisting the choice to disk so it was shared across processes would let an interactive switch silently change which model unattended scheduled jobs run on.
- **Not persisted across a restart.** A bridge restart (deploy, crash-recovery) resets to the `RACHEL_MODEL` boot default, losing any live `/model`/`/effort` switch made in that process. See [[capabilities/telegram-frontend]] for the separate `RACHEL_SESSION_FILE` seam that *does* persist session identity across a restart — model/effort choice is not part of what that seam carries.
- **`--help`/`-h` and the help text (`renderHelp`) render the model/effort lists from this module's own exports**, not a duplicated literal list, so a whitelist change can't drift out of sync with the CLI help output.

## Relationships

- [[architecture/overview]] — where `rachel.ts`'s per-turn `options` construction lives
- [[capabilities/telegram-frontend]] — the other surface `handleConfigCommand` serves; contrast its `RACHEL_SESSION_FILE` persistence (which this capability does not share)
- [[capabilities/proactive-layer]] — `proactive/allowedTools.ts`, the sibling seam this module's error-handling deliberately diverges from (throw vs. safe-fallback), and why
