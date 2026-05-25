# Eval Gate Architecture

> **Audience.** Engineers building or refactoring the CI integration for AI eval. Tech leads tired of "the eval is on a dashboard but doesn't block anything." **Scope.** The *engineering* practice of the eval gate — where it sits in CI/CD, threshold setting, fast vs full suite splits, override patterns, integration with release artifacts. Pair with [eval-engineering-playbook.md](./eval-engineering-playbook.md) (the broader practice), [golden-set-design.md](./golden-set-design.md) and [regression-eval-suites.md](./regression-eval-suites.md) (what the gate runs against), [release-artifacts-for-ai.md](../cicd-and-eval-gates/) (the broader CI/CD context, coming). **Worked client.** Meridian Health.

---

## 1. Why this document exists

An eval that does not gate anything is a dashboard. A dashboard does not prevent regressions. The single most important step in any team's eval-engineering journey is making the eval *block deployment when it regresses*. The discipline is what turns the eval suite from "interesting trend data" into "active quality control."

Most teams' eval gates fail in a few predictable ways: thresholds set too tight (constant noise; team learns to ignore); thresholds set too loose (gate never fires; useless); override mechanism missing (legitimate intentional changes are blocked forever); override mechanism too easy (gate is routinely bypassed).

This document is the discipline that makes the gate work in practice. Threshold calibration; fast / full suite split; runtime budget; override pattern with audit; rollback on post-merge failure detection. The gate is the *enforcement mechanism* for the entire eval practice; without it, the practice loses teeth.

The [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 5 introduces the pattern; this document is the depth.

This document is opinionated about three things:

1. **The gate blocks on regression, not on absolute pass rate.** The discipline is "no worse than the trailing baseline." Absolute floors apply too, but the primary signal is regression vs baseline.
2. **Fast subset gates PRs; full suite gates releases.** Per-PR runs must be fast (~10 min); the full suite runs nightly and on release candidates. The split is operationally critical.
3. **Override is audited, not absent.** Sometimes a regression is intentional (deliberate tightening of refusal behavior, planned format change). The override mechanism exists with audit; without it, the team has no path to ship intentional changes.

Structure: (2) the gate hierarchy; (3) fast subset selection; (4) threshold calibration; (5) the override pattern; (6) post-merge detection and rollback; (7) integration with release artifacts; (8) cost considerations; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The gate hierarchy

Eval gates operate at multiple points in the CI/CD pipeline; each has a different role.

### 2.1 The gate layers

| Layer | Trigger | What runs | Blocking |
|---|---|---|---|
| Pre-commit | Local developer environment | Lint / schema check | Yes (local) |
| Per-PR fast | PR opened or updated | Fast eval subset (~30 cases, < 10 min) | Yes |
| Per-PR class-relevant | PR touches specific area | Class-relevant cases | Yes |
| Nightly on main | Scheduled | Full suite | Trend-only (not blocking) |
| Release candidate | Pre-deploy to staging | Full suite + adversarial subset | Yes |
| Production canary | Post-deploy to a slice | Online judge sampling | Auto-rollback on failure |

Each layer catches a different class of issue at the right speed / cost trade-off.

### 2.2 The fast / full split

The fast subset (per-PR) is the load-bearing layer. It must:

- Run in under 10 minutes (longer than this and developers tab away).
- Cost in proportion (a few dollars per PR run is acceptable; tens of dollars per PR is not).
- Cover the workload's major case classes proportionally.
- Include critical regression cases (per [regression-eval-suites.md](./regression-eval-suites.md)).

The full suite (nightly and release candidate) is the comprehensive measurement:

- Runs the entire golden set + regression suite + specialized suites (adversarial, refusal, etc.).
- May take 30-60+ minutes.
- Costs in proportion ($5-30 per run).
- Produces the canonical quality measurement.

Both are needed. Per-PR catches obvious regressions early; full suite catches the rest before release.

### 2.3 The gate composition

The eval gate may compose multiple checks:

- Golden set pass rate (above threshold).
- Regression suite (no previously-passing cases failing).
- Cost regression check (per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) section 6.1 — estimated per-call cost increase > X% requires override).
- Latency regression check.

Each check has its own threshold; failure of any check fails the gate.

The composition is the engineering work. A gate that only checks pass rate misses cost regressions; a gate that only checks cost misses quality regressions.

---

## 3. Fast subset selection

The fast subset for per-PR runs is itself an engineering decision.

