---
title: "Coderails: hook efficiency + token measurement"
slug: hook-efficiency-token-measurement
status: backlog
priority: high
repo: "[coderails]"
due: null
---

## Overview

Fix token waste from confidence-label hook blocks, implement steering-based hook strategy, and measure token savings from all 7 token-reduction measures introduced to date.

## Part 1: Fix Confidence-Label Hook Efficiency

### Current Problem (verified from tasks_to_do.md lines 27-28)

Stop hook `check_confidence_labels.sh` blocks responses ≥200 chars with no labels, forcing retry:
1. Claude outputs response (tokens spent)
2. Hook blocks it
3. Claude repeats output with labels (tokens re-spent)
Result: token duplication on every block

### Solution

Replace Stop hook block with PreToolUse steering hook (pattern at tasks_to_do.md lines 29-44). Example from Scout MCP:
```json
"PreToolUse": [{
  "matcher": "Task|Agent",
  "hooks": [{
    "type": "command",
    "command": "INSTR=$(...); jq --arg instr \"$INSTR\" -c '{ hookSpecificOutput: { ... updatedInput: (.tool_input + { prompt: (\"# Scout MCP...\") + $INSTR + \"---\" + .tool_input.prompt) } }'"
  }]
}]
```

This injects guidance into the prompt BEFORE execution, not after. No retry loop.

### Acceptance Criteria (Part 1)

- PreToolUse hook designed and tested
- Replaces Stop hook for confidence labels
- No forced retries observed in test runs
- Token usage measured before/after

## Part 2: Verify Token Cost of Current Approach

### Investigation

Verify (unverifiable — requires Anthropic API access): are we burning costs or just hitting cache?

- Current block pattern: does it hit cache on retry (cost: tokens only, not dollars)?
- Proposed steering: does it still hit cache?
- Report findings to Gary

### Acceptance Criteria (Part 2)

- Cache hit/miss behavior measured
- Decision made: worth fixing now or defer?

## Part 3: Measure Token Savings from 7 Reduction Measures

### Scope

Seven token-reduction measures were introduced (verified — line 48 of tasks_to_do.md). Measure their actual impact:

1. Are savings being made?
2. How are they being reported/measured/studied?
3. If not at all: design observability stack

### Measures to Audit

(Inferred from context; verify against coderails codebase):
- Prompt caching
- Confidence label optimization
- Memory consolidation
- Stop hook efficiency
- Pre-computation steps
- Draft-first (reduces refactoring tokens)
- Skill narrowing (allowed-tools per task)

### Acceptance Criteria (Part 3)

- Audit complete: which 7 measures are in place
- Measurement system designed (what metric, how often, where stored)
- Observability dashboard created (or file + script to extract data)
- Weekly/monthly report structure defined

## Dependencies

- None

## Notes

Three sub-tasks: (1) hook steering fix, (2) cost verification, (3) observability stack.
