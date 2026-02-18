<purpose>
Verify phase goal achievement through multi-stage review: spec compliance, automated quality, and optional multi-perspective code review.

Executed by a verification subagent spawned from execute-phase.md.

**Three stages catch different failure modes:**
- Stage 1 (Spec Compliance): "Does the code deliver what the phase promised?"
- Stage 2 (Code Quality): "Is the delivered code well-built?" (lint, types, tests, anti-patterns)
- Stage 3 (Parallel Code Review): "Did three independent reviewers find issues tooling missed?" (optional, config-driven)

Code can perfectly match the spec but be poorly written. Code can pass all linters but contain logic errors. All applicable stages must pass.
</purpose>

<core_principle>
**Task completion ≠ Goal achievement**

A task "create chat component" can be marked complete when the component is a placeholder. The task was done — but the goal "working chat interface" was not achieved.

Goal-backward verification:
1. What must be TRUE for the goal to be achieved?
2. What must EXIST for those truths to hold?
3. What must be WIRED for those artifacts to function?

Then verify each level against the actual codebase.

**Stage gates:**
- Stage 2 does NOT run if Stage 1 has blockers. Fix spec compliance first.
- Stage 3 does NOT run if Stage 2 has blockers. Fix automated quality first.
- Stage 3 only runs when `workflow.parallel_review` is `true` in `.planning/config.json`.
</core_principle>

<required_reading>
@~/.claude/get-shit-done/references/verification-patterns.md
@~/.claude/get-shit-done/templates/verification-report.md
</required_reading>

<process>

<step name="load_context" priority="first">
Load phase operation context:

```bash
INIT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs init phase-op "${PHASE_ARG}")
```

Extract from init JSON: `phase_dir`, `phase_number`, `phase_name`, `has_plans`, `plan_count`.

Then load phase details and list plans/summaries:
```bash
node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "${phase_number}"
grep -E "^| ${phase_number}" .planning/REQUIREMENTS.md 2>/dev/null
ls "$phase_dir"/*-SUMMARY.md "$phase_dir"/*-PLAN.md 2>/dev/null
```

Extract **phase goal** from ROADMAP.md (the outcome to verify, not tasks) and **requirements** from REQUIREMENTS.md if it exists.
</step>

<!-- ═══════════════════════════════════════════════════════════════════ -->
<!-- STAGE 1: SPEC COMPLIANCE — Does the code deliver what was planned? -->
<!-- ═══════════════════════════════════════════════════════════════════ -->

<step name="establish_must_haves">
**Option A: Must-haves in PLAN frontmatter**

Use gsd-tools to extract must_haves from each PLAN:

```bash
for plan in "$PHASE_DIR"/*-PLAN.md; do
  MUST_HAVES=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs frontmatter get "$plan" --field must_haves)
  echo "=== $plan ===" && echo "$MUST_HAVES"
done
```

Returns JSON: `{ truths: [...], artifacts: [...], key_links: [...] }`

Aggregate all must_haves across plans for phase-level verification.

**Option B: Use Success Criteria from ROADMAP.md**

If no must_haves in frontmatter (MUST_HAVES returns error or empty), check for Success Criteria:

```bash
PHASE_DATA=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs roadmap get-phase "${phase_number}" --raw)
```

Parse the `success_criteria` array from the JSON output. If non-empty:
1. Use each Success Criterion directly as a **truth** (they are already written as observable, testable behaviors)
2. Derive **artifacts** (concrete file paths for each truth)
3. Derive **key links** (critical wiring where stubs hide)
4. Document the must-haves before proceeding

Success Criteria from ROADMAP.md are the contract — they override PLAN-level must_haves when both exist.

**Option C: Derive from phase goal (fallback)**

If no must_haves in frontmatter AND no Success Criteria in ROADMAP:
1. State the goal from ROADMAP.md
2. Derive **truths** (3-7 observable behaviors, each testable)
3. Derive **artifacts** (concrete file paths for each truth)
4. Derive **key links** (critical wiring where stubs hide)
5. Document derived must-haves before proceeding
</step>

<step name="verify_truths">
For each observable truth, determine if the codebase enables it.

**Status:** ✓ VERIFIED (all supporting artifacts pass) | ✗ FAILED (artifact missing/stub/unwired) | ? UNCERTAIN (needs human)