### 3.1 The selection criteria

A good fast subset:

- **Proportional coverage of case classes.** If 40% of the suite is clinical-protocol, ~40% of the subset is clinical-protocol.
- **Critical cases over-represented.** High-stakes / high-severity cases get more weight than their proportion in the suite.
- **Regression cases included.** Critical regression cases are always in the fast subset.
- **Stratified across question types and complexities.**

The subset is small enough to run fast but balanced enough to be representative.

### 3.2 The size

For most workloads: 25-40 cases. Below 20 is statistically noisy; above 50 starts to push the runtime past the budget.

For Meridian Care Coordinator: 30 cases in the fast subset.

### 3.3 The selection process

The fast subset is selected deliberately:

- Annual review: rebalance the subset based on the current suite composition.
- Per-incident adjustment: when a regression case is added to the suite, consider whether it should also be in the fast subset.
- Coverage check: does the subset cover the cases that have historically failed? Add critical-historical cases.

The subset is itself a versioned artifact (per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md)-style discipline); changes are PR-reviewed.

### 3.4 The class-relevant subsets

For PRs that touch a specific area, a class-relevant subset runs additionally:

- PR touches retrieval code → retrieval-relevant cases run (the cases that test retrieval quality).
- PR touches a specific worker's prompt → cases that exercise that worker run.
- PR touches multi-tenant code → tenant-isolation canary cases run.

The class-relevant subsets are smaller (10-20 cases) but more targeted. They run alongside the universal fast subset.

### 3.5 The selection trade-off

Fast subset selection is a coverage-vs-speed trade-off:

- Smaller subset → faster runtime → cheaper per-PR → less coverage.
- Larger subset → slower runtime → more expensive → more coverage.

The sweet spot is workload-specific. Tune over time based on the rate of post-merge regressions caught by the full suite.

---

## 4. Threshold calibration

Setting the right thresholds is harder than running the gate.

### 4.1 The baseline-relative threshold

The primary threshold is *relative to the trailing baseline*:

- Per-PR fast subset: block if pass-rate drops below trailing-7-day baseline minus X percentage points.
- Full suite: block if pass-rate drops below trailing-30-day baseline minus Y percentage points.

For the Meridian clinical fast subset:
- Baseline pass-rate: ~94%.
- Threshold: block if pass-rate < (baseline - 4 points) = < 90%.

The relative threshold accommodates the natural noise of the eval (model nondeterminism, judge variance) while catching real regressions.

### 4.2 The absolute floor

In addition to the relative threshold, an absolute floor:

- Block if pass-rate < absolute floor regardless of baseline.

For Meridian:
- Clinical fast subset: absolute floor = 85%.
- Drug-interaction subset: absolute floor = 95%.

The absolute floor catches situations where the baseline itself has drifted low; the regression is real even if it doesn't fail the relative threshold.

### 4.3 The per-class threshold

For high-stakes classes, a stricter per-class threshold:

- Drug-interaction class: block on any regression on previously-passing cases (zero tolerance).
- Refusal class: block if refusal-when-should drops below 95%.

Per-class thresholds catch class-specific regressions that aggregate metrics hide.

### 4.4 The threshold derivation

Set thresholds from observed baseline:

- After 2-4 weeks of eval runs, observe the natural variance.
- Set the relative threshold at: baseline - (2-3 standard deviations of the variance).
- Set the absolute floor at: baseline - 8-10 points (a meaningful drop).

The thresholds are configured per suite per case-class.

### 4.5 The threshold recalibration

Thresholds need recalibration:

