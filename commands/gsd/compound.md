---
name: gsd:compound
description: Capture learnings from completed phase to make future phases faster
argument-hint: "[phase number, e.g., '4']"
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Write
---
<objective>
Capture learnings, patterns, and decisions from a completed phase so future planning and execution benefit from accumulated knowledge.

Purpose: Each phase should make the next one easier. Without explicit capture, insights die with the context window.

Output: {phase_num}-LEARNINGS.md in the phase directory. STATE.md updated with extracted patterns.
</objective>

<execution_context>
@~/.claude/get-shit-done/workflows/compound.md
</execution_context>

<context>
Phase: $ARGUMENTS (optional)
- If provided: Compound specific phase (e.g., "4")
- If not provided: Compound most recently completed phase

@.planning/STATE.md
@.planning/ROADMAP.md
</context>

<process>
Execute the compound workflow from @~/.claude/get-shit-done/workflows/compound.md end-to-end.
</process>
