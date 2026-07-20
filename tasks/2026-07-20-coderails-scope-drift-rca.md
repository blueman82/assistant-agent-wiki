---
title: "Coderails: RCA on scope drift + build RCA routine"
slug: scope-drift-rca-build-routine
status: backlog
priority: high
repo: "[coderails]"
due: null
---

## Overview

Post-mortem on scope-drift incident from session 7304590f, followed by building a systematic RCA routine to prevent recurrence.

## Part 1: RCA on Scope Drift Incident

Session: `7304590f-db26-4c6a-b268-a1148d262f8f`

### Root Causes (verified from tasks_to_do.md lines 9-11)

- **evals.json — absent.** No frozen loop-scope success criteria. Agent graded work against a definition that moved as implementation progressed. Same oracle-sharing failure flagged in two other agents.

- **spec.md / plan.md — absent.** Scope lived only in conversation, causing drift when corrections arrived.

- **proof.json — absent (critical).** Agent set `proof_disposition: "none: no executable end-state reachable"` when it believed nothing could merge. That assumption became false when #248 merged and daemon began judging. Agent never revisited proof.json. A gate designed to force executable proof was satisfied by a stale excuse.

### Acceptance Criteria (Part 1)

- Root causes documented in detail
- Session 7304590f replayed to extract exact failure points
- Three sub-causes mapped to prevention measures

## Part 2: Build RCA Routine

Design and implement a systematic process to detect and prevent scope drift before it locks gates.

### Routine Scope

- **Trigger**: agentic loop detection (scope change, gate re-evaluation, proof assumption invalidation)
- **Automated checks**: evals.json present, spec.md current, plan.md aligned
- **Gate**: refuse to proceed if checks fail (hard stop, not warning)
- **Escalation**: alert on scope drift detected mid-loop

### Acceptance Criteria (Part 2)

- RCA routine implemented in coderails
- Integrated into agentic-loop flow (PreLoop hook or equivalent)
- Tests cover: missing evals, stale spec, invalidated proof assumption
- Documentation written

## Dependencies

- Agentic-loop architecture (task 3)

## Notes

This is two tasks merged: RCA analysis + prevention mechanism build.