- After a baseline shift (a successful improvement that raises the baseline).
- After a workload change (case classes change distribution).
- After a judge calibration drift event (the judge's verdicts shifted).

Quarterly recalibration is the default cadence.

### 4.6 The flakiness handling

Per [regression-eval-suites.md](./regression-eval-suites.md) section 5.5, flaky cases distort the gate:

- A case that passes 70% of the time produces ~30% false-failure rate.
- Multiple flaky cases stack: probability of a clean run drops.

The discipline: per-case pass rate tracked; cases below 90% are investigated; quarantined or fixed.

---

## 5. The override pattern

Some PRs intentionally regress some cases. The override pattern provides a controlled path.

### 5.1 When override is appropriate

- A planned tightening of refusal behavior (previously the model was too lenient; cases that were passing under the lenient behavior now fail; the team accepts this as the desired direction).
- A planned format change (output structure changes; cases written against the old format fail; the format change is the intended improvement).
- A planned model migration (the new model handles some cases differently; some regressions are accepted in exchange for other improvements).

In each case, the regression is intentional; the team can articulate why; the eval gate should not block.

### 5.2 The override mechanism

The PR includes an override:

- A label or commit-message tag (e.g., `[eval-override: deliberate refusal tightening]`).
- A reviewer approval explicitly acknowledging the trade-off.
- The override reason documented in the PR description.

CI sees the override; gate fails are converted to warnings; PR can merge.

### 5.3 The override audit

Every override is logged:

- Date.
- PR.
- Reviewer who approved.
- Justification.
- Which cases regressed.

The log is reviewed monthly:

- How many overrides this month?
- What justifications?
- Are any patterns concerning?

High override rates suggest the eval is mis-calibrated (cases are too brittle) or the team is bypassing discipline (cases are real but team is rationalizing).

### 5.4 The override scope

Overrides are not blanket. The override applies to *this PR only*; subsequent PRs are subject to the gate normally. If the intentional change should permanently alter the eval baseline:

- Update the affected cases in the eval suite (new expected behavior).
- Or retire the affected cases (they no longer represent intended behavior).

The update / retirement is a follow-up PR after the override-PR merges.

### 5.5 The override-rate metric

Track overrides as a percentage of PRs:

- Healthy: < 5% of PRs use override.
- Concerning: 5-15%.
- Alarm: > 15%.

High rates trigger review of the eval calibration.

---

## 6. Post-merge detection and rollback

The gate is not infallible. Post-merge detection backs it up.

### 6.1 The post-merge eval

After a PR merges:

- The full eval suite runs on the merged main branch (post-merge, scheduled).
- If the full suite identifies regressions the fast subset missed, the team is notified.
- The PR's author / reviewer is paged for investigation.

### 6.2 The auto-rollback option

For high-stakes systems, some teams configure auto-rollback on post-merge regression:

- If the post-merge full suite shows regression > Y points, automatically revert the merge.
- The team is notified; investigation follows; the fix is re-tried with the regression addressed.

Auto-rollback is aggressive; it requires confidence in the eval signal. Not all teams adopt it.

### 6.3 The production canary integration

Per the production canary pattern (per [release-artifacts-for-ai.md](../cicd-and-eval-gates/) coming), a release goes to a small percentage of production traffic first. The online judge SLI on the canary is monitored:

- If the canary's SLI drops, the canary is rolled back automatically.
- Full rollout proceeds only if the canary's SLI matches expectations.

The canary is the production-side complement to the offline eval gate; together they provide layered defense.

### 6.4 The post-merge alerts

When post-merge regressions are detected:

- Tier 1 alert if the regression is severe (per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md)).
- Tier 2 alert otherwise.
- Author and reviewer notified.
- Investigation tracked.

The discipline: post-merge regressions are addressed, not accumulated.

---

## 7. Integration with release artifacts

Per [release-artifacts-for-ai.md](../cicd-and-eval-gates/) (coming), every release pins eval-suite version.

### 7.1 The eval-suite version pin

The release manifest includes:

- `eval_suite.version`: which version of the eval suite validated this release.
- `eval_suite.pass_rates`: per-class pass rates at release-candidate time.

The release knows what eval validated it; rollbacks can reproduce the eval state.

### 7.2 The release-candidate gate

The release-candidate gate is stricter than the per-PR gate:

- Full suite (not subset).
- Adversarial / refusal / security cases included.
- Tighter thresholds (no degradation tolerated on the highest-stakes classes).
- Override discouraged at this stage.

Release candidates that fail the gate cannot promote to production without explicit incident-grade override.

### 7.3 The rollback target

Rollback per the release manifest:

- Each release pins the eval suite version it was validated against.
- Rolling back restores that suite version too (in case the eval suite itself has evolved).
- Post-rollback, the system's expected behavior matches the rollback-target's eval.

---

## 8. Cost considerations

The eval gate's cost is meaningful at scale.

### 8.1 The per-PR cost

Fast subset × judge calls × per-call cost:
- 30 cases × ~3 judge calls × ~$0.012 = ~$1 per PR.
- Active development might be 50-100 PRs per week; ~$50-100 per week in fast-subset costs.

### 8.2 The full-suite cost

200-300 cases × ~3 judge calls × ~$0.012 = $7-10 per full run.

Nightly + release-candidate + on-demand = ~$200-400/month in full-suite costs.