For each truth: identify supporting artifacts → check artifact status → check wiring → determine truth status.

**Example:** Truth "User can see existing messages" depends on Chat.tsx (renders), /api/chat GET (provides), Message model (schema). If Chat.tsx is a stub or API returns hardcoded [] → FAILED. If all exist, are substantive, and connected → VERIFIED.
</step>

<step name="verify_artifacts">
Use gsd-tools for artifact verification against must_haves in each PLAN:

```bash
for plan in "$PHASE_DIR"/*-PLAN.md; do
  ARTIFACT_RESULT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs verify artifacts "$plan")
  echo "=== $plan ===" && echo "$ARTIFACT_RESULT"
done
```

Parse JSON result: `{ all_passed, passed, total, artifacts: [{path, exists, issues, passed}] }`

**Artifact status from result:**
- `exists=false` → MISSING
- `issues` not empty → STUB (check issues for "Only N lines" or "Missing pattern")
- `passed=true` → VERIFIED (Levels 1-2 pass)

**Level 3 — Wired (manual check for artifacts that pass Levels 1-2):**
```bash
grep -r "import.*$artifact_name" src/ --include="*.ts" --include="*.tsx"  # IMPORTED
grep -r "$artifact_name" src/ --include="*.ts" --include="*.tsx" | grep -v "import"  # USED
```
WIRED = imported AND used. ORPHANED = exists but not imported/used.

| Exists | Substantive | Wired | Status |
|--------|-------------|-------|--------|
| ✓ | ✓ | ✓ | ✓ VERIFIED |
| ✓ | ✓ | ✗ | ⚠️ ORPHANED |
| ✓ | ✗ | - | ✗ STUB |
| ✗ | - | - | ✗ MISSING |
</step>

<step name="verify_wiring">
Use gsd-tools for key link verification against must_haves in each PLAN:

```bash
for plan in "$PHASE_DIR"/*-PLAN.md; do
  LINKS_RESULT=$(node ~/.claude/get-shit-done/bin/gsd-tools.cjs verify key-links "$plan")
  echo "=== $plan ===" && echo "$LINKS_RESULT"
done
```

Parse JSON result: `{ all_verified, verified, total, links: [{from, to, via, verified, detail}] }`

**Link status from result:**
- `verified=true` → WIRED
- `verified=false` with "not found" → NOT_WIRED
- `verified=false` with "Pattern not found" → PARTIAL

**Fallback patterns (if key_links not in must_haves):**

| Pattern | Check | Status |
|---------|-------|--------|
| Component → API | fetch/axios call to API path, response used (await/.then/setState) | WIRED / PARTIAL (call but unused response) / NOT_WIRED |
| API → Database | Prisma/DB query on model, result returned via res.json() | WIRED / PARTIAL (query but not returned) / NOT_WIRED |
| Form → Handler | onSubmit with real implementation (fetch/axios/mutate/dispatch), not console.log/empty | WIRED / STUB (log-only/empty) / NOT_WIRED |
| State → Render | useState variable appears in JSX (`{stateVar}` or `{stateVar.property}`) | WIRED / NOT_WIRED |

Record status and evidence for each key link.
</step>

<step name="verify_requirements">
If REQUIREMENTS.md exists:
```bash
grep -E "Phase ${PHASE_NUM}" .planning/REQUIREMENTS.md 2>/dev/null
```

For each requirement: parse description → identify supporting truths/artifacts → status: ✓ SATISFIED / ✗ BLOCKED / ? NEEDS HUMAN.
</step>

<step name="stage1_gate">
**Stage 1 Gate: Evaluate spec compliance before proceeding to code quality.**

Determine spec compliance status:
- **spec_passed:** All truths VERIFIED, all artifacts pass levels 1-3, all key links WIRED, all requirements SATISFIED
- **spec_gaps:** Any truth FAILED, artifact MISSING/STUB, key link NOT_WIRED, or requirement BLOCKED

**If spec_gaps:** Record all spec issues. Proceed to Stage 2 ONLY for informational purposes — spec gaps are the primary blockers.

**If spec_passed:** Proceed to Stage 2 as the determining gate.

