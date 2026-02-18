---
name: gsd:analyze-phase
description: Cross-artifact consistency and coverage analysis before execution
argument-hint: "<phase-number>"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Task
  - Write
---
<objective>
Sweep across all planning artifacts (REQUIREMENTS.md, ROADMAP.md, CONTEXT.md, PLAN.md files, PROJECT.md boundaries) for a phase and surface:

1. **Coverage gaps** — requirements with no plan coverage, or plan tasks with no requirement anchor
2. **Contradictions** — CONTEXT.md decisions contradicted by plan actions, or plan assumptions that conflict with PROJECT.md boundaries
3. **Orphaned artifacts** — deferred ideas that leaked into plans, or v2 requirements treated as v1
4. **Traceability breaks** — requirement IDs in ROADMAP.md that don't match REQUIREMENTS.md, or plan `requirements` frontmatter referencing non-existent IDs
5. **Boundary violations** — plans that include "Never" actions or skip "Always" actions from Agent Boundaries

Produces {phase_num}-ANALYSIS.md in the phase directory. Run after `/gsd:plan-phase`, before `/gsd:execute-phase`.

Unlike `gsd-plan-checker` (which validates plan *structure* and goal achievement), the analyzer validates *cross-artifact consistency* — ensuring all documents agree with each other.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/analyze-phase.md
</execution_context>

<context>
Phase: $ARGUMENTS
- Required: phase number (e.g., "3")

@.planning/PROJECT.md
@.planning/REQUIREMENTS.md
@.planning/ROADMAP.md
@.planning/STATE.md
</context>

<process>
Execute the analyze-phase workflow from @~/.claude/get-shit-done/workflows/analyze-phase.md end-to-end.
Preserve all workflow gates (artifact loading, analyzer spawning, report generation, routing).
</process>
