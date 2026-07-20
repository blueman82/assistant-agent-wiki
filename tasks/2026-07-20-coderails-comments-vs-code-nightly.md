---
title: "Coderails: comments vs code nightly routine"
slug: comments-vs-code-nightly
status: backlog
priority: medium
repo: "[coderails]"
due: null
---

## Overview

Build a nightly routine that checks comment quality vs code quality across all local cloned codebases. Use an external verifier to audit comment-code alignment.

## Scope

- Create a nightly checker script that runs against opted-in codebases
- Opt-in flag: add to workflow.config.yaml template
- External verifier: design interface (command, output format)
- Install/uninstall bash scripts: bundle with template
- Report generation: findings logged or pushed

## Acceptance Criteria

- Nightly routine created and scheduled (coordinates with task 5)
- workflow.config.yaml updated with opt-in flag
- Install script: adds verifier, sets up launchd/cron trigger
- Uninstall script: cleans up trigger and verifier
- External verifier called correctly (stdin: file content, stdout: JSON verdict)
- At least one test codebase opted in and verified working

## Dependencies

- Task 5: loop scheduling to nightly (this IS one of the nightly loops)

## Notes

Original task (lines 21-22 of tasks_to_do.md): "create a comments versus code routine with external verifier - this will run nightly across all local cloned codebases that have an opt in flag on workflow.config.yaml file must be part of template for the yaml file and install bash script and uninstall bash script"
