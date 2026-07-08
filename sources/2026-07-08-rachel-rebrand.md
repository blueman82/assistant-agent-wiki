---
title: "Rachel rebrand"
type: source
origin: "PR #15 feat/rachel-rebrand (assistant-agent), merged 3b871f7"
created: 2026-07-08
last_updated: 2026-07-08
sources: ["CLAUDE.md", "README.md", "AGENTS.md", "rachel.ts", "prompts/system.md", "bridge/launchd.plist"]
tags: [rebrand, naming, decision]
---

## Key takeaways

- The "secretary" branding was retired codebase-wide and replaced with the name **Rachel**. This is a rename, not a behaviour change — plumbing/brain split, tool routing, and the send gate all work exactly as before.
- `secretary.ts` → `rachel.ts`. The inline SDK agent name changed from `secretary` to `rachel`.
- Env vars: `SECRETARY_MODEL` → `RACHEL_MODEL`, `SECRETARY_MAX_TURNS` → `RACHEL_MAX_TURNS`.
- Config/log directory: `~/.secretary/` → `~/.rachel/` (send-gate audit log, Telegram bridge log, Telegram token/chat-id config).
- launchd service label: `com.secretary.telegram-bridge` → `com.rachel.telegram-bridge` (`bridge/launchd.plist`).
- Launcher symlink: `bin/secretary` → `bin/rachel`.
- The npm package name **stays `assistant-agent`** — that did not change. Only the agent's own name/branding did.

## Live cutover (this session)

The launchd bridge service was cut over live: unloaded the old `com.secretary.telegram-bridge` label, loaded the new `com.rachel.telegram-bridge` from `~/.rachel/` config. The `~/bin/secretary` symlink is left dangling — Gary will recreate it as `~/bin/rachel` himself.

## Impact

Updated for the rename (prose/paths only, no factual/architectural changes): [[architecture/overview]], [[architecture/mcp-integrations]], [[capabilities/send-gate]], [[capabilities/slack]], [[capabilities/tasks]], [[capabilities/telegram-frontend]], [[patterns/extending-system-md]], [[sources/2026-06-29-wire-in-slack]], [[index]].

`log.md` entries before this date describe historical state and were left as-is (append-only convention) — they still say "secretary" because that was the name at the time.
