<purpose>
Parallel code review of phase changes using 3 independent Claude subagents.

All three reviewers perform the same general review of all modified files. Because each subagent runs in a fresh context with stochastic sampling, they independently surface different issues. Findings flagged by multiple reviewers carry higher confidence.

Findings are deduplicated, prioritized by consensus, and written to REVIEW.md.

**This is NOT a replacement for Stage 1 (spec compliance) or Stage 2 (automated quality).** It catches what tooling cannot: logic errors, subtle bugs, architectural drift, security gaps.
</purpose>

<process>

<step name="initialize" priority="first">
Load phase context:

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init phase-op "${PHASE_ARG}")
```

Extract: `phase_dir`, `phase_number`, `phase_name`.

Load project context for reviewers:
```bash
cat .planning/PROJECT.md 2>/dev/null | head -100
```

Extract phase goal:
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "${PHASE_NUMBER}"
```
</step>

<step name="extract_modified_files">
Extract files modified in this phase from SUMMARY.md:

```bash
MODIFIED_FILES=$(grep -h "files_modified\|key-files\|created\|modified" "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null | grep -oE "[a-zA-Z0-9_./-]+\.[a-z]+" | sort -u)
```

**If no SUMMARY.md exists:** Check git commits tagged for this phase:
```bash
MODIFIED_FILES=$(git log --oneline --name-only | grep -E "^\S+\.\w+$" | sort -u | head -50)
```

**If no modified files found:** Report and exit — nothing to review.

Filter out non-reviewable files:
```bash
echo "$MODIFIED_FILES" | grep -vE "\.(md|json|lock|yml|yaml|env|gitignore|svg|png|jpg)$"
```

Store the filtered list as `REVIEW_FILES`.

**File count guard:** If `REVIEW_FILES` exceeds 30 files, prioritize:
1. Files in `src/` over config/test files
2. Files with more lines changed (from git diff --stat)
3. Cap at 30 files — note skipped files in the report
</step>

<step name="spawn_parallel_reviewers">
Spawn 3 identical reviewer subagents in parallel. Each performs a general review of ALL modified files across all review areas defined in gsd-reviewer.

The value of 3 independent runs: each Claude instance samples differently, so they independently surface different issues. When two or three independently flag the same location, confidence is high.

```
Task(
  prompt="""
<review_context>

**Reviewer:** 1 of 3
**Phase:** {phase_number} — {phase_name}
**Phase Goal:** {phase_goal}

**Project Context:**
{first 100 lines of PROJECT.md — stack, conventions}

**Modified Files to Review:**
{REVIEW_FILES list}

**Phase Directory:** {phase_dir}

</review_context>

<instructions>
You are reviewer 1. Read and follow the gsd-reviewer agent instructions.
Review EVERY file in the modified files list.
Return structured findings in the exact format specified by gsd-reviewer.
</instructions>

@~/.claude/get-shit-done/agents/gsd-reviewer.md
""",
  subagent_type="gsd-reviewer",
  description="Review Phase {phase}: reviewer 1/3"
)
```

```
Task(
  prompt="""
<review_context>

**Reviewer:** 2 of 3
**Phase:** {phase_number} — {phase_name}
**Phase Goal:** {phase_goal}

**Project Context:**
{first 100 lines of PROJECT.md — stack, conventions}

**Modified Files to Review:**
{REVIEW_FILES list}

**Phase Directory:** {phase_dir}

</review_context>

<instructions>
You are reviewer 2. Read and follow the gsd-reviewer agent instructions.
Review EVERY file in the modified files list.
Return structured findings in the exact format specified by gsd-reviewer.
</instructions>

@~/.claude/get-shit-done/agents/gsd-reviewer.md
""",
  subagent_type="gsd-reviewer",
  description="Review Phase {phase}: reviewer 2/3"
)
```

```
Task(
  prompt="""
<review_context>

**Reviewer:** 3 of 3
**Phase:** {phase_number} — {phase_name}
**Phase Goal:** {phase_goal}

**Project Context:**
{first 100 lines of PROJECT.md — stack, conventions}

**Modified Files to Review:**
{REVIEW_FILES list}

**Phase Directory:** {phase_dir}

</review_context>

<instructions>
You are reviewer 3. Read and follow the gsd-reviewer agent instructions.
Review EVERY file in the modified files list.
Return structured findings in the exact format specified by gsd-reviewer.
</instructions>

@~/.claude/get-shit-done/agents/gsd-reviewer.md
""",
  subagent_type="gsd-reviewer",
  description="Review Phase {phase}: reviewer 3/3"
)
```

**Wait for ALL three to complete** before proceeding.
</step>

<step name="compile_findings">
Parse structured findings from all three reviewers.

**Deduplication (±3 line tolerance):**

For each finding, check if another reviewer flagged the same file within ±3 lines with a similar issue:
- If match found: merge into single finding, note which reviewers flagged it (e.g., "2/3 reviewers", "3/3 reviewers")
- If no match: keep as single-reviewer finding

