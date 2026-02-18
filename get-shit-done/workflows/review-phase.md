<purpose>
Parallel code review of phase changes using 3 independent Claude subagents.

All three reviewers perform the same general review of all modified files. Because each subagent runs in a fresh context with stochastic sampling, they independently surface different issues. Findings flagged by multiple reviewers carry higher confidence.

Findings are deduplicated, conflicts resolved, and prioritized by consensus into REVIEW.md. Optionally auto-applies non-controversial fixes (`--fix` flag). Tracks review pass history to prevent churn on follow-up runs. Assesses convergence to signal when review loops should stop.

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

**Detect follow-up pass:**

```bash
PRIOR_REVIEWS=$(ls "$PHASE_DIR"/*-REVIEW.md 2>/dev/null | wc -l | tr -d ' ')
```

If `PRIOR_REVIEWS` > 0:
- Set `REVIEW_PASS` to `PRIOR_REVIEWS + 1`
- Extract the prior review's timestamp and findings summary:
  ```bash
  PRIOR_REVIEW=$(ls -t "$PHASE_DIR"/*-REVIEW.md 2>/dev/null | head -1)
  PRIOR_CRITICAL=$(grep "^critical:" "$PRIOR_REVIEW" 2>/dev/null | cut -d: -f2 | tr -d ' ')
  PRIOR_WARNINGS=$(grep "^warnings:" "$PRIOR_REVIEW" 2>/dev/null | cut -d: -f2 | tr -d ' ')
  PRIOR_DATE=$(grep "^reviewed:" "$PRIOR_REVIEW" 2>/dev/null | cut -d: -f2- | tr -d ' ')
  ```
- Rename the prior review to preserve history: `mv "$PRIOR_REVIEW" "${PRIOR_REVIEW%.md}.pass${PRIOR_REVIEWS}.md"`
- Set `IS_FOLLOWUP=true`

If `PRIOR_REVIEWS` == 0:
- Set `REVIEW_PASS=1`, `IS_FOLLOWUP=false`

**Collect recent fix commits (for follow-up context):**

If `IS_FOLLOWUP=true`:
```bash
RECENT_FIX_COMMITS=$(git log --oneline --since="$PRIOR_DATE" --all -- $(echo "$REVIEW_FILES" | tr '\n' ' ') 2>/dev/null | head -20)
```

This gives reviewers context about what changed since the last review, so they don't re-litigate fixed issues.
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

**Build the follow-up context block** (empty string if first pass):

If `IS_FOLLOWUP=true`:
```
FOLLOWUP_CONTEXT="""
<followup_pass>
**This is review pass #{REVIEW_PASS}.** A prior review ran on {PRIOR_DATE} and found {PRIOR_CRITICAL} critical / {PRIOR_WARNINGS} warning issues. Fixes have been applied since then.

**Recent fix commits since last review:**
{RECENT_FIX_COMMITS}

**Follow-up rules:**
- Check git log to understand WHY recent changes were made before recommending reversals
- If a pattern looks intentional based on recent commit messages, don't recommend reversing it without strong justification
- Focus on issues that may have been INTRODUCED by recent changes, not re-reviewing unchanged logic
- If you spot a finding that was likely already addressed by a fix commit, skip it
- Aim for convergence: the goal is to ship, not to endlessly refactor
</followup_pass>
"""
```

If `IS_FOLLOWUP=false`:
```
FOLLOWUP_CONTEXT=""
```

**Spawn 3 reviewers** (each gets identical context including the follow-up block):

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

{FOLLOWUP_CONTEXT}

</review_context>

<instructions>
You are reviewer 1. Read and follow the gsd-reviewer agent instructions.
Review EVERY file in the modified files list.
Return structured findings in the exact format specified by gsd-reviewer.
</instructions>

@~/.claude/get-shit-done/agents/gsd-reviewer.md
""",
  subagent_type="gsd-reviewer",
  description="Review Phase {phase}: reviewer 1/3 (pass #{REVIEW_PASS})"
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

{FOLLOWUP_CONTEXT}

</review_context>

<instructions>
You are reviewer 2. Read and follow the gsd-reviewer agent instructions.
Review EVERY file in the modified files list.
Return structured findings in the exact format specified by gsd-reviewer.
</instructions>

@~/.claude/get-shit-done/agents/gsd-reviewer.md
""",
  subagent_type="gsd-reviewer",
  description="Review Phase {phase}: reviewer 2/3 (pass #{REVIEW_PASS})"
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

{FOLLOWUP_CONTEXT}

</review_context>

<instructions>
You are reviewer 3. Read and follow the gsd-reviewer agent instructions.
Review EVERY file in the modified files list.
Return structured findings in the exact format specified by gsd-reviewer.
</instructions>