### 8.3 The total eval cost

Cumulative monthly:
- Per-PR fast subsets: ~$200-400.
- Full suite runs: ~$200-400.
- Online judging (per [online-eval-and-feedback.md](./online-eval-and-feedback.md)): ~$300-500.
- Total: ~$700-1300/month.

For Meridian's scale, this is a small fraction of overall AI spend; the eval discipline's value far exceeds its cost.

### 8.4 The cost-budget for the gate

The eval-gate cost has its own budget; if exceeded, the team:

- Investigates (high judge cost may indicate inefficient judge prompts).
- Optimizes (reduce judge calls per case where possible).
- Adjusts sample rates (online judging may sample less aggressively).

### 8.5 The optimization patterns

- **Cache judge verdicts.** When a case's input is identical across runs, the judge verdict can be cached.
- **Parallel judge calls.** Run sub-judges in parallel; suite runtime is bounded by max-parallel rather than sum.
- **Conditional judge.** Skip the expensive judge for cases that pass cheap heuristic checks.

Optimizations are applied as cost pressures arise; not premature.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The gate hierarchy

Meridian's eval-gate configuration:

| Layer | Suite | Runtime | Cost | Blocking |
|---|---|---|---|---|
| Pre-commit | Schema lint | < 30 sec | $0 | Yes |
| Per-PR fast | 30-case stratified subset | ~7 min | ~$1 | Yes |
| Per-PR class-relevant | 10-20 cases per relevant class | additional ~3 min | ~$0.30 | Yes |
| Nightly on main | Full suite (290 cases) | ~25 min | ~$10 | Trend |
| Release candidate | Full suite + adversarial (~320 cases) | ~35 min | ~$13 | Yes |
| Production canary | Online judge on 5% of canary traffic for 4 hours | continuous | ~$3 | Auto-rollback |

The hierarchy catches issues at the appropriate speed / cost trade-off.

### 9.2 The threshold configuration

| Suite | Relative threshold | Absolute floor |
|---|---|---|
| Clinical fast subset | baseline - 4 points | 90% |
| Drug-interaction fast subset | baseline - 2 points (tighter) | 95% |
| Conversational fast subset | baseline - 5 points | 85% |
| Refusal fast subset | baseline - 3 points | 92% |
| Regression suite | zero tolerance (any previously-passing failure) | n/a |

Each threshold is calibrated from observed baseline; recalibrated quarterly.

### 9.3 The override usage

Quarterly review of overrides (2026-Q2):
- 14 PRs used override.
- 6 were prompt-tightening changes (deliberate behavior change; cases updated in follow-up PR).
- 4 were format changes (new output structure; cases updated).
- 2 were model-migration related (acceptable trade-offs in eval).
- 2 were one-off acceptable regressions (acknowledged, accepted, documented).
- Override rate: ~5% of PRs (within healthy range).

### 9.4 The post-merge detection

In 2026-Q2 to date, the post-merge full suite caught 3 regressions that the per-PR fast subset missed:
- All 3 involved case classes the fast subset did not stratify proportionally.
- Each was investigated within 2 hours of the nightly run.
- 1 led to a fast-subset rebalancing (clinical sub-class was under-represented).

Post-merge detection is working; the fast subset is being tuned in response.

### 9.5 The CI / CD integration

The eval gate is integrated with the standard CI/CD:
- GitHub Actions runs the per-PR fast suite as a required check.
- The check status appears on the PR alongside other CI status.
- Failures show the specific failing cases with verdicts and justifications.
- Override-label triggers the override path; reviewer approval visible.

The integration is invisible to most engineers (the gate just works); when it fires, the failure is actionable.

### 9.6 The platform discipline

- Fast subset reviewed annually.
- Thresholds recalibrated quarterly.
- Override audit monthly.
- Post-merge regressions investigated within 24 hours.
- Cost of the gate budgeted as a finops line item.

---

## 10. Anti-patterns

### 10.1 "Eval dashboard but no gate"

The team has eval running but it doesn't block anything. Regressions ship; the dashboard turns red; nobody acts.

**Corrective.** Gate per section 2. The eval is enforcement, not measurement.

### 10.2 "Single threshold, no fast / full split"

The full eval runs on every PR; PR feedback loop is 45 minutes; developers tab away; the gate becomes flaky merely from impatience.

**Corrective.** Fast subset per section 3; full suite at nightly and release-candidate.

