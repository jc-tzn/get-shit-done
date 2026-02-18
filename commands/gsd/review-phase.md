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
Run parallel code reviews using 3 independent Claude subagents on phase-modified files. Compiles deduplicated, prioritized findings into {phase_num}-REVIEW.md.

Catches logic errors, security issues, architecture drift, and performance problems that automated tooling misses. Fills the gap between Stage 2 (automated quality) and UAT (manual testing).

Supports multi-pass reviews: detects prior REVIEW.md files, marks pass number, and adjusts reviewer focus to avoid re-litigating fixed issues. Assesses convergence to prevent infinite review loops.

With `--fix`: auto-applies non-controversial fixes (consensus suggestions, localized warnings) and commits them. Critical findings always require human decision.

Output: {phase_num}-REVIEW.md in the phase directory.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/review-phase.md
</execution_context>

<context>
Phase: $ARGUMENTS
- Required: phase number (e.g., "4")
- Optional: `--fix` flag to auto-apply non-controversial findings

@.planning/STATE.md
@.planning/ROADMAP.md
</context>

<process>
Execute the review-phase workflow from @~/.claude/get-shit-done/workflows/review-phase.md end-to-end.
Preserve all workflow gates (file extraction, follow-up detection, parallel spawning, compilation, conflict resolution, auto-apply, convergence assessment, report generation).
</process>
