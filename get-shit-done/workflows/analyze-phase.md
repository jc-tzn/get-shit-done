<purpose>
Cross-artifact consistency and coverage analysis for a phase. Validates that REQUIREMENTS, ROADMAP, CONTEXT, PLAN files, and PROJECT boundaries all agree before execution. Orchestrates the gsd-analyzer agent.
</purpose>

<required_reading>
Read all files referenced by the invoking prompt's execution_context before starting.
</required_reading>

<process>

## 1. Initialize

Load phase context:

```bash
INIT_RAW=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init phase-op "$PHASE" --include state,roadmap,requirements)
if [[ "$INIT_RAW" == @file:* ]]; then
  INIT_FILE="${INIT_RAW#@file:}"
  INIT=$(cat "$INIT_FILE")
  rm -f "$INIT_FILE"
else
  INIT="$INIT_RAW"
fi
```

Parse JSON for: `phase_dir`, `phase_number`, `phase_name`, `padded_phase`, `has_plans`, `plan_count`, `planning_exists`, `roadmap_exists`.

File contents from `--include`: `state_content`, `roadmap_content`, `requirements_content`.

**If `planning_exists` is false:** Error — run `/gsd:new-project` first.
**If `has_plans` is false:** Error — run `/gsd:plan-phase` first. Analysis requires plans to exist.
**If `roadmap_exists` is false:** Error — ROADMAP.md is required for cross-artifact analysis.

## 2. Validate Prerequisites

Check that core artifacts exist:

```bash
PHASE_DIR=$(echo "$INIT" | node -e "const d=require('fs').readFileSync(0,'utf8');console.log(JSON.parse(d).phase_dir)")

ls .planning/PROJECT.md 2>/dev/null
ls .planning/REQUIREMENTS.md 2>/dev/null
ls .planning/ROADMAP.md 2>/dev/null
ls "$PHASE_DIR"/*-PLAN.md 2>/dev/null
ls "$PHASE_DIR"/*-CONTEXT.md 2>/dev/null
```

| Artifact | Required | Fallback |
|----------|----------|----------|
| PROJECT.md | Yes | Error — run `/gsd:new-project` |
| REQUIREMENTS.md | Yes | Error — run `/gsd:new-project` |
| ROADMAP.md | Yes | Error — run `/gsd:new-project` |
| PLAN.md files | Yes (at least 1) | Error — run `/gsd:plan-phase` first |
| CONTEXT.md | No | Skip Dimension 2 (Context-to-Plan Alignment), note in report |