### 10.3 "Absolute threshold only, no baseline-relative"

The gate blocks if pass rate < 90%. Baseline is 94%; a regression to 91% passes the gate but is a real problem.

**Corrective.** Baseline-relative threshold per section 4.1; absolute floor as backstop.

### 10.4 "No override mechanism"

The team has no way to ship intentional regressions; PRs that legitimately change behavior are blocked forever.

**Corrective.** Override pattern per section 5; with audit, not just bypass.

### 10.5 "Override is too easy"

Override is a self-attested label; no reviewer approval required. Engineers use it routinely to "ship" through the gate.

**Corrective.** Reviewer approval required per section 5.2; monthly audit reviews usage; high rates trigger calibration review.

### 10.6 "Post-merge detection absent"

The PR merges; nightly catches a regression; nobody investigates. The regression accumulates with the next month's changes.

**Corrective.** Post-merge alerts per section 6.4; investigation tracked.

### 10.7 "Threshold never recalibrated"

The threshold was set at the eval's inception and never updated. The baseline has shifted; the threshold is no longer meaningful.

**Corrective.** Quarterly recalibration per section 4.5.

### 10.8 "Cost of the gate is unbudgeted"

The gate's cost grows quietly; the team learns about it via the monthly invoice.

**Corrective.** Per-feature cost attribution for eval; budget per section 8.4.

---

## 11. Findings (sprint-assignable)

### GATE-001 — Severity: Critical
**Finding.** Eval gate is not enforced; regressions ship to production.
**Recommendation.** Implement the gate per section 2; required PR check.
**Owner.** ai-platform-eng + sre, sprint N+1.

### GATE-002 — Severity: Critical
**Finding.** No fast subset; full suite runs on every PR; developer feedback loop too slow.
**Recommendation.** Define fast subset per section 3; ~10 min runtime.
**Owner.** ai-platform-eng, sprint N+1.

### GATE-003 — Severity: High
**Finding.** Threshold is absolute floor only; baseline-relative regressions are missed.
**Recommendation.** Baseline-relative threshold per section 4.1 + absolute floor as backstop.
**Owner.** ai-platform-eng, sprint N+2.

### GATE-004 — Severity: High
**Finding.** Override mechanism is absent; intentional behavior changes cannot ship.
**Recommendation.** Override per section 5; with reviewer approval and audit.
**Owner.** ai-platform-eng + sre, sprint N+2.

### GATE-005 — Severity: High
**Finding.** Override usage is unaudited; team may be routinely bypassing the gate.
**Recommendation.** Monthly audit per section 5.3; track override rate.
**Owner.** ai-platform-eng team lead, sprint N+2.

### GATE-006 — Severity: High
**Finding.** Post-merge eval detection is absent; regressions the fast subset misses go undetected.
**Recommendation.** Nightly full-suite run on main per section 6.1; investigation alerts.
**Owner.** ai-platform-eng + sre, sprint N+3.

### GATE-007 — Severity: High
**Finding.** Per-class thresholds are uniform; high-stakes classes have the same tolerance as low-stakes.
**Recommendation.** Per-class thresholds per section 4.3.
**Owner.** ai-platform-eng, sprint N+3.

### GATE-008 — Severity: High
**Finding.** Regression suite is not zero-tolerance; previously-passing regression cases can fail without blocking.
**Recommendation.** Zero tolerance per section 4.3 for regression cases.
**Owner.** ai-platform-eng + sre, sprint N+2.

### GATE-009 — Severity: Medium
**Finding.** Cost regression check is not part of the gate; cost-significant prompt or model changes ship without review.
**Recommendation.** Cost regression as a gate check per section 2.3.
**Owner.** ai-platform-eng + finops, sprint N+3.

### GATE-010 — Severity: Medium
**Finding.** Fast subset composition was set once and never reviewed; coverage may have drifted from suite composition.
**Recommendation.** Annual review per section 3.3.
**Owner.** ai-platform-eng team lead, sprint N+3.

### GATE-011 — Severity: Medium
**Finding.** Class-relevant subsets are not configured; PRs touching specific areas run the same fast subset as PRs touching unrelated code.
**Recommendation.** Class-relevant subsets per section 3.4.
**Owner.** ai-platform-eng, sprint N+3.

### GATE-012 — Severity: Medium
**Finding.** Release-candidate gate has the same threshold as per-PR; release-time strictness is missing.
**Recommendation.** Release-candidate gate per section 7.2; tighter thresholds.
**Owner.** ai-platform-eng + sre, sprint N+3.

