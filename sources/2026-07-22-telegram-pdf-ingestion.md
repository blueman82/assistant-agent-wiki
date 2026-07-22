---
title: "PR #54 — Telegram PDF Ingestion"
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources: ["bridge/telegram-bridge.ts", "bridge/telegram-bridge.test.ts", "prompts/system.md", "CLAUDE.md"]
tags: [telegram, bridge, pdf, document, capability, bugfix]
---

## Origin

PR #54 (`feature/telegram-pdf-ingestion`) — merged 2026-07-22, merge commit `6e96a41`.
Files changed: `CLAUDE.md`, `bridge/telegram-bridge.ts`, `bridge/telegram-bridge.test.ts`, `prompts/system.md`.

A same-day post-review follow-up, commit `197a14f` ("Fix dot-less PDF filename saving with wrong extension"), fixed a bug found by `code-reviewer` during PR #54's own review pass — see below.

## What it adds

Rachel can now receive PDF documents sent by Gary via Telegram, alongside the existing photo/image-document path from PR #17 ([[sources/2026-07-08-telegram-image-reception]]). Gary explicitly asked for this ("I need you to be able to read PDF files"), which reverses PR #17's earlier decision to drop PDF support as unasked-for scope — see that page's supersession note.

Rachel's `Read` tool already handles PDFs natively (it doesn't need this bridge change to read a PDF already on disk); the gap this PR closes is purely on the ingestion side — the bridge previously rejected any non-image document outright.

## Key decisions

**Extend the document handler, don't fork a new one**: the existing `msg.photo || msg.document` branch in `bridge/telegram-bridge.ts` gained a boolean, `isPdf = mime === "application/pdf"` (line 636), computed alongside the existing `image/*` check. Download and temp-file mechanics (`downloadFileFn` seam, `~/.rachel/tmp/<fileId>.<ext>` path, 20 MB `getFile` guard) are unchanged and shared between images and PDFs.

**Distinct FIFO tag, not a shared one**: a PDF is queued to `runTurn` as `[document: /path]`, not `[image: /path]`. `const tag = isPdf ? "document" : "image"` (line 655) drives both the FIFO push and the download-failure reply text. The rationale: a PDF isn't consumed via vision, so instructing Rachel to "look at this image" would be misleading — the contract is "call `Read` on this path", which for a PDF invokes the tool's native PDF-reading path rather than an image-viewing one.

**Non-image, non-PDF documents**: reply text updated from "I can only receive images." to "I can only receive images or PDFs. Try sending a JPEG, PNG, or PDF." — same immediate-reply-not-silent-drop behaviour as PR #17.

**system.md contract**: a new "Receiving PDFs from Telegram" section mirrors the existing "Receiving images from Telegram" section — instructs Rachel to call `Read` on a `[document: <path>]` message before responding.

**CLAUDE.md**: the architecture description of `bridge/telegram-bridge.ts` was updated to say it "handles inbound photo, image-document, and PDF-document messages" and documents both the `[image: ...]` and `[document: ...]` tags.

## Post-review bugfix (commit `197a14f`)

**Bug**: the `file_name` extension-derivation fallback was hardcoded to `"jpg"` for any document arriving with a dot-less filename (e.g. literally named `invoice`, no extension). For a PDF this meant the file was downloaded and saved as `<fileId>.jpg` — PDF bytes under an image extension — which would have made Rachel's subsequent `Read` call fail to decode it as either format.

**Fix**: `ext = dot >= 0 ? msg.document.file_name.slice(dot + 1) : (isPdf ? "pdf" : "jpg")` — the fallback now checks `isPdf` instead of assuming image.

**Also in this commit**: hoisted the repeated `isPdf ? "document" : "image"` ternary into a single `tag` const (`code-simplifier`'s review suggestion on PR #54), used consistently for the FIFO tag, the download-failure log line, and the failure reply text.

**Lesson**: any extension-derivation fallback that assumes a single content type (here, "no dot means image") breaks silently the moment a second content type is added to the same code path. The fix generalizes correctly — it keys off the MIME-derived `isPdf` flag rather than special-casing PDF filenames — but the bug shows the fallback should have been type-aware from the start of the *first* multi-type document handler, not patched in after the fact.

## Verification (per PR body, not independently re-run this wiki session)

`npm run typecheck` clean; `npm test` green (`bridge/telegram-bridge.test.ts` gained 230 lines of new coverage across the PR + bugfix commits, per `git show --stat`).

## Known debt — pre-existing, not introduced or fixed by this PR

Both gaps below were already true for the image path (see [[sources/2026-07-08-telegram-image-reception]]'s Known debt) and now apply equally to PDFs, since the download/temp-file code is fully shared:

- No file-size cap on the download itself, beyond Telegram's own 20 MB `getFile` API ceiling.
- `~/.rachel/tmp/` temp files (image or PDF) are never cleaned up — the directory grows unboundedly.

Neither is this PR's scope to fix; noted here only so a future reader doesn't mistake PDF ingestion as the origin of either gap.

## Relationships

- [[capabilities/telegram-frontend]] — the capability this PR extended (see "Image and PDF reception" section)
- [[sources/2026-07-08-telegram-image-reception]] — PR #17: the prior image-only implementation, and the "PDF scope dropped" decision this PR reverses by explicit later request
- [[architecture/overview]] — `bridge/telegram-bridge.ts`'s role in the plumbing layer