Report stage 1 result in VERIFICATION.md under `## Stage 1: Spec Compliance`.
</step>

<!-- ═══════════════════════════════════════════════════════════════════ -->
<!-- STAGE 2: CODE QUALITY — Is the delivered code well-built?          -->
<!-- ═══════════════════════════════════════════════════════════════════ -->

<step name="code_quality_review">
Before scanning for anti-patterns, perform a lightweight code quality review on files modified in this phase.

**Extract modified files from SUMMARY.md:**
```bash
grep -h "files_modified\|key-files\|created\|modified" "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null | grep -oE "[a-zA-Z0-9_./-]+\.[a-z]+" | sort -u
```

**For each modified file, check:**

1. **Linting** — Run the project's linter if available:
```bash
npm run lint -- --quiet 2>/dev/null || npx eslint {file} --quiet 2>/dev/null
```

2. **Type checking** — Run type checker if TypeScript:
```bash
npx tsc --noEmit 2>/dev/null | grep -c "error" || echo "0"
```

3. **Test execution** — Run related tests:
```bash
npm test -- --passWithNoTests 2>/dev/null
```

**Record results as a quality gate:**

| Check | Status | Details |
|-------|--------|---------|
| Lint | pass/fail | {error count} |
| Types | pass/fail | {error count} |
| Tests | pass/fail | {failure count} |

**If any check fails:** Record as a blocker in the verification report. These are automated quality issues that should be fixed before UAT.

**If all pass:** Proceed to anti-pattern scanning.

This gate ensures code quality issues are caught programmatically before asking the user to manually test features.
</step>

<step name="scan_antipatterns">
Extract files modified in this phase from SUMMARY.md, scan each:

| Pattern | Search | Severity |
|---------|--------|----------|
| TODO/FIXME/XXX/HACK | `grep -n -E "TODO\|FIXME\|XXX\|HACK"` | ⚠️ Warning |
| Placeholder content | `grep -n -iE "placeholder\|coming soon\|will be here"` | 🛑 Blocker |
| Empty returns | `grep -n -E "return null\|return \{\}\|return \[\]\|=> \{\}"` | ⚠️ Warning |
| Log-only functions | Functions containing only console.log | ⚠️ Warning |

Categorize: 🛑 Blocker (prevents goal) | ⚠️ Warning (incomplete) | ℹ️ Info (notable).
</step>

<!-- ═══════════════════════════════════════════════════════════════════ -->
<!-- STAGE 3: MULTI-PERSPECTIVE REVIEW (optional, config-driven)       -->
<!-- ═══════════════════════════════════════════════════════════════════ -->

<step name="stage3_parallel_review">
**Stage 3 only runs when enabled in config.**

```bash
PARALLEL_REVIEW=$(node -e "try{const c=require('./.planning/config.json');console.log(c.workflow?.parallel_review||false)}catch{console.log(false)}")
```

**If `PARALLEL_REVIEW` is `false` or config missing:** Skip Stage 3 entirely. Proceed to `identify_human_verification`.

**If `PARALLEL_REVIEW` is `true`:**

Execute the review-phase workflow as a nested operation:

```
Task(
  prompt="Run multi-perspective code review for phase {phase_number}.
Phase directory: {phase_dir}
Phase goal: {phase_goal}
Execute the review-phase workflow end-to-end.
Return the review status (review_clean, review_passed, review_blocked) and the report path.",
  subagent_type="gsd-reviewer",
  description="Stage 3: Multi-perspective review for Phase {phase}"
)
```

Alternatively, if the orchestrator IS the main agent (not a subagent), invoke the review-phase workflow inline by following @~/.claude/get-shit-done/workflows/review-phase.md.

**Read the review result:**
```bash
REVIEW_STATUS=$(grep "^status:" "$PHASE_DIR"/*-REVIEW.md 2>/dev/null | cut -d: -f2 | tr -d ' ')
REVIEW_CRITICAL=$(grep "^critical:" "$PHASE_DIR"/*-REVIEW.md 2>/dev/null | cut -d: -f2 | tr -d ' ')
```

**Stage 3 gate:**
- `review_clean` or `review_passed` → Stage 3 passes
- `review_blocked` → Stage 3 fails (critical findings exist)