### GATE-013 — Severity: Medium
**Finding.** Production canary auto-rollback is not configured; bad releases proceed to full rollout.
**Recommendation.** Canary auto-rollback per section 6.3.
**Owner.** ai-platform-eng + sre, sprint N+3.

### GATE-014 — Severity: Medium
**Finding.** Threshold recalibration is not scheduled; thresholds drift.
**Recommendation.** Quarterly recalibration per section 4.5.
**Owner.** ai-platform-eng team lead, sprint N+4.

### GATE-015 — Severity: Medium
**Finding.** Flakiness in cases distorts gate signal; team is learning to re-run failures.
**Recommendation.** Per-case pass rate tracking per [regression-eval-suites.md](./regression-eval-suites.md) section 5.5; quarantine flaky cases.
**Owner.** ai-platform-eng, sprint N+3.

### GATE-016 — Severity: Medium
**Finding.** Eval cost is not budgeted; growth in suite size silently increases cost.
**Recommendation.** Per-feature cost attribution per section 8.4; budget and alerting.
**Owner.** ai-platform-eng + finops, sprint N+4.

### GATE-017 — Severity: Low
**Finding.** Gate failures display verdict only; engineers cannot see judge justification without re-running.
**Recommendation.** CI output includes judge justification per section 9.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### GATE-018 — Severity: Low
**Finding.** Eval-suite versioning is not surfaced in release manifests; rollback may load wrong eval suite.
**Recommendation.** Eval-suite version in release manifest per section 7.1.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team without an eval gate:

- [ ] **Sprint 0 — design.** Decide gate layers (per-PR fast / per-release / etc.). Choose fast-subset size and composition.
- [ ] **Sprint 1 — fast subset.** Build the fast subset. Verify runtime < 10 min.
- [ ] **Sprint 1 — CI integration.** Per-PR fast subset as required check.
- [ ] **Sprint 1 — initial thresholds.** Set initial thresholds (absolute floor + relative); plan recalibration after 2-4 weeks of data.
- [ ] **Sprint 2 — full suite.** Nightly on main; release-candidate gate.
- [ ] **Sprint 2 — override mechanism.** Label / reviewer-approval pattern; documented.
- [ ] **Sprint 3 — post-merge detection.** Nightly catches regressions the fast subset missed; investigation workflow.
- [ ] **Sprint 3 — per-class thresholds.** High-stakes classes get stricter thresholds.
- [ ] **Sprint 4 — cost gate.** Cost regression check added to the gate composition.
- [ ] **Sprint 4 — production canary integration.** Canary auto-rollback (for high-stakes systems).
- [ ] **Sprint 5 — calibration discipline.** Quarterly recalibration scheduled; monthly override audit.
- [ ] **Ongoing — discipline.** Threshold tuning; subset rebalancing; flakiness investigation.

A team that completes this sequence has an eval gate that enforces quality without becoming a productivity tax. A team that ships eval-as-dashboard accumulates regressions that production discovers.

---

## 13. References

- This repo: [eval-engineering/eval-engineering-playbook.md](./eval-engineering-playbook.md) — the broader practice (section 5).
- This repo: [eval-engineering/golden-set-design.md](./golden-set-design.md) — what the gate runs against.
- This repo: [eval-engineering/regression-eval-suites.md](./regression-eval-suites.md) — the zero-tolerance suite.
- This repo: [eval-engineering/llm-as-judge-patterns.md](./llm-as-judge-patterns.md) — the judge that scores.
- This repo: [eval-engineering/online-eval-and-feedback.md](./online-eval-and-feedback.md) — production-side complement.
- This repo: [eval-engineering/eval-of-rag.md](./eval-of-rag.md) — RAG-specific gate considerations.
- This repo: [cicd-and-eval-gates/pipeline-architecture-for-ai.md](../cicd-and-eval-gates/) (coming) — the broader CI/CD context.
- This repo: [cicd-and-eval-gates/release-artifacts-for-ai.md](../cicd-and-eval-gates/) (coming) — release manifest integration.
- This repo: [cicd-and-eval-gates/prompt-version-pinning.md](../cicd-and-eval-gates/) (coming) — prompt-pinning integration.
- This repo: [cicd-and-eval-gates/canary-rollouts.md](../cicd-and-eval-gates/) (coming) — canary patterns.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alerts on post-merge failures.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
