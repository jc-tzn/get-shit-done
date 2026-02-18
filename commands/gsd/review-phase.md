---
name: gsd:review-phase
description: Multi-perspective code review by parallel Claude subagents
argument-hint: "<phase-number>"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Write
  - Task
---
<objective>
Run parallel code reviews from three distinct perspectives on phase-modified files. Compiles deduplicated, prioritized findings into {phase_num}-REVIEW.md.

Purpose: Catch logic errors, security issues, architecture drift, and performance problems that automated tooling (lint/types/tests) misses. Fills the gap between Stage 2 (automated quality) and UAT (manual testing).

Output: {phase_num}-REVIEW.md in the phase directory. Critical findings block UAT.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/review-phase.md
</execution_context>

<context>
Phase: $ARGUMENTS
- Required: phase number (e.g., "4")

@.planning/STATE.md
@.planning/ROADMAP.md
</context>

<process>
Execute the review-phase workflow from @~/.claude/get-shit-done/workflows/review-phase.md end-to-end.
Preserve all workflow gates (file extraction, parallel spawning, compilation, report generation).
</process>
