# Log

## [2026-06-07] init | bootstrapped wiki with 5 seed pages

## [2026-06-29] ingest | wire-in-slack: new capabilities/slack + sources/2026-06-29-wire-in-slack; updated mcp-integrations, extending-system-md, index

## [2026-06-29] edit | scrubbed dead .env.ketchup references from slack pages (token file deleted, slack skill rewritten to MCP)

## [2026-07-07] ingest | new capabilities/send-gate documenting the PreToolUse enforcement hook (secretary.ts + gate/sendGate.ts); updated index. wiki-lint pass: no contradictions, no orphans, no stale (90+ day) pages found

## [2026-07-07] ingest | Telegram front-end post-merge docs pass (assistant-agent PR #9, docs/telegram-frontend, merged 7e885ad): new capabilities/telegram-frontend documenting bridge/telegram-bridge.ts (owns the single getUpdates loop, FIFO chat queue, single-user gating, /reset /status /stop bridge commands, launchd persistence); updated capabilities/send-gate with the update-routing split (bridge owns getUpdates, gate/surfaces/telegram.ts only consumes injected callback_query updates); updated index