Record Stage 3 result in VERIFICATION.md under `## Stage 3: Multi-Perspective Review`.
</step>

<step name="identify_human_verification">
**Always needs human:** Visual appearance, user flow completion, real-time behavior (WebSocket/SSE), external service integration, performance feel, error message clarity.

**Needs human if uncertain:** Complex wiring grep can't trace, dynamic state-dependent behavior, edge cases.

Format each as: Test Name → What to do → Expected result → Why can't verify programmatically.
</step>

<step name="determine_status">
Combine all applicable stages into final status:

**passed:** Stage 1 spec_passed AND Stage 2 quality_passed AND (Stage 3 skipped OR Stage 3 review_passed/review_clean).

**gaps_found:** Stage 1 spec_gaps OR Stage 2 quality_failed OR Stage 3 review_blocked. Report which stage(s) failed.

**human_needed:** All stages pass automated checks but human verification items remain.

**Score:** `verified_truths / total_truths` (Stage 1) + `quality_checks_passed / total_quality_checks` (Stage 2) + review status (Stage 3, if run)

**In the report, separate stages clearly:**

```markdown
## Stage 1: Spec Compliance
Status: {spec_passed | spec_gaps}
Score: {N}/{M} truths verified

[truths table, artifacts table, wiring table, requirements table]

## Stage 2: Code Quality
Status: {quality_passed | quality_failed}

[lint/types/tests results, anti-pattern scan, quality issues]

## Stage 3: Parallel Code Review
Status: {review_clean | review_passed | review_blocked | skipped}

{If run: summary table from REVIEW.md, critical findings inline}
{If skipped: "Stage 3 not enabled. Enable with workflow.parallel_review: true in .planning/config.json"}
```

This separation lets fix plans target the right problem: spec gaps need implementation work, quality gaps need refactoring work, review findings need targeted fixes.
</step>

<step name="generate_fix_plans">
If gaps_found:

1. **Cluster related gaps:** API stub + component unwired → "Wire frontend to backend". Multiple missing → "Complete core implementation". Wiring only → "Connect existing components".

2. **Generate plan per cluster:** Objective, 2-3 tasks (files/action/verify each), re-verify step. Keep focused: single concern per plan.

3. **Order by dependency:** Fix missing → fix stubs → fix wiring → verify.
</step>

<step name="create_report">
```bash
REPORT_PATH="$PHASE_DIR/${PHASE_NUM}-VERIFICATION.md"
```

Fill template sections: frontmatter (phase/timestamp/status/score), goal achievement, artifact table, wiring table, requirements coverage, anti-patterns, human verification, gaps summary, fix plans (if gaps_found), metadata.

See ~/.claude/get-shit-done/templates/verification-report.md for complete template.
</step>

<step name="return_to_orchestrator">
Return status (`passed` | `gaps_found` | `human_needed`), score (N/M must-haves), report path.

If gaps_found: list gaps + recommended fix plan names.
If human_needed: list items requiring human testing.

Orchestrator routes: `passed` → update_roadmap | `gaps_found` → create/execute fixes, re-verify | `human_needed` → present to user.
</step>

</process>

<success_criteria>

**Stage 1: Spec Compliance**
- [ ] Must-haves established (from frontmatter or derived)
- [ ] All truths verified with status and evidence
- [ ] All artifacts checked at all three levels
- [ ] All key links verified
- [ ] Requirements coverage assessed (if applicable)
- [ ] Stage 1 gate evaluated (spec_passed | spec_gaps)

**Stage 2: Code Quality**
- [ ] Code quality gate run (lint, types, tests)
- [ ] Anti-patterns scanned and categorized

**Stage 3: Parallel Code Review (if enabled)**
- [ ] Config checked for workflow.parallel_review
- [ ] If enabled: 3 independent reviewer subagents spawned, REVIEW.md created
- [ ] If enabled: review status incorporated into final determination
- [ ] If disabled: Stage 3 marked as skipped in report

**Final**
- [ ] Human verification items identified
- [ ] Overall status determined from all applicable stages
- [ ] Fix plans generated (if gaps_found) — tagged by stage (spec vs quality vs review)
- [ ] VERIFICATION.md created with multi-stage report
- [ ] Results returned to orchestrator

</success_criteria>