Display:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► ANALYZING PHASE {X}: {Name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Artifacts:
  PROJECT.md ......... ✓
  REQUIREMENTS.md .... ✓
  ROADMAP.md ......... ✓
  CONTEXT.md ......... {✓ | ⊘ skipping alignment checks}
  PLAN.md files ...... {N} found

Spawning cross-artifact analyzer...
```

## 3. Load All Artifact Contents

Read all artifacts into memory for the analyzer prompt:

```bash
PROJECT_CONTENT=$(cat .planning/PROJECT.md)
REQUIREMENTS_CONTENT=$(cat .planning/REQUIREMENTS.md)
ROADMAP_CONTENT=$(cat .planning/ROADMAP.md)

CONTEXT_CONTENT=""
CONTEXT_FILE=$(ls "$PHASE_DIR"/*-CONTEXT.md 2>/dev/null | head -1)
if [ -n "$CONTEXT_FILE" ]; then
  CONTEXT_CONTENT=$(cat "$CONTEXT_FILE")
fi

PLAN_CONTENTS=""
for plan in "$PHASE_DIR"/*-PLAN.md; do
  PLAN_CONTENTS="$PLAN_CONTENTS\n\n--- FILE: $(basename $plan) ---\n$(cat "$plan")"
done
```

## 4. Spawn Analyzer Agent

Spawn `gsd-analyzer` as a Task subagent with all artifact contents:

```
Task: gsd-analyzer for Phase {X}

<agent_prompt>
@~/.claude/get-shit-done/agents/gsd-analyzer.md
</agent_prompt>

<artifacts>

<project_md>
${PROJECT_CONTENT}
</project_md>

<requirements_md>
${REQUIREMENTS_CONTENT}
</requirements_md>

<roadmap_md>
${ROADMAP_CONTENT}
</roadmap_md>

<context_md>
${CONTEXT_CONTENT}
<!-- If empty: "No CONTEXT.md for this phase. Skip Dimension 2 (Context-to-Plan Alignment)." -->
</context_md>

<plan_files>
${PLAN_CONTENTS}
</plan_files>

</artifacts>

<task>
Analyze Phase ${PHASE_NUMBER} ("${PHASE_NAME}") for cross-artifact consistency.

Run all applicable analysis dimensions. Return the structured report (ANALYSIS CLEAN or ISSUES FOUND format from your structured_returns section).

Phase directory: ${PHASE_DIR}
</task>
```

**Wait for analyzer to complete.**

## 5. Process Analyzer Results

Parse the analyzer's return for status: `clean`, `warnings_found`, or `blockers_found`.

**Count issues by severity:**
```
BLOCKERS = count of severity: blocker
WARNINGS = count of severity: warning
```

## 6. Write Analysis Report

Write the report to the phase directory:

```bash
ANALYSIS_FILE="${PHASE_DIR}/${PADDED_PHASE}-ANALYSIS.md"
```

The report content is the analyzer's structured return, with a frontmatter header:

```markdown
---
phase: {phase_number}
phase_name: "{phase_name}"
analyzed: "{ISO date}"
status: clean | warnings_found | blockers_found
blockers: {N}
warnings: {N}
artifacts_checked:
  - PROJECT.md
  - REQUIREMENTS.md
  - ROADMAP.md
  - CONTEXT.md  # or "absent"
  - "{N} PLAN.md files"
dimensions_run:
  - requirement_plan_coverage
  - context_plan_alignment  # or "skipped (no CONTEXT.md)"
  - roadmap_requirements_traceability
  - boundary_compliance
  - cross_plan_consistency
  - success_criteria_derivation
---

{analyzer structured report content}
```

If `commit_docs` is enabled in config:

```bash
git add "$ANALYSIS_FILE"
git commit -m "analysis(${PADDED_PHASE}): cross-artifact consistency check — ${STATUS}"
```

## 7. Present Results

### If clean:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {X} ANALYSIS: CLEAN ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

All 6 consistency dimensions passed.
{N} requirements covered, {M} plans checked, 0 issues.

Report: {ANALYSIS_FILE}

▶ Next: /gsd:execute-phase {X}
```

### If warnings only:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {X} ANALYSIS: {N} WARNING(S)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

{Display each warning with 1-line summary}

Report: {ANALYSIS_FILE}

▶ Proceed: /gsd:execute-phase {X}  (warnings noted, not blocking)
▶ Fix first: /gsd:plan-phase {X}   (revise plans to address warnings)
```

### If blockers found:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {X} ANALYSIS: {N} BLOCKER(S)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

{Display each blocker with description and fix hint}

{If also warnings: "{M} additional warning(s) — see report."}

Report: {ANALYSIS_FILE}

⚠ Fix required before execution:
  /gsd:plan-phase {X}  — revise plans to resolve blockers
  Or manually update REQUIREMENTS.md / ROADMAP.md if the issue is there
```

</process>

<offer_next>
Output this markdown directly (not as a code block):

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PHASE {X} ANALYZED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Phase {X}: {Name}** — {status}

| Dimension | Status |
|-----------|--------|
| Requirement Coverage | {Passed | {N} issues} |
| Context Alignment | {Passed | Skipped | {N} issues} |
| Roadmap Traceability | {Passed | {N} issues} |
| Boundary Compliance | {Passed | {N} issues} |
| Cross-Plan Consistency | {Passed | {N} issues} |
| Success Criteria | {Passed | {N} issues} |

───────────────────────────────────────────────────────────────

## ▶ Next Up

{If clean/warnings: "/gsd:execute-phase {X}"}
{If blockers: "/gsd:plan-phase {X} — fix blockers first"}

───────────────────────────────────────────────────────────────

**Also available:**
- cat {ANALYSIS_FILE} — full report
- /gsd:review-phase {X} — code review (after execution)

───────────────────────────────────────────────────────────────
</offer_next>

<success_criteria>
- [ ] .planning/ directory validated
- [ ] Phase validated against roadmap
- [ ] All required artifacts confirmed (PROJECT, REQUIREMENTS, ROADMAP, PLANs)
- [ ] CONTEXT.md presence detected (optional — skip Dimension 2 if absent)
- [ ] gsd-analyzer spawned with all artifact contents
- [ ] Analyzer completed and returned structured report
- [ ] Analysis report written to {phase_num}-ANALYSIS.md
- [ ] Report committed (if commit_docs enabled)
- [ ] Results displayed with clear status and next steps
- [ ] User knows whether to proceed or fix issues
</success_criteria>
