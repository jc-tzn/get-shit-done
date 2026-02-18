<purpose>
Capture learnings from a completed phase so future phases benefit from accumulated knowledge.

Each execution cycle produces insights that would otherwise die with the context window:
- Which patterns worked and should be repeated
- Which approaches failed and should be avoided
- Which tools/libraries had gotchas worth documenting
- Which estimates were wrong and why
- Which deviations from the plan were improvements

This workflow extracts those insights and persists them where future planning agents will find them.
</purpose>

<philosophy>
**Compounding, not documenting.**

The goal is NOT documentation for humans. It's structured knowledge that future GSD agents consume:

- `gsd-phase-researcher` reads LEARNINGS.md to avoid repeating mistakes
- `gsd-planner` reads LEARNINGS.md to reuse proven patterns
- `gsd-executor` benefits indirectly through better plans

Write for the agent, not the archive.
</philosophy>

<process>

<step name="initialize" priority="first">
Load phase context:

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init phase-op "${PHASE_ARG}")
```

Extract: `phase_dir`, `phase_number`, `phase_name`, `commit_docs`.

If no phase argument, find most recently completed phase:

```bash
for dir in .planning/phases/*/; do
  if ls "$dir"*-SUMMARY.md 1>/dev/null 2>&1; then
    echo "$dir"
  fi
done | tail -1
```
</step>

<step name="gather_sources">
Read all phase artifacts:

```bash
cat "$phase_dir"/*-SUMMARY.md 2>/dev/null
cat "$phase_dir"/*-VERIFICATION.md 2>/dev/null
cat "$phase_dir"/*-UAT.md 2>/dev/null
cat "$phase_dir"/*-PLAN.md 2>/dev/null
cat "$phase_dir"/*-RESEARCH.md 2>/dev/null
cat "$phase_dir"/*-CONTEXT.md 2>/dev/null
```

Also read STATE.md for decisions made during this phase:
```bash
cat .planning/STATE.md 2>/dev/null
```
</step>

<step name="extract_learnings">
Analyze the gap between PLAN and SUMMARY/VERIFICATION/UAT to extract learnings across these dimensions:

**1. Pattern Wins** — What worked well and should be repeated?
- Libraries/tools that performed as expected
- Architectural patterns that made execution smooth
- Task sizing that hit the sweet spot
- Dependency ordering that enabled parallelism

**2. Pattern Failures** — What went wrong and should be avoided?
- Deviations logged in SUMMARY.md (Rule 1-4 triggers)
- Verification failures from VERIFICATION.md
- UAT issues from UAT.md
- Scope overruns (plans that needed splitting mid-execution)

**3. Estimation Calibration** — How accurate were the plans?
- Compare planned tasks vs actual tasks (from SUMMARY deviations)
- Compare planned files_modified vs actual files touched
- Note context budget accuracy (did plans stay under 50%?)

**4. Decision Outcomes** — Which decisions proved correct/incorrect?
- Decisions from STATE.md that affected this phase
- CONTEXT.md locked decisions — did they hold up?
- Planner revision cycles — what did the checker catch?

**5. Reusable Artifacts** — What was created that future phases can leverage?
- Shared utilities, types, or patterns established
- Configuration or infrastructure that subsequent phases build on
- Testing patterns that can be replicated

**6. Gotchas** — Non-obvious issues that would bite a fresh agent?
- Environment-specific quirks
- Library version incompatibilities
- Edge cases discovered during execution
</step>

<step name="write_learnings">
Create LEARNINGS.md:

```markdown
---
phase: {phase_number}-{phase_name}
compounded: {ISO timestamp}
source_plans: [{list of plan files}]
source_summaries: [{list of summary files}]
---

# Phase {N}: {Name} — Learnings

## Pattern Wins

### {Pattern Name}
**What:** {description}
**Evidence:** {which plan/task demonstrated this}
**Reuse:** {how future phases should apply this}

## Pattern Failures

### {Failure Name}
**What went wrong:** {description}
**Root cause:** {why it happened}
**Avoidance:** {what to do differently}

## Estimation Calibration

| Metric | Planned | Actual | Delta | Lesson |
|--------|---------|--------|-------|--------|
| Plans | {N} | {N} | {+/-N} | {insight} |
| Tasks | {N} | {N} | {+/-N} | {insight} |
| Deviations | 0 | {N} | {+N} | {insight} |

## Decision Outcomes

| Decision | Source | Outcome | Verdict |
|----------|--------|---------|---------|
| {decision} | {CONTEXT/STATE/planner} | {what happened} | {correct/incorrect/neutral} |

## Reusable Artifacts

| Artifact | Path | Future Use |
|----------|------|------------|
| {name} | {path} | {how other phases can use it} |

## Gotchas

### {Gotcha Name}
**Trigger:** {when this bites you}
**Symptom:** {what you see}
**Fix:** {what to do}
```

Write to: `$PHASE_DIR/{phase_num}-LEARNINGS.md`
</step>

<step name="update_state">
Append a Learnings section to STATE.md if it doesn't exist, or update it:

Extract the 2-3 most impactful learnings (patterns that change how future phases should be planned) and add them to STATE.md under a `## Learnings` section.

Format:
```markdown
## Learnings

- **[Phase {N}]** {one-line learning that affects future planning}
- **[Phase {N}]** {one-line learning that affects future planning}
```

These persist across sessions and are read by gsd-planner and gsd-phase-researcher.
</step>

<step name="commit">
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(${PHASE}): compound learnings from phase execution" --files "$PHASE_DIR/${PHASE_NUM}-LEARNINGS.md" .planning/STATE.md
```
</step>

<step name="present_summary">
```
## Compounded: Phase {N}

| Dimension | Count |
|-----------|-------|
| Pattern Wins | {N} |
| Pattern Failures | {N} |
| Decision Outcomes | {N} |
| Reusable Artifacts | {N} |
| Gotchas | {N} |

**Key insight:** {single most important learning}

STATE.md updated with {N} learnings for future phases.

### Next Steps

- `/gsd:plan-phase {next}` — Plan next phase (planner will read these learnings)
- `/gsd:discuss-phase {next}` — Discuss next phase first
```
</step>

</process>

<downstream_consumers>

| Consumer | What They Read | How They Use It |
|----------|---------------|-----------------|
| `gsd-phase-researcher` | Gotchas, Pattern Failures | Avoid known pitfalls in research recommendations |
| `gsd-planner` | Pattern Wins, Estimation Calibration, Reusable Artifacts | Better task sizing, reuse existing patterns, accurate scope |
| `gsd-plan-checker` | Decision Outcomes | Validate plans don't repeat known-bad decisions |

</downstream_consumers>

<success_criteria>
- [ ] All phase artifacts read (SUMMARY, VERIFICATION, UAT, PLAN, RESEARCH, CONTEXT)
- [ ] Learnings extracted across all 6 dimensions
- [ ] LEARNINGS.md created with structured, agent-consumable content
- [ ] STATE.md updated with top learnings
- [ ] Committed to git
- [ ] Summary presented with next steps
</success_criteria>
