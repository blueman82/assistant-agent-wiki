---
title: "Coderails: judge automation on/off"
slug: judge-automation-toggle
status: backlog
priority: high
repo: "[coderails]"
due: null
---

## Overview

Create automated mechanisms to turn judge automation ON and OFF. Currently judge is controlled via manual launchctl commands; this task builds scripts to toggle state reliably.

## Scope

### Judge Automation Mechanics (verified from coderails exploration)

- **ON**: `sudo install.sh` in `/scripts/tier-gate/` installs launchd daemon plist, sets it to run every 300 seconds, requires credentials at `/etc/coderails-tier-gate/credentials`
- **OFF**: manual `sudo launchctl bootout system/com.coderails.tier-gate` + delete plist and files
- **Current state**: not running (verified — launchctl list shows absent)

### Tasks

1. **Build judge-on script** — automate: render plist template, install, set up credentials, bootstrap daemon
2. **Build judge-off script** — automate: launchctl bootout, delete plist, revoke tokens/permissions (with manual approval step before removal)
3. **Integrate into CI/CD** — scripts callable from GitHub Actions or CLI

## Acceptance Criteria

- judge-on.sh: idempotent, prompts for credentials, verifies daemon running post-install
- judge-off.sh: prompts for token/permission revocation approval, then removes (manual-then-script pattern)
- Both scripts documented and tested
- judge status queryable via new `judge-status` command

## Dependencies

- Judge already installed/functional (verified existing)

## Notes

Original task (lines 24-25 of tasks_to_do.md): "turning off the judge architecture (automated apart from token in github which is human but a script can build in exact steps required for that and 'hold' until human confirms token/permission removals and then script proceeds to verify as such."

Extension: also build judge-on automation, not just off.