**Priority scoring:**

| Condition | Priority |
|-----------|----------|
| Critical + flagged by 2-3 reviewers | P1 — consensus critical |
| Critical + flagged by 1 reviewer | P2 — single critical |
| Warning + flagged by 2-3 reviewers | P3 — consensus warning |
| Warning + flagged by 1 reviewer | P4 — single warning |
| Suggestion + flagged by 2-3 reviewers | P5 — consensus suggestion |
| Suggestion + flagged by 1 reviewer | P6 — single suggestion |

Sort findings by priority (P1 first).

**Consensus-based severity promotion:**

Findings flagged by multiple independent reviewers are more likely real issues:
- Warning flagged by 3/3 reviewers → promote to critical
- Suggestion flagged by 3/3 reviewers → promote to warning
- Warning flagged by 2/3 reviewers → keep as warning but mark high-confidence
</step>

<step name="determine_review_status">
Based on compiled findings:

**review_passed:** Zero critical findings (warnings and suggestions are informational).

**review_blocked:** One or more critical findings exist. These block UAT until addressed.

**review_clean:** Zero findings across all reviewers. Code quality is strong.
</step>

<step name="write_review_report">
Create the review report:

```bash
REPORT_PATH="$PHASE_DIR/${PHASE_NUM}-REVIEW.md"
```

```markdown
---
phase: {XX-name}
reviewed: {ISO timestamp}
status: {review_passed | review_blocked | review_clean}
reviewers: 3
files_reviewed: {N}
total_findings: {N}
critical: {N}
warnings: {N}
suggestions: {N}
consensus_findings: {N}
---

# Phase {X}: {Name} — Code Review Report

**Reviewed:** {timestamp}
**Status:** {status}
**Reviewers:** 3 independent Claude instances
**Files Reviewed:** {N}

## Summary

| Severity | Count | Consensus (2+ reviewers) |
|----------|-------|--------------------------|
| Critical | {N} | {N} |
| Warning | {N} | {N} |
| Suggestion | {N} | {N} |

{If review_blocked:}
**{N} critical finding(s) must be addressed before UAT.**

{If review_clean:}
**No issues found across 3 independent reviews.**

## Findings

{Sorted by priority, each finding:}

### {#}. [{severity icon}] {issue one-liner}

**File:** `{path}` **Line:** {N}
**Severity:** {critical | warning | suggestion}
**Consensus:** {1/3 | 2/3 | 3/3} reviewers

{detail}

**Suggested fix:**
{suggested_fix}

---

## Per-Reviewer Summary

### Reviewer 1
{N} findings: {N} critical, {N} warnings, {N} suggestions

### Reviewer 2
{N} findings: {N} critical, {N} warnings, {N} suggestions

### Reviewer 3
{N} findings: {N} critical, {N} warnings, {N} suggestions

---

_Reviewed by 3 parallel Claude subagents (independent general review)_
```

Commit the report:
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(phase-${PHASE_NUM}): parallel code review (3 reviewers)" --files "$REPORT_PATH"
```
</step>

<step name="present_results">
Display results to user:

**If review_clean:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CODE REVIEW COMPLETE ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase {X}: {Name} — No issues found across 3 independent reviews.

Report: {phase_dir}/{phase_num}-REVIEW.md

Ready for UAT: `/gsd:verify-work {X}`
```

**If review_passed (warnings/suggestions only):**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CODE REVIEW COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase {X}: {Name} — {N} warnings, {N} suggestions (no blockers)

| # | File | Severity | Issue | Consensus |
|---|------|----------|-------|-----------|
{top 10 findings}

Full report: {phase_dir}/{phase_num}-REVIEW.md

Ready for UAT: `/gsd:verify-work {X}`
```

**If review_blocked (critical findings):**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CODE REVIEW — CRITICAL ISSUES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase {X}: {Name} — {N} critical issue(s) block UAT

{For each critical finding:}
### {#}. {issue}
File: `{path}` Line: {N}
Consensus: {N}/3 reviewers
{detail}
Fix: {suggested_fix}

Full report: {phase_dir}/{phase_num}-REVIEW.md

───────────────────────────────────────────────────────

**Options:**
1. Fix critical issues → re-run `/gsd:review-phase {X}`
2. Proceed to UAT anyway → `/gsd:verify-work {X}`
3. Plan fixes → `/gsd:plan-phase {X} --gaps`
```
</step>

</process>

<success_criteria>
- [ ] Modified files extracted from SUMMARY.md
- [ ] 3 reviewer subagents spawned in parallel (identical general review)
- [ ] All 3 reviewers completed before compilation
- [ ] Findings deduplicated across reviewers (±3 line tolerance)
- [ ] Findings prioritized by severity × consensus
- [ ] Severity promoted when consensus is strong (3/3 → upgrade)
- [ ] REVIEW.md created with all findings and per-reviewer breakdown
- [ ] Report committed
- [ ] Results presented with clear next steps
</success_criteria>
