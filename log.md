# Log

## [2026-06-07] init | bootstrapped wiki with 5 seed pages

## [2026-06-29] ingest | wire-in-slack: new capabilities/slack + sources/2026-06-29-wire-in-slack; updated mcp-integrations, extending-system-md, index

## [2026-06-29] edit | scrubbed dead .env.ketchup references from slack pages (token file deleted, slack skill rewritten to MCP)

## [2026-07-07] ingest | new capabilities/send-gate documenting the PreToolUse enforcement hook (secretary.ts + gate/sendGate.ts); updated index. wiki-lint pass: no contradictions, no orphans, no stale (90+ day) pages found

## [2026-07-07] ingest | Telegram front-end post-merge docs pass (assistant-agent PR #9, docs/telegram-frontend, merged 7e885ad): new capabilities/telegram-frontend documenting bridge/telegram-bridge.ts (owns the single getUpdates loop, FIFO chat queue, single-user gating, /reset /status /stop bridge commands, launchd persistence); updated capabilities/send-gate with the update-routing split (bridge owns getUpdates, gate/surfaces/telegram.ts only consumes injected callback_query updates); updated index

## [2026-07-08] ingest | Telegram reply formatting fix (assistant-agent PR #14, fix/telegram-reply-formatting, merged 4a70688): updated capabilities/telegram-frontend with the stripMarkdown sanitiser (bridge/api.ts, wired into sendChunked pre-chunk) and the new prompts/system.md "Plain text replies" ground rule; documented the parse_mode/JSON-envelope rejection rationale (malformed entity → HTTP 400 → lost message; deterministic stripping can't lose a message); new sources/2026-07-08-telegram-reply-formatting.md. Lint pass: no contradictions found (grep for markdown/parse_mode/plain text across capabilities+architecture+patterns turned up only unrelated hits — tasks.md's own file-format note and extending-system-md's description of prompts/system.md as a markdown file)

## [2026-07-08] ingest | Rachel rebrand (assistant-agent PR #15, feat/rachel-rebrand, merged 3b871f7): secretary → Rachel rename, naming only, no behaviour change. Updated architecture/overview, architecture/mcp-integrations, capabilities/send-gate, capabilities/slack, capabilities/tasks, capabilities/telegram-frontend, patterns/extending-system-md, sources/2026-06-29-wire-in-slack, index — secretary.ts→rachel.ts, SECRETARY_*→RACHEL_*, ~/.secretary/→~/.rachel/, com.secretary.telegram-bridge→com.rachel.telegram-bridge; new sources/2026-07-08-rachel-rebrand.md. npm package name "assistant-agent" unchanged, left alone. Historical entries above this line still say "secretary" — append-only log, left as-is, accurate to the time they were written. Lint pass: post-rename grep for "secretary" (case-insensitive, excluding raw/ and log.md's own history) returns zero hits; "Rachel" appears in all 9 touched pages; no orphans (new source page linked from index); no dead links introduced