@~/.claude/get-shit-done/agents/gsd-reviewer.md
""",
  subagent_type="gsd-reviewer",
  description="Review Phase {phase}: reviewer 3/3 (pass #{REVIEW_PASS})"
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

**Conflict resolution:**

When two reviewers suggest contradictory fixes for the same location:
- **Prefer existing code / DRY** — if one reviewer says "use existing method X" and another says "extract new method Y," prefer the existing code
- **Prefer simpler** — if one reviewer suggests adding abstraction and another says current code is fine, prefer simpler unless the complexity is flagged as critical
- **Prefer consensus** — if 2 reviewers agree and 1 disagrees, go with the 2
- **Log the conflict** in `CONFLICTS` list with both suggestions and the resolution reason

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

<step name="auto_apply_fixes">
**This step only runs when the `--fix` flag is passed to the command.** If no `--fix` flag, skip to `determine_review_status`.

Categorize compiled findings into auto-fixable vs. requires-human:

**Auto-fixable** (apply without asking):
- Suggestions flagged by 2+ reviewers (high-confidence, low-risk)
- Warnings with a concrete `suggested_fix` that is a localized change (single file, < 10 lines)
- Dead code removal (unused imports, unreachable branches) flagged by 2+ reviewers

**Requires human** (report only, do not apply):
- All critical findings (too risky for auto-fix)
- Warnings that involve architectural changes (new files, moving code between modules)
- Any finding where reviewers conflicted on the fix approach
- Findings that touch test files or configuration

**For each auto-fixable finding:**
1. Read the target file
2. Apply the suggested fix
3. Run a quick syntax/type check on the modified file:
   ```bash
   npx tsc --noEmit --pretty "$FILE" 2>&1 | tail -5
   ```
4. If check passes → record as `APPLIED`
5. If check fails → revert the change, record as `REVERTED` with reason

**Skipped findings log:**

For every finding NOT auto-applied, record in `SKIPPED_FINDINGS` list:

```
| # | File | Issue | Reason Skipped |
|---|------|-------|----------------|
| 3 | src/api/chat.ts | Missing error boundary | Critical — requires human decision |
| 7 | src/utils/parse.ts | Extract to shared util | Architectural change — out of auto-fix scope |
| 9 | src/components/List.tsx | Reviewers disagreed | Conflict: R1 says memoize, R2 says restructure |
```

**Post-apply verification:**

After all auto-fixes are applied:
```bash
npm run lint -- --quiet 2>/dev/null; echo "LINT_EXIT=$?"
npx tsc --noEmit 2>/dev/null; echo "TSC_EXIT=$?"
npm test -- --passWithNoTests 2>/dev/null; echo "TEST_EXIT=$?"
```

If any check fails, revert ALL auto-fixes and report the failure. Auto-fix must not introduce regressions.

If all checks pass, commit the auto-fixes:
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "fix(phase-${PHASE_NUM}): auto-apply ${APPLIED_COUNT} review findings" --files "${APPLIED_FILES}"
```

Record: `APPLIED_COUNT`, `SKIPPED_COUNT`, `REVERTED_COUNT`.
</step>

<step name="determine_review_status">
Based on compiled findings (after auto-apply, if run):

**review_passed:** Zero critical findings remaining (warnings and suggestions are informational).

**review_blocked:** One or more critical findings remain. These block UAT until addressed.

**review_clean:** Zero findings across all reviewers. Code quality is strong.
</step>

<step name="assess_convergence">
Determine whether another review pass is warranted.

**Signals favoring "ready to ship":**
- `review_clean` or `review_passed` with only suggestions remaining
- This is pass #2+ and total findings decreased compared to prior pass
- Auto-fix resolved all non-critical findings (if `--fix` was used)
- No critical findings remain

**Signals favoring "run another pass":**
- Auto-fix applied substantial changes (> 5 files or > 50 lines changed)
- Critical findings were fixed manually between passes but new code was introduced
- This is pass #1 and `review_blocked` — fixes haven't been applied yet

**Set `CONVERGENCE`:**
- `converged` — minor or no findings, ready to proceed
- `another_pass` — substantial changes applied, recommend re-review
- `blocked` — critical findings still open, must fix first

On follow-up passes: if total findings from this pass are <= 20% of the prior pass's findings, set `converged` regardless of other signals.
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
pass: {REVIEW_PASS}
status: {review_passed | review_blocked | review_clean}
convergence: {converged | another_pass | blocked}
reviewers: 3
files_reviewed: {N}
total_findings: {N}
critical: {N}
warnings: {N}
suggestions: {N}
consensus_findings: {N}
auto_applied: {N or 0 if --fix not used}
skipped: {N}
conflicts_resolved: {N}
---

# Phase {X}: {Name} — Code Review Report

**Reviewed:** {timestamp}
**Pass:** #{REVIEW_PASS}
**Status:** {status}
**Convergence:** {converged | another_pass | blocked}
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

{If IS_FOLLOWUP:}
**Pass #{REVIEW_PASS}** — Prior review on {PRIOR_DATE} found {PRIOR_CRITICAL} critical / {PRIOR_WARNINGS} warnings. This pass found {current totals}. {Trend: improved / regressed / stable}.

## Findings

{Sorted by priority, each finding:}

### {#}. [{severity icon}] {issue one-liner}

**File:** `{path}` **Line:** {N}
**Severity:** {critical | warning | suggestion}
**Consensus:** {1/3 | 2/3 | 3/3} reviewers
{If auto-applied:} **Status:** Auto-fixed

{detail}

**Suggested fix:**
{suggested_fix}

---

{If --fix was used:}
## Auto-Applied Fixes

{APPLIED_COUNT} findings auto-fixed, {SKIPPED_COUNT} skipped, {REVERTED_COUNT} reverted.

| # | File | Issue | Status |
|---|------|-------|--------|
{For each auto-fixable finding: applied / reverted with reason}

Post-fix checks: lint {pass/fail}, types {pass/fail}, tests {pass/fail}

## Skipped Findings

| # | File | Issue | Reason Skipped |
|---|------|-------|----------------|
{For each skipped finding with justification}

{If CONFLICTS is non-empty:}
## Conflict Resolutions

| File | Reviewer A | Reviewer B | Resolution | Rationale |
|------|------------|------------|------------|-----------|
{For each conflict with both sides and the chosen resolution}

## Per-Reviewer Summary

### Reviewer 1
{N} findings: {N} critical, {N} warnings, {N} suggestions

### Reviewer 2
{N} findings: {N} critical, {N} warnings, {N} suggestions

### Reviewer 3
{N} findings: {N} critical, {N} warnings, {N} suggestions

## Another Pass Needed?

{If converged:}
**No.** Findings are minor or resolved. Ready to proceed to UAT.

{If another_pass:}
**Yes.** Substantial changes were applied during this review. Recommend running `/gsd:review-phase {X}` once more to verify the fixes didn't introduce new issues. Expect the next pass to converge.

{If blocked:}
**Yes — critical findings must be fixed first.** Fix the {N} critical issues above, then re-run `/gsd:review-phase {X}`.

---

_Review pass #{REVIEW_PASS} by 3 parallel Claude subagents_
```

Commit the report:
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs commit "docs(phase-${PHASE_NUM}): code review pass #${REVIEW_PASS} (3 reviewers)" --files "$REPORT_PATH"
```
</step>

<step name="present_results">
Display results to user. Include pass number and convergence in all outputs.

**If review_clean:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CODE REVIEW COMPLETE ✓  (pass #{REVIEW_PASS})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase {X}: {Name} — No issues found across 3 independent reviews.

Report: {phase_dir}/{phase_num}-REVIEW.md

Ready for UAT: `/gsd:verify-work {X}`
```

**If review_passed (warnings/suggestions only):**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CODE REVIEW COMPLETE  (pass #{REVIEW_PASS})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase {X}: {Name} — {N} warnings, {N} suggestions (no blockers)
{If --fix used:} Auto-fixed: {APPLIED_COUNT} | Skipped: {SKIPPED_COUNT}

| # | File | Severity | Issue | Consensus |
|---|------|----------|-------|-----------|
{top 10 findings}

Full report: {phase_dir}/{phase_num}-REVIEW.md

{If converged:} Ready for UAT: `/gsd:verify-work {X}`
{If another_pass:} Recommend one more pass: `/gsd:review-phase {X}` (substantial auto-fixes applied)
```

**If review_blocked (critical findings):**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► CODE REVIEW — CRITICAL ISSUES  (pass #{REVIEW_PASS})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase {X}: {Name} — {N} critical issue(s) block UAT
{If --fix used:} Auto-fixed: {APPLIED_COUNT} non-critical | Skipped: {SKIPPED_COUNT} (including {N} critical)

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
2. Fix + auto-apply non-critical → `/gsd:review-phase {X} --fix`
3. Proceed to UAT anyway → `/gsd:verify-work {X}`
4. Plan fixes → `/gsd:plan-phase {X} --gaps`
```
</step>

</process>

<success_criteria>
- [ ] Follow-up pass detected (prior REVIEW.md check, pass number set)
- [ ] Modified files extracted from SUMMARY.md
- [ ] 3 reviewer subagents spawned in parallel with follow-up context (if applicable)
- [ ] All 3 reviewers completed before compilation
- [ ] Findings deduplicated across reviewers (±3 line tolerance)
- [ ] Conflicts between reviewers resolved with logged rationale
- [ ] Findings prioritized by severity × consensus
- [ ] Severity promoted when consensus is strong (3/3 → upgrade)
- [ ] Auto-fixes applied if --fix flag set (with post-fix verification)
- [ ] Skipped findings logged with justification
- [ ] Convergence assessed (converged / another_pass / blocked)
- [ ] REVIEW.md created with pass number, convergence, conflicts, skipped log
- [ ] Report committed
- [ ] Results presented with convergence-aware next steps
</success_criteria>
