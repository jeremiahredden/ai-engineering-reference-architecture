# Regression Eval Suites

> **Audience.** Engineers and tech leads building the regression-suite discipline. Anyone who has seen the same AI bug return in production six months after it was "fixed." **Scope.** The *engineering* practice of building regression suites from production bugs and incidents — case capture, suite organization, CI integration, lifecycle. Pair with [golden-set-design.md](./golden-set-design.md), [eval-engineering-playbook.md](./eval-engineering-playbook.md), [eval-gate-architecture.md](./eval-gate-architecture.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Production AI bugs recur. A team fixes a quality issue; six months later, an unrelated change (prompt revision, model upgrade, corpus refresh) reintroduces the same bug. Users notice; the team rediscovers the bug; the team fixes it again. Without a regression discipline, the team is on a cycle of fix-and-re-fix.

The regression-suite discipline breaks the cycle. Every fixed bug becomes a permanent eval case. The case persists in the regression suite. Every PR that touches the system runs against the regression suite (or a fast subset). The bug cannot return without the eval gate catching it first.

The discipline is operationally light: capture the failing case at fix time; add to the regression suite; run the suite in CI. The leverage is enormous: the regression suite is the team's accumulated knowledge of failure modes, encoded as automated tests.

The [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 6 introduces the pattern. This document is the depth — case capture mechanics, suite organization at scale, integration with the broader CI / observability stack, lifecycle management.

This document is opinionated about three things:

1. **Every fixed production bug becomes a regression case.** No exceptions. The case is added as part of the fix; the PR does not merge without it.
2. **The regression suite grows continuously.** Unlike the golden set (which is curated and balanced), the regression suite accumulates by design. Cases are removed only when the underlying scenario becomes obsolete.
3. **The regression suite is its own structural object** — separate from the golden set conceptually, even if they share infrastructure. Different growth pattern, different curation discipline, different purpose.

Structure: (2) the bug-to-regression-case workflow; (3) case structure for regressions; (4) suite organization at scale; (5) CI integration; (6) the bug → fix → case process gate; (7) suite lifecycle; (8) integration with broader eval; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The bug-to-regression-case workflow

The core workflow that turns production bugs into eval cases.

### 2.1 The workflow steps

When a production AI bug is identified:

1. **Reproduce.** Re-run the failing question (or pull the trace from the failed interaction).
2. **Capture.** Save the original question, the original (wrong) answer, the actual circumstances (corpus version, model version, prompt version at the time of failure).
3. **Diagnose.** Identify the root cause.
4. **Fix.** Apply the fix (prompt change, model rollback, corpus update, etc.).
5. **Verify.** Confirm the question now produces the right answer.
6. **Regression case.** Construct the regression case: the question, the correct answer, the rubric. Add to the regression suite.
7. **Test.** Run the suite (now including the new case); verify the case passes with the fix.
8. **Close.** Bug ticket closes only after the regression case is added.

The discipline is the closure step: the bug cannot close without the case. Without this gate, the practice degrades; cases get missed; bugs return.

### 2.2 The fast capture pattern

The bug fix is usually time-sensitive. The capture process must be fast — adding the regression case should take minutes, not hours:

- A one-command tool extracts the necessary fields from the trace.
- The case template is pre-filled; the engineer adds only the corrected expected answer and the diagnostic notes.
- PR to add the case is small and fast-reviewed.

If capture is slow, the practice degrades. Engineers will skip capture under deadline pressure, and the regression suite grows incomplete.

### 2.3 The capture-without-fix pattern

Sometimes a bug is observed but not immediately fixable (the root cause requires substantial work; a workaround is shipped). The regression case is captured anyway:

- The case is added to a "known-failing" suite — runs in CI but does not block.
- When the fix lands, the case moves to the active regression suite.
- The case being in the known-failing suite is itself a tracked metric (we have N known failures awaiting fix).

This keeps the bug visible until it is fully resolved.

### 2.4 The user-reported vs system-detected distinction

Production bugs come from two sources:

- **User-reported.** A user said the answer was wrong; the team investigates and fixes.
- **System-detected.** Online judge sampling flagged a quality issue; the team investigates and fixes.

Both sources produce regression cases by the same workflow. The differences:

- User-reported cases often have more user context (the user's frustration framing helps articulate why this case matters).
- System-detected cases often have richer trace data (the online judge already produced the verdict and justification).

Both become regression cases. The provenance is tracked.

### 2.5 The triage discipline

Not every reported bug becomes a regression case. Triage:

- Is the bug actually a bug? (Sometimes user reports are misunderstandings.)
- Is the bug reproducible? (Non-reproducible bugs can't be tested.)
- Is the bug in scope? (Some reports are out-of-scope requests, not bugs.)

Bugs that pass triage become regression cases. Triage is recorded; out-of-scope reports inform feature roadmap.

---

## 3. Case structure for regressions

Regression cases share the structure of golden-set cases (per [golden-set-design.md](./golden-set-design.md)) with regression-specific additions.

### 3.1 The case structure

```yaml
case_id: REG-2026-0042
question: "What's the dosing for warfarin in a 78-year-old patient with renal impairment?"

expected_answer: |
  Warfarin dosing in elderly patients with renal impairment requires individualized
  titration based on INR monitoring. Initial dose: 2-2.5mg daily, lower than the
  standard 5mg starting dose. Cite: AHA 2024 Anticoagulation Guidelines section 4.3.

required_claims:
  - claim: "Initial dose should be 2-2.5mg daily, lower than standard"
    must_cite: "clinical-guideline:aha-anticoag-2024:section-4.3"
  - claim: "INR monitoring is required"
    must_cite: "clinical-guideline:aha-anticoag-2024"

# Regression-specific metadata
regression_source:
  bug_id: BUG-2026-0089
  reported_date: 2026-03-18
  reported_by: clinician at Mercy Cleveland
  detection_source: user-reported
  fix_date: 2026-03-22
  fix_pr: meridian-prompts#2147
  root_cause: |
    The supervisor prompt's elderly-patient handling was too brief; the model
    defaulted to standard dosing instead of the reduced elderly dose. Fixed by
    expanding the elderly-patient-considerations section of the supervisor prompt.

# What the wrong answer was (for documentation)
original_wrong_answer: |
  Warfarin starting dose is 5mg daily, titrated based on INR.

class_tags:
  question_type: lookup
  domain: clinical
  complexity: medium  # requires recognizing elderly+renal modifier
  stakes: critical    # wrong dose can cause harm
  failure_history: fixed_once

scoring:
  rubric: meridian_clinical_rubric_v2
  # Same scoring as a golden-set case
```

### 3.2 The regression-source field

The `regression_source` field captures the bug's provenance:

- bug_id linking to the bug tracker.
- detection source (user-reported / system-detected).
- root cause documentation.
- fix PR reference.

The provenance is useful for:

- Aggregate analysis ("what fraction of our regressions came from user reports vs system detection?").
- Audit (regulatory reviewers can trace from regression case to fixed bug to deployed fix).
- Onboarding (new engineers can read the regression history).

### 3.3 The original_wrong_answer field

Capturing what the wrong answer was (not just the right one) helps:

- Document the failure mode for the team's understanding.
- Calibrate the judge (the wrong answer should fail the rubric; the right answer should pass).
- Future investigation (similar failure patterns can be matched against the historical wrong answer).

### 3.4 The minimum capture (when fast is needed)

Under deadline pressure, the minimum capture is:

- question
- expected_answer (the corrected answer)
- regression_source.bug_id
- regression_source.fix_date

The rest is enriched later. The PR adds the case; subsequent PRs enrich the metadata.

### 3.5 The duplicate detection

When adding a regression case, check for duplicates:

- A near-identical question already in the suite (the same bug has been fixed before).
- A case with the same required_claims (the same scenario tested another way).

Duplicates are not always wrong (sometimes a different angle on the same scenario is useful), but they should be deliberate.

---

## 4. Suite organization at scale

A growing regression suite needs organization.

### 4.1 The categorization

Regression cases are tagged by:

- **Class.** Same case-class taxonomy as the golden set per [golden-set-design.md](./golden-set-design.md) section 3.
- **Severity.** What was the severity of the original bug? (Critical / high / medium / low.)
- **Failure mode.** What pattern of failure did this case represent? (Hallucination / wrong citation / missed claim / off-topic / wrong refusal.)
- **Year added.** For ageing analysis.

### 4.2 Subdirectory structure

A regression suite of 200+ cases benefits from directory organization:

```
meridian-eval/cases/regression/
├── clinical/
│   ├── dosing/
│   ├── drug-interaction/
│   ├── refusal/
│   └── citation/
├── coordination/
│   ├── messaging/
│   └── scheduling/
├── safety/
│   ├── escalation/
│   └── pii-redaction/
└── general/
```

Engineers can find related cases; new cases land in the appropriate subdirectory.

### 4.3 The subset definition

For CI integration (per section 5), subsets of the regression suite are defined:

- **Critical regression subset.** ~20-40 cases representing critical / high-severity historical bugs. Runs on every PR (fast).
- **Class-specific regression subsets.** Each class can run as a subset for class-focused changes.
- **Full regression suite.** Runs nightly and on release candidates.

Subsets are defined declaratively (a YAML manifest listing case IDs); changes go through PR review.

### 4.4 The age-stratified organization

Older cases (regressions from 1+ years ago) may be less likely to recur in modern code paths. The discipline:

- Tag cases with year added.
- Periodically review old cases: are they still testing relevant failure modes?
- Retire cases whose failure modes are now structurally impossible (the underlying code path was removed, the integration was decommissioned).

Retirement is rare; the default is to keep.

---

## 5. CI integration

The regression suite is the load-bearing CI signal.

### 5.1 The CI matrix

| Trigger | Suite | Blocking |
|---|---|---|
| Per-PR (fast) | Critical regression subset (~30 cases) | Yes |
| Per-PR (class-relevant) | Class-specific subset (~20-40 cases) | Yes |
| Nightly on main | Full regression suite | Trend-only, not blocking |
| Release candidate | Full regression suite | Yes |
| Hotfix | Critical regression subset only | Yes |

Per-PR runs gate merges; release-candidate runs gate releases; nightly runs produce trend data.

### 5.2 The threshold setting

Per [eval-gate-architecture.md](./eval-gate-architecture.md), the threshold for regression suite is stricter than for golden set:

- Regression cases must not regress. A regression case that previously passed and now fails is a hard block.
- New regression cases are expected to pass from their introduction (they are added with the fix that makes them pass).

This is different from golden-set thresholds where a small percentage drop may be acceptable. For regression: zero tolerance on previously-passing cases.

### 5.3 The override pattern

For intentional regressions (a planned tightening of refusal behavior that drops pass-rate on cases that were too lenient):

- The override pattern per [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 5.4 applies.
- Override label on the PR with justification.
- Reviewer approval explicitly acknowledging the trade-off.
- Logged for post-deploy review.

Overrides on the regression suite are rare and reviewed seriously.

### 5.4 The CI cost

Regression suite cost:
- Critical subset (30 cases × ~3 judge calls × ~$0.012) = ~$1 per PR run.
- Full suite (200+ cases × ~3 judge calls × ~$0.012) = ~$8 per full run.
- Nightly + release-candidate + per-PR: a few hundred dollars per month.

Cost is justified; a single missed regression that the suite catches typically exceeds the monthly suite cost.

### 5.5 The flakiness handling

Eval cases can be flaky (model nondeterminism produces occasional failures even when behavior is correct). The discipline:

- Initial flakiness: when a case is added, run it 5 times; pass rate should be ≥ 80%.
- Production flakiness: track per-case pass rate over time; flag cases below 90%.
- Flaky cases are investigated: is the rubric too strict? Is the model genuinely inconsistent? Is the case poorly designed?
- Fix or quarantine: tighten the rubric, redesign the case, or quarantine while investigating.

Without flakiness discipline, the team learns to ignore failures, defeating the gate.

---

## 6. The bug → fix → case process gate

The discipline that makes the practice real: the bug cannot close without the regression case.

### 6.1 The process

The bug-fix PR workflow:

1. Engineer fixes the bug; opens a PR.
2. PR includes the fix AND the regression case (or a follow-up PR with the case).
3. CI runs (including the new case); confirms the case passes.
4. Reviewer approves the fix and the case.
5. Merge; bug ticket can close.

### 6.2 The enforcement

The enforcement happens at the bug-tracker level (bug cannot close without a linked regression case) and at the PR-review level (reviewer asks "where's the regression case?" if missing).

Some teams automate the linkage: a bug ticket's closing comment must include a regression case ID. Tooling enforces.

### 6.3 The retroactive capture

Bugs that were fixed before the discipline was in place can be retroactively captured. The team:

- Reviews recent bug history.
- For each fixed bug: can we reproduce the original failure now? Can we construct a regression case?
- Backfill the suite with retroactive cases.

This is one-time work; the suite catches up with the historical bug record.

### 6.4 The exception handling

Some bugs do not warrant regression cases:

- Trivial typos in prompts.
- One-off incidents that cannot recur structurally.
- Environment-specific bugs (a one-time deployment misconfiguration).

Exceptions are documented in the bug ticket; the reviewer ratifies the exception.

If exceptions become routine, the discipline is failing.

### 6.5 The metric

"What percentage of fixed bugs in the last quarter have regression cases?" is a process-health metric. Below 95% indicates a discipline gap.

---

## 7. Suite lifecycle

The regression suite is a living artifact.

### 7.1 The growth shape

Unlike the golden set, the regression suite grows monotonically by design. Every bug fix adds a case; cases are rarely removed.

Growth shape over time:

- Year 1: ~50-100 cases.
- Year 2: 100-200 cases.
- Year 3+: 200-500 cases at maturity.

The growth pace correlates with bug fix rate, which correlates with system maturity (early systems have many bugs; mature systems have fewer).

### 7.2 The retirement criteria

Cases are retired only when:

- The underlying code path was removed (the failure mode is structurally impossible now).
- The feature was deprecated.
- The case is genuinely redundant (a clearly-superior case covers the same scenario).

Retirement is PR-reviewed; the retirement rationale is documented.

### 7.3 The case ageing

As cases age:

- The failure mode they protect against may become structurally less likely (the broader codebase evolves).
- The cost of running them remains (the case still consumes CI time and judge cost).

The discipline: annually review cases older than 2 years. Confirm they still test relevant failure modes. Retire those that don't.

### 7.4 The aggregate health metrics

For the regression suite as a whole, track:

- Total case count.
- Growth rate (cases added per month).
- Per-class distribution.
- Per-severity distribution.
- Coverage by year (how many cases from each year are still active).
- Pass rate trend (should be very near 100%; drops indicate real regressions).

Dashboard visibility per [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) and similar.

---

## 8. Integration with broader eval

The regression suite is one component of the broader eval practice.

### 8.1 Relationship to golden set

Regression suite and golden set are separate structural objects:

- **Golden set:** curated, balanced, designed for coverage of case classes.
- **Regression suite:** accumulated, grows monotonically, captures failure modes.

Some cases bridge: a case added to the golden set during initial design that later failed in production. The case stays in the golden set; if the failure is significant, it may also enter the regression suite (as a duplicate, intentionally).

### 8.2 Relationship to online judge

Per [online-eval-and-feedback.md](./online-eval-and-feedback.md), the online judge runs against production traffic. Production failures detected by the online judge feed into the regression-case workflow.

### 8.3 Relationship to eval gate

Per [eval-gate-architecture.md](./eval-gate-architecture.md), the gate runs both golden set and regression suite on PRs. The thresholds differ; the regression suite has zero tolerance for previously-passing-now-failing cases.

### 8.4 Relationship to release artifacts

Per [release-artifacts-for-ai.md](../cicd-and-eval-gates/) (coming), each release pins the eval suite version including the regression suite. A release knows which regression cases were in the suite at the time of release; auditors can confirm coverage.

### 8.5 Relationship to incident response

Per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md), production incidents follow runbooks. The runbook ends with: "add the regression case." The runbook is the enforcement point for the process gate.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The regression suite

Meridian's clinical regression suite has ~87 cases at 2026-Q2.

By source:
- User-reported: 52 cases.
- System-detected (online judge): 23 cases.
- Internal dogfooding: 12 cases.

By severity:
- Critical (clinical-decision impacting): 18 cases.
- High (clinical-content quality): 38 cases.
- Medium (formatting / completeness): 24 cases.
- Low (style / preference): 7 cases.

By year added:
- 2024 (pre-GA dogfooding): 11 cases.
- 2025: 38 cases.
- 2026 YTD: 38 cases (still adding).

### 9.2 The bug-to-regression workflow in practice

For BUG-2026-0089 (the warfarin dosing bug in the case example above):

1. **2026-03-18:** Clinician at Mercy Cleveland reports that the Care Coordinator gave standard warfarin dosing for an elderly patient with renal impairment.
2. **2026-03-18 14:00:** On-call triages; reproduces the bug; opens BUG-2026-0089.
3. **2026-03-19:** Diagnosis: supervisor prompt does not adequately handle elderly-patient context.
4. **2026-03-20-21:** Fix developed: supervisor prompt update; eval-validated.
5. **2026-03-22:** Fix PR opened. PR includes:
   - The supervisor prompt change.
   - REG-2026-0042 (the regression case).
6. **2026-03-22:** CI runs. Regression case passes with the fix. Without the fix, the case would have failed.
7. **2026-03-22:** PR merged; deployed; bug closed.

The full cycle: 4 days. The regression case prevents this specific failure from recurring.

### 9.3 The suite organization

The Meridian regression suite organization:

```
meridian-eval/cases/regression/
├── clinical/
│   ├── dosing/                  (18 cases)
│   ├── drug-interaction/        (15 cases)
│   ├── citation/                (11 cases)
│   └── refusal/                 (8 cases)
├── coordination/
│   ├── messaging/               (12 cases)
│   └── scheduling/              (6 cases)
├── safety/
│   ├── escalation/              (9 cases)
│   └── pii-handling/            (5 cases)
└── general/                     (3 cases)
```

Engineers know where to find related cases; new cases land in the appropriate subdirectory.

### 9.4 The CI integration

- **Per-PR fast subset:** 28 critical-severity regression cases. Runs in ~6 minutes. Blocks merge on any regression failure.
- **Class-specific subsets:** When a PR touches a specific class (clinical / coordination / safety), the corresponding subset runs additionally.
- **Nightly on main:** Full regression suite (~87 cases). ~25 minutes. Trend dashboard; not blocking.
- **Release candidate:** Full regression suite + adversarial subset. Blocks release on regression.

### 9.5 The 2026-Q2 regression discipline metric

In 2026-Q2 to date:

- 14 production bugs filed.
- 12 bugs fixed.
- 12 regression cases added.
- 100% process-gate adherence.
- 0 regressions on previously-passing regression cases.

The discipline is holding. The team's confidence in the regression suite is high; the suite is the trusted defense against bug recurrence.

### 9.6 The platform discipline

- Bug-fix PRs include regression cases.
- Bug tickets cannot close without a linked regression case (enforced via PR template).
- Quarterly retroactive backfill (for bugs predating the discipline).
- Annual case-ageing review.
- Per-class coverage and growth-rate dashboards.

---

## 10. Anti-patterns

### 10.1 "Fix without regression case"

The team fixes bugs but does not add regression cases. The same bug recurs months later.

**Corrective.** Process gate: bug cannot close without regression case per section 6.

### 10.2 "Regression suite runs only nightly"

The full regression suite runs nightly but not on PRs. A PR that introduces a regression is merged; the nightly catches it the next morning; the team scrambles.

**Corrective.** Critical regression subset per-PR per section 5.1; trade-off favors blocking the PR over scrambling the next morning.

### 10.3 "Suite shares no structure with golden set"

Regression suite uses its own case format; engineers maintain two sets of tooling.

**Corrective.** Shared case structure per [golden-set-design.md](./golden-set-design.md); same eval runner; same judge.

### 10.4 "No provenance"

Regression cases have no `regression_source` field; the connection to the original bug is lost.

**Corrective.** Provenance per section 3.2.

### 10.5 "Capture is slow"

The regression-case workflow is documented but slow (10+ steps); engineers skip under deadline pressure.

**Corrective.** Fast-capture tooling per section 2.2; one-command extract from the trace.

### 10.6 "Flaky cases ignored"

The suite has cases that flake (~70% pass rate); the team learns to "rerun if it fails." Real regressions get rerun and ignored.

**Corrective.** Flakiness discipline per section 5.5; quarantine flaky cases until they're investigated.

### 10.7 "Cases never retired"

The suite has cases from 2020 testing code paths that no longer exist; runtime grows; insights buried.

**Corrective.** Annual ageing review per section 7.3.

### 10.8 "Process gate is informal"

The "rule" that bugs include regression cases is a convention, not enforced. Some bugs close without cases.

**Corrective.** PR template / bug tracker integration enforces; metric tracked per section 6.5.

---

## 11. Findings (sprint-assignable)

### REG-001 — Severity: Critical
**Finding.** Fixed production bugs do not become regression cases; recurring bugs are observed.
**Recommendation.** Process gate per section 6: bug cannot close without a linked regression case.
**Owner.** ai-platform-eng + sre, sprint N+1.

### REG-002 — Severity: Critical
**Finding.** Regression suite does not exist; bug recurrences are detected only via re-reports.
**Recommendation.** Build the regression suite as a separate structural object; populate from recent bug history.
**Owner.** ai-platform-eng, sprint N+1.

### REG-003 — Severity: High
**Finding.** Regression suite runs only nightly; PRs that introduce regressions merge.
**Recommendation.** Critical subset per-PR per section 5.1.
**Owner.** ai-platform-eng + sre, sprint N+2.

### REG-004 — Severity: High
**Finding.** Capture process is slow; engineers skip under deadline pressure.
**Recommendation.** Fast-capture tooling per section 2.2; minimum capture template.
**Owner.** ai-platform-eng, sprint N+2.

### REG-005 — Severity: High
**Finding.** Regression cases lack provenance; the connection to the original bug is lost over time.
**Recommendation.** Provenance metadata per section 3.2.
**Owner.** ai-platform-eng, sprint N+2.

### REG-006 — Severity: High
**Finding.** Flaky cases are ignored or routinely re-run; the team has learned to disregard regression failures.
**Recommendation.** Flakiness discipline per section 5.5; quarantine and investigate.
**Owner.** ai-platform-eng, sprint N+2.

### REG-007 — Severity: High
**Finding.** Suite has zero tolerance threshold not configured; regression failures are treated as merely concerning.
**Recommendation.** Zero tolerance for previously-passing regression cases per section 5.2.
**Owner.** ai-platform-eng + sre, sprint N+3.

### REG-008 — Severity: Medium
**Finding.** Process-gate metric is not tracked; discipline gaps go unnoticed.
**Recommendation.** Track per section 6.5; alert if < 95%.
**Owner.** ai-platform-eng team lead, sprint N+3.

### REG-009 — Severity: Medium
**Finding.** Retroactive backfill of pre-discipline bugs has not been done; historical failure modes are not covered.
**Recommendation.** One-time backfill effort per section 6.3.
**Owner.** ai-platform-eng, sprint N+3.

### REG-010 — Severity: Medium
**Finding.** Suite organization is flat; engineers cannot find related cases.
**Recommendation.** Subdirectory structure per section 4.2.
**Owner.** ai-platform-eng, sprint N+3.

### REG-011 — Severity: Medium
**Finding.** Class-specific subset definitions do not exist; PRs touching a specific class run the same fast subset as PRs touching anything.
**Recommendation.** Class-specific subsets per section 4.3.
**Owner.** ai-platform-eng, sprint N+3.

### REG-012 — Severity: Medium
**Finding.** Suite case-ageing review is not scheduled; old cases accumulate.
**Recommendation.** Annual review per section 7.3.
**Owner.** ai-platform-eng team lead, sprint N+4.

### REG-013 — Severity: Medium
**Finding.** Override pattern for the regression suite is unused or undocumented; intentional regressions become contested PRs.
**Recommendation.** Override per section 5.3; documented with explicit justification requirement.
**Owner.** ai-platform-eng + sre, sprint N+3.

### REG-014 — Severity: Medium
**Finding.** Coverage dashboards for the regression suite are absent; team cannot see where coverage is thin.
**Recommendation.** Per-class coverage and growth-rate dashboards per section 7.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### REG-015 — Severity: Medium
**Finding.** Known-failing suite pattern is not used; bugs awaiting fix are not captured as eval cases.
**Recommendation.** Known-failing suite per section 2.3; cases move to active suite when fixed.
**Owner.** ai-platform-eng, sprint N+4.

### REG-016 — Severity: Low
**Finding.** Duplicate-detection workflow does not exist; suite accumulates near-duplicate cases.
**Recommendation.** Quarterly duplicate scan; consolidation.
**Owner.** ai-platform-eng, sprint N+5.

### REG-017 — Severity: Low
**Finding.** Original_wrong_answer field is not captured; the team forgets what the failure mode looked like.
**Recommendation.** Field per section 3.3.
**Owner.** ai-platform-eng, sprint N+5.

### REG-018 — Severity: Low
**Finding.** Regression-suite documentation is thin; new contributors do not understand the discipline.
**Recommendation.** Documentation alongside the case structure; include in onboarding.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting from "we fix bugs but don't capture regression cases":

- [ ] **Sprint 0 — design.** Decide the regression-case structure (per section 3) and suite location.
- [ ] **Sprint 1 — first cases.** Backfill 10-20 high-value cases from recent bug history. Validate the workflow.
- [ ] **Sprint 1 — workflow tooling.** Build the fast-capture tool per section 2.2; one-command extract.
- [ ] **Sprint 2 — process gate.** PR template / bug tracker enforcement: bug cannot close without case.
- [ ] **Sprint 2 — CI integration.** Critical subset runs per-PR; full suite runs nightly.
- [ ] **Sprint 3 — flakiness discipline.** Initial flakiness check on new cases; quarantine on production flakiness.
- [ ] **Sprint 3 — provenance.** Provenance metadata on every case.
- [ ] **Sprint 4 — coverage dashboards.** Per-class growth and pass rate; visible to the team.
- [ ] **Sprint 4 — process-gate metric.** Track adherence; alert on gaps.
- [ ] **Sprint 5 — retroactive backfill.** Bugs predating the discipline get cases.
- [ ] **Ongoing — discipline.** Every bug → case; annual ageing review.

A team that completes this sequence has the regression discipline that prevents bug recurrence. A team that skips it pays in re-fixed bugs and lost user trust.

---

## 13. References

- This repo: [eval-engineering/eval-engineering-playbook.md](./eval-engineering-playbook.md) — the broader practice (section 6).
- This repo: [eval-engineering/golden-set-design.md](./golden-set-design.md) — the curated suite alongside this.
- This repo: [eval-engineering/llm-as-judge-patterns.md](./llm-as-judge-patterns.md) — the judge that scores regression cases.
- This repo: [eval-engineering/eval-gate-architecture.md](./eval-gate-architecture.md) — the CI gate integration.
- This repo: [eval-engineering/online-eval-and-feedback.md](./online-eval-and-feedback.md) — the production-feedback source for regression cases.
- This repo: [cicd-and-eval-gates/release-artifacts-for-ai.md](../cicd-and-eval-gates/) (coming) — release pinning.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — runbooks that end in regression-case addition.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
