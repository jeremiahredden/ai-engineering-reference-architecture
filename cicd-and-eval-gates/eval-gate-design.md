# Eval Gate Design

> **Audience.** Engineers wiring an eval gate into a CI/CD pipeline. Eval engineers tuning gate thresholds. Tech leads whose eval gate is bypassed because it fires on noise or because it never fires at all. **Scope.** The *engineering* design of an eval gate as the load-bearing CI step for AI changes: gate placement (PR-time fast-eval, release-time full-eval, post-deploy live-eval); threshold-setting discipline; fast-subset selection; full-suite scheduling; integration with the [eval-engineering](../eval-engineering/) folder; override-with-justification pattern. Pair with [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) (the pipeline that surrounds the gate). Cross-link to [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) (upstream eval-side design) and [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) (the suite definitions). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The eval gate is what makes "this change passed CI" mean something for AI features. Without it, the AI portion of the CI/CD pipeline is procedural decoration: a PR opens, a reviewer rubber-stamps, the change merges, and quality regressions reach production unnoticed. With it, regressions get caught at the boundary they were created at.

Two failure modes dominate gate design. The first is the *too-tight* gate: threshold is set close enough to the noise floor that flaky cases trigger failures, engineers learn to bypass the gate, and within a quarter the gate is decoration. The second is the *too-loose* gate: threshold is set so far from the actual regression line that real regressions slip through unnoticed and the team trusts the gate when they shouldn't. Both produce the same observed outcome: production AI incidents that "the gate should have caught." Both are the gate's fault, but for opposite reasons.

The patterns in this document are about designing the gate so it provides real signal — neither bypass-bait nor false confidence. That requires understanding the gate has three placement points (PR-time, release-time, post-deploy-time), each with different signal-to-noise ratios and different acceptable thresholds. It requires understanding the gate's threshold has to be anchored to the empirically-measured flake floor of the eval suite, not to an aspirational quality bar. And it requires the discipline to maintain the gate over time: re-tune thresholds quarterly, archive overrides, audit usage.

The honest framing: the eval gate is the single most-load-bearing piece of AI engineering infrastructure a team builds. It catches regressions code review cannot catch. It enforces quality discipline the team would not enforce manually. It produces the audit trail compliance asks for. A team that builds the gate well will operate AI changes with a fraction of the production-incident rate of a team that does not.

This document is opinionated about four things:

1. **The gate exists at three placement points** — PR-time (fast subset), release-time (full suite), post-deploy (live canary eval). Each has different thresholds. A "single gate" design is wrong.
2. **Thresholds are anchored to the measured flake floor**, not to aspirational quality numbers. A gate that fires on noise gets bypassed; a gate that doesn't fire on real regressions provides false confidence.
3. **Critical-case failures are absolute, not statistical.** A specific named set of critical cases must pass every time. The overall pass-rate metric is a separate gate.
4. **Overrides are allowed, logged, and audited.** A regression that ships is sometimes the right call. The discipline is recording the override, not banning it.

Structure: (2) the three placement points; (3) what the gate actually checks; (4) threshold-setting against the flake floor; (5) fast-subset selection; (6) full-suite scheduling; (7) integration with eval engineering; (8) the override pattern; (9) gate maintenance over time; (10) worked Meridian example; (11) anti-patterns; (12) findings; (13) adoption checklist; (14) references.

---

## 2. The three placement points

The "eval gate" is not one gate. It is three gates at three points in the pipeline, each with different signal characteristics.

### 2.1 PR-time gate (fast eval)

**Triggered by:** every PR that touches AI artifacts (prompts, models, corpora, eval suites).

**Signal:** a 50–200-case subset of the full eval suite, run in < 10 minutes.

**Purpose:** catch obvious regressions before review starts. Don't let a reviewer waste time on a PR whose prompt fundamentally breaks the feature.

**Threshold:** above the flake floor, permissive enough that small variance doesn't trip it. Block on critical-case failure and safety regression; allow modest pass-rate drift on non-critical cases.

**Action on failure:** the PR check fails. The engineer fixes the issue or marks the regression as intentional with justification.

### 2.2 Release-time gate (full eval)

**Triggered by:** every release-candidate build (post-merge to main, on a schedule for nightly verification).

**Signal:** the complete eval suite (thousands of cases), run over hours.

**Purpose:** confirm the candidate is shippable. This is the production gate.

**Threshold:** tighter than PR-time. Pass-rate must be within tight tolerance of baseline; no critical or safety failures; cross-feature regressions checked; cost and latency profiles within tolerance.

**Action on failure:** the release candidate is blocked. Investigation precedes promotion.

### 2.3 Post-deploy gate (live canary eval)

**Triggered by:** canary deployment of a new version (1% / 10% / 50% traffic levels).

**Signal:** live-judge against canary production traffic plus user behavior metrics.

**Purpose:** catch what offline eval missed because production traffic differs from the eval distribution.

**Threshold:** more permissive than offline eval (smaller sample, higher inherent noise) but tight on safety. Quality drift caught against pre-canary baseline.

**Action on failure:** canary is rolled back; the release is held until investigated.

### 2.4 The interaction

The three gates have to be designed together. If PR-time is tight enough to catch everything, it will fire on noise and get bypassed. If release-time is too loose, regressions slip past. If post-deploy is the only real check, the team learns about regressions in production rather than before.

The discipline is to *cascade* the gates:

- PR-time: catches *obvious* regressions cheaply.
- Release-time: catches *subtle* regressions thoroughly.
- Post-deploy: catches *production-mix-specific* regressions in the smallest blast radius.

Each gate has its own threshold and its own purpose. None replaces the others.

---

## 3. What the gate actually checks

The gate is not one number. It is a set of conditions, all of which must hold.

### 3.1 Pass rate (statistical)

The overall pass-rate on the eval suite. Reported as percentage of cases passing the judge's rubric.

- Baseline: the pass rate on the prior-shipped version, on the same eval suite.
- Delta: candidate pass rate minus baseline.
- Gate criterion: delta ≥ -N percentage points (N tuned per §4).

Pass rate is the *headline* metric but it is noisy. It is one of several gate checks, not the only one.

### 3.2 Critical-case pass rate (absolute)

A specific named set of cases that must pass on every release. Typically 10–50 cases representing the most-load-bearing scenarios.

- Baseline: 100% pass on the named set.
- Gate criterion: 100% pass; no exceptions.

The critical set is curated. It is updated when a new class of regression is discovered (the regression's test case is added to the critical set).

For Care Coordinator's critical set: drug-allergy interaction detection, dose-range validation, escalation-trigger recognition, refusal on diagnostic-rather-than-care-coordination queries. These cases are the system's load-bearing safety properties; any regression on them is unacceptable regardless of overall pass-rate.

### 3.3 Safety-case pass rate (absolute)

Separate from critical: cases that test refusal, harm prevention, sensitive-content handling.

- Baseline: 100% pass.
- Gate criterion: 100% pass.

The safety set lives alongside the critical set but is owned by the safety / trust team rather than the product team.

### 3.4 Cost per case (regression)

Mean cost per case in the suite.

- Baseline: cost per case on prior version.
- Delta: candidate minus baseline.
- Gate criterion: delta ≤ +10% (tunable).

Distinct gate to catch quality-positive but cost-explosive changes.

### 3.5 Latency per case (regression)

Mean p50 / p95 latency per case.

- Baseline: latency on prior version.
- Delta: candidate minus baseline.
- Gate criterion: p95 delta ≤ +15%; p50 delta ≤ +10%.

Same logic as cost: catches slowness that quality wouldn't reveal.

### 3.6 Cross-feature regression (release-time only)

Does changing the prompt for feature A break the output format expected by feature B downstream?

- Run cross-feature integration cases.
- Gate criterion: no new failures on cross-feature suite.

Cross-feature is expensive to evaluate, so it lives in the full eval (release-time), not the fast eval (PR-time).

### 3.7 Composite gate logic

The gate passes only if all checks pass:

```python
def evaluate_gate(eval_results, baseline, thresholds):
    if eval_results.critical_pass_rate < 1.0:
        return Block(reason="critical case failure")
    if eval_results.safety_pass_rate < 1.0:
        return Block(reason="safety regression")
    if eval_results.pass_rate < baseline.pass_rate - thresholds.pass_rate_delta:
        return Block(reason="pass-rate regression")
    if eval_results.cost_per_case > baseline.cost_per_case * (1 + thresholds.cost_delta):
        return Block(reason="cost regression")
    if eval_results.p95_latency > baseline.p95_latency * (1 + thresholds.latency_delta):
        return Block(reason="latency regression")
    if eval_results.cross_feature_failures > 0:  # release-time only
        return Block(reason="cross-feature regression")
    return Pass()
```

The block reason is surfaced to the engineer. Each block carries the data needed to investigate.

---

## 4. Threshold-setting against the flake floor

This is the single most-mismanaged aspect of eval gate design. Most teams pick thresholds by intuition; the result is either constant false positives or constant false negatives.

### 4.1 The flake floor

The flake floor is the apparent variation in eval results when nothing has actually changed. Two runs of the *same* eval suite against the *same* model and *same* prompt will produce slightly different results because:

- Sampling at temperature > 0 produces different outputs.
- Judge variance: the live-judge has its own noise.
- Tokenization or runtime non-determinism may shift outputs.

Measure the flake floor empirically:

- Lock the model, prompt, and eval suite.
- Run the eval twice with no changes.
- Compute the difference in pass rate, mean cost, mean latency, etc.
- That difference is the flake floor for each metric.

For a typical 1000-case eval suite, the flake floor on pass rate is often 0.3–1.0 percentage points. On cost per case, it might be 1–3%. On latency, 5–10%.

### 4.2 Setting thresholds from the floor

The gate threshold must be > 2× the flake floor (otherwise the gate fires on noise).

- Pass-rate flake floor 0.4pp → gate threshold ≥ 0.8pp. (Practical: 1.0pp for PR-time fast eval; 0.5pp for release-time full eval where the larger sample reduces noise.)
- Cost flake floor 2% → gate threshold ≥ 4%. (Practical: 10% for PR-time; 5% for release-time.)
- Latency flake floor 8% → gate threshold ≥ 16%. (Practical: 15% for p95 release-time.)

The flake floor is *per eval suite, per model*. Re-measure when either changes substantially.

### 4.3 The trap of "stricter is better"

Engineers' instinct is to set thresholds tightly. The trap: a threshold below the flake floor produces false positives at a rate proportional to (flake floor / threshold). At 1× the floor, 50% of clean PRs fail. At 0.5× the floor, 80% fail. Engineers learn to override; the gate becomes meaningless.

Discipline: threshold ≥ 2× the floor. Always.

### 4.4 The trap of "we'll just rerun"

Some teams accept the false-positive rate and rerun the eval on every flake. This works for low-PR-volume teams but breaks at scale: the eval is expensive (full eval can cost hundreds of dollars), and the rerun strategy doubles the cost without improving signal.

Better: set the threshold to the actual signal-to-noise ratio of the eval, and trust the gate's verdict.

### 4.5 Re-measuring the floor

The floor moves when:

- The judge model changes.
- The judge prompt changes.
- The model under eval changes (a different model has different output variance).
- The eval suite changes (different cases have different inherent variance).
- The runtime changes (a different inference stack has different non-determinism).

Re-measure the floor quarterly, and on any of the above changes. Document the floor in the eval-gate config:

```yaml
eval_gate_config:
  fast_eval:
    flake_floor:
      pass_rate_pp: 0.4
      measured_at: 2026-05-01
      measured_against: claude-opus-4-7@2026-04-12
    thresholds:
      pass_rate_pp: 1.0
      cost_delta_pct: 10
      latency_p95_delta_pct: 20

  full_eval:
    flake_floor:
      pass_rate_pp: 0.2
      measured_at: 2026-05-01
      measured_against: claude-opus-4-7@2026-04-12
    thresholds:
      pass_rate_pp: 0.5
      cost_delta_pct: 5
      latency_p95_delta_pct: 15
      cross_feature_failures: 0
```

---

## 5. Fast-subset selection

The fast eval is a curated subset of the full eval. The selection matters.

### 5.1 Selection goals

The fast subset should:

- Run in < 10 minutes.
- Cover the most-load-bearing cases (regressions here = production incidents).
- Cover all critical and safety cases (no regressions allowed).
- Be representative of the traffic distribution (proportional to query types).
- Be stable across releases (so trend comparisons are meaningful).

### 5.2 The composition

A typical fast subset:

- **Critical cases:** 10–30 cases. All of them. Always.
- **Safety canaries:** 10–20 cases. All of them.
- **Stratified sample:** 50–150 cases sampled across query types, weighted by traffic share.
- **Recent-regression cases:** 5–20 cases drawn from cases that regressed in the last 90 days.

Total: 75–220 cases, runnable in < 10 minutes against a frontier model.

### 5.3 Selection discipline

The fast subset is *curated*, not randomly resampled per run. A stable subset enables:

- Trend comparison ("the pass rate on this subset has been 99.1% for three releases").
- Anchoring against a known baseline.
- Reproducible reasoning ("this PR regressed cases A, B, C").

Periodically (quarterly or on major eval-suite refresh), the subset is reviewed:

- Cases that have not failed in 6+ months may be candidates for rotation off.
- Cases representing newly-discovered failure modes get added.
- The total size stays in the 75–220 range.

### 5.4 The cost of the fast eval

Per-run cost depends on suite size and model:

- 150-case fast eval × ~5K tokens per case × Opus pricing = ~$15 per run.
- Multiplied by 50 PRs/week = $750/week = $39K/year for one team.

Bounded but real. Budget it.

### 5.5 Scoping who runs the fast eval

Not every PR touches AI artifacts. Application-only PRs don't need the AI eval. Condition the fast eval on PR scope:

- PR touches `prompts/`, `models/`, `corpus/`, or `eval/`: fast eval runs.
- PR touches only application code: fast eval skipped; standard code tests run.

The CI logic checks the paths in the PR diff and conditions the job accordingly.

---

## 6. Full-suite scheduling

The full eval is too expensive to run on every PR. The scheduling pattern:

### 6.1 On every release candidate

- Triggered automatically post-merge when a release-candidate build is created.
- Blocks promotion of the release candidate.
- Result archived in the release artifact.

### 6.2 Nightly on main

- Confirms main is in a shippable state.
- Catches drift from PR-time evals: aggregate of small approved regressions, environmental shifts, judge drift.
- Result archived; flagged to the team if the full-eval gate fails on main.

### 6.3 Weekly on extended suite

- An extended eval (10K+ cases including long-tail) runs weekly.
- Catches regressions on rare cases that the standard eval suite doesn't cover.

### 6.4 On schedule changes

When the eval suite, the judge, or the rubric changes, all three schedules re-run their baselines against the new configuration. Gate thresholds are re-anchored to the new baselines.

### 6.5 Full-eval cost budgeting

- 10K-case full eval × 5K tokens/case × Opus pricing = ~$1000 per run.
- Nightly + per-release-candidate × 4 weeks = ~$60K/quarter.
- Worth the spend if it prevents one production incident. Production incidents in AI systems typically cost 10–100× this in remediation, customer trust, and engineering time.

---

## 7. Integration with eval engineering

The eval gate is the *consumer* of the eval engineering work. The integration:

### 7.1 The judge is upstream

The eval gate uses the judge defined in [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md). The judge's prompt, rubric, and model are owned by the eval engineering team. The gate does not modify them; the gate consumes them.

### 7.2 The suite is upstream

The eval gate runs the suite defined in [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md). The suite composition is the eval team's call; the gate's job is to run it consistently and read the result.

### 7.3 The baseline is jointly owned

The baseline pass-rate, baseline cost, baseline latency are the eval team's measurements. The gate logic (threshold ≥ baseline - X) is the platform team's responsibility. Co-ordination is needed when either changes:

- Eval team refreshes the suite → platform team re-measures the flake floor and re-tunes thresholds.
- Platform team adopts a new pipeline tool → eval team verifies the new tool produces the same baseline.

### 7.4 The override is a shared decision

An eval team-flagged regression can be overridden by the platform team only with the eval team's approval. The eval team owns the *fact* of the regression; the platform team owns the *decision* to ship despite it.

### 7.5 Live-eval integration for post-deploy

The post-deploy gate uses [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md). The same judge, the same rubric, the same scoring — but applied to production traffic during canary rather than to the offline suite.

---

## 8. The override pattern

Sometimes a regression should ship. The discipline is making the override visible and accountable.

### 8.1 When to override

Legitimate reasons:

- **Intentional regression on edge cases.** A new safety policy refuses queries the previous version answered; refusal-rate regresses by 2pp. The team decided this is correct.
- **Cost-positive quality trade.** A cheaper model is being adopted; quality drops 0.3pp but cost drops 30%. The team decided the trade is acceptable.
- **Flaky case under investigation.** A specific case has been flaky for two weeks; investigation is open. Override allows shipping while the investigation continues.
- **Emergency safety patch.** A safety incident requires a patch that has not been fully evaled; the override allows the patch while the full eval runs in the background.

### 8.2 The override mechanics

The PR or release includes an `eval-override.yaml`:

```yaml
override:
  ticket: AI-1247
  approver: tech-lead-name
  reason: "Intentional refusal-rate increase per new safety policy"
  expected_regressions:
    - metric: pass_rate_pp
      cases: [refusal-edge-001, refusal-edge-002, refusal-edge-014]
      expected_delta: -2.0pp
      until: 2026-07-01  # re-eval after refusal-policy stabilizes
```

The gate reads the override. Regressions in the listed cases / metrics are allowed up to the specified delta. Regressions outside the listed scope still block.

### 8.3 The approval discipline

Override requires sign-off:

- Fast-eval override: tech lead.
- Full-eval override: tech lead + product owner; if safety regression, safety team lead.
- Cost / latency override: tech lead + SRE; if material, finance / FinOps.
- Cross-feature regression override: tech lead + the affected downstream feature's owner.

Sign-off is in the override file, logged in the release artifact.

### 8.4 The audit cadence

Quarterly: the AI Platform team reviews overrides used in the prior quarter.

- Are any patterns of override suggesting a gate is mis-tuned?
- Have any "until" dates expired without follow-up?
- Are overrides clustering on specific features / engineers / projects?

The audit informs gate re-tuning.

### 8.5 What overrides do not authorize

- Skipping the eval entirely.
- Marking a regression as "expected" without specifying which cases.
- Permanent overrides (every override has an expiration).
- Override of safety regressions without safety-team approval.

---

## 9. Gate maintenance over time

The gate is a living system. Maintain it.

### 9.1 The maintenance cadence

**Quarterly:**

- Re-measure the flake floor for both fast and full eval.
- Re-tune thresholds against the updated floor.
- Audit overrides used in the prior quarter.
- Review critical-case set: any cases to add, any to retire?
- Review fast-subset composition.

**On major changes (judge, rubric, model, runtime):**

- Re-measure floor immediately.
- Re-establish baselines.
- Communicate changes to engineers.

**On every production incident:**

- Add a test case representing the incident to the critical or relevant suite.
- If the incident slipped past the gate, investigate why; the gate may need adjustment.

### 9.2 The metrics on the gate

The team tracks gate health:

- Pass-fail rate per stage. (PR fast-eval pass rate; release full-eval pass rate.)
- False-positive rate. (Gate failed but no real regression; estimated via re-run.)
- Override rate. (Gate failed; override used.)
- Cost of gate runs. (Quarterly.)
- Time spent in gate. (Engineer-hours waiting for fast eval, hours blocked on full eval.)

These metrics are presented to engineering leadership quarterly. A gate with a 30% override rate is broken; a gate with a 1% false-positive rate is well-tuned.

### 9.3 Roadmap for the gate

The gate is not "done" at first ship. Common follow-up investments:

- **Differential evaluation:** only re-eval cases whose inputs might be affected by the change. Reduces fast-eval cost dramatically for prompt-only changes.
- **Adaptive thresholds:** thresholds tuned per-feature based on per-feature noise floor.
- **Tenant-aware gate:** per-tenant eval subsets for tenants with divergent behavior.
- **Eval-on-real-traffic:** the post-deploy gate becomes more sophisticated; canary criteria become a richer signal than offline eval.

---

## 10. Worked Meridian example: gate that catches a prompt regression

The Care Coordinator team is shipping a prompt change that refactors the system prompt for clarity. The engineer believes the refactor is semantically equivalent but more readable.

### 10.1 PR-time

- PR opens with the prompt change.
- Lint passes.
- Fast eval runs on 150 stratified cases including 18 critical and 12 safety canaries.
  - Critical cases: 17/18 pass. **One critical case fails.**
  - Specifically: a "drug-allergy interaction detection" case. The candidate output omits the warning that the baseline included.
  - Safety: 12/12 pass.
  - Pass rate overall: 145/150 = 96.7% vs baseline 98.7%. Delta -2.0pp.
  - Cost: within tolerance.
  - Latency: within tolerance.
- Gate: BLOCK (critical-case failure + pass-rate regression).
- PR check fails with detailed report. Engineer reads the failing case's diff.

### 10.2 Investigation

The engineer reviews the output diff: the refactored prompt removed the section that emphasized drug-allergy warnings. The refactor was *not* semantically equivalent — it dropped a load-bearing instruction.

The engineer adds the missing instruction back. Pushes the fix.

### 10.3 Re-run

- Fast eval runs again.
- Critical: 18/18. Safety: 12/12. Overall: 148/150 = 98.7% — matches baseline.
- Gate: PASS.

### 10.4 Release-candidate

- Merge to main. Release candidate built.
- Full eval runs: 1247 cases.
- All gates pass with margin.
- Cost +0.3%, latency p95 -1.2%, pass rate 99.43% vs baseline 99.42%.
- Gate: PASS.

### 10.5 Canary

- Deploys at 1%; live-judge on canary traffic.
- 4-hour window, ~1200 canary conversations.
- Live-judge quality: 7.41 vs baseline 7.42. Inside noise.
- Refusal rate: unchanged. Latency: unchanged.
- Canary gate: PASS. Auto-ramp.

### 10.6 Promote

Full promotion completes. Release artifact archived with all eval results.

### 10.7 The counterfactual

Without the gate:

- The prompt change would have shipped.
- Drug-allergy warnings would have been silently weaker for ~24–48 hours.
- A patient near-miss is plausible.
- The team learns about the regression from a clinical-team incident report.

With the gate:

- The PR fails at minute 8.
- The engineer fixes the issue in 20 minutes.
- The release proceeds normally.

The gate caught a regression that code review missed. That is the gate's whole purpose.

### 10.8 Findings closed

- **ARCH-CARE-065** (no PR-time eval for prompt changes; refactors shipped unverified).
- **ARCH-CARE-066** (no critical-case set; load-bearing cases not pinned).
- **ARCH-CARE-067** (gate thresholds not anchored to flake floor; previous attempts at gating fired on noise and were bypassed).
- **ARCH-CARE-068** (override usage untracked; previous overrides invisible to leadership).

---

## 11. Anti-patterns

### 11.1 The gate that fires on noise

Threshold set below the flake floor. Pass rate drops by 0.2pp on a noisy run; gate fires; engineer overrides. Within a quarter, override is the default; gate is decoration.

The fix: measure the floor; threshold ≥ 2× floor.

### 11.2 The gate that never fires

Threshold set so loose that any regression less than catastrophic passes. Team trusts the green check; regressions reach production.

The fix: threshold near the historical pass-rate floor minus the flake floor. A green check should mean "no regression," not "the system still half-works."

### 11.3 The "one gate to rule them all"

Single gate at one placement point. Either it's tight enough that it bypasses too often (PR-time) or it catches issues too late (release-only) or it only fires in production (post-deploy only).

The fix: three gates at three placement points, each with its own threshold.

### 11.4 The forever-override

Override marked "until further notice" with no expiration. The override stays in place permanently; the regression is forgotten; baseline gradually shifts to include it.

The fix: every override has an expiration. The audit catches expirations that have lapsed.

### 11.5 The unowned flake floor

Nobody measures the flake floor. Thresholds are set by intuition; they drift over time as the suite, model, and judge evolve. Gate behavior becomes opaque.

The fix: quarterly flake-floor measurement; documented in eval-gate-config; visible to engineers.

### 11.6 The mixed-purpose suite

The same suite is used as the fast eval, full eval, and post-deploy eval. The signal characteristics are wrong for at least one of the three.

The fix: distinct fast-subset (curated 75–220 cases), full suite (thousands), and post-deploy canary criteria (production telemetry + live-judge).

### 11.7 The cost-blind gate

Gate checks quality only. A quality-positive but cost-explosive change passes. Finance flags the cost a month later; team can't decompose the cause.

The fix: cost gate. Separate criterion, distinct threshold.

### 11.8 The unowned critical set

Critical cases were defined once, never reviewed. New incident classes are not added; retired features' cases remain. The set drifts away from the current load-bearing scenarios.

The fix: quarterly review of the critical set; ownership assigned (product + eval team).

---

## 12. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-EG-001 | Gate exists at only one placement point | High | Implement all three: PR-time, release-time, post-deploy per §2 | AI Platform + Eval Eng |
| CICD-EG-002 | Flake floor not measured; thresholds set by intuition | High | Measure floor quarterly; document in eval-gate-config; threshold ≥ 2× floor | Eval Eng |
| CICD-EG-003 | Pass-rate is the only gate metric; cost / latency not checked | High | Add cost and latency gates per §3.4 / §3.5 | AI Platform + FinOps + SRE |
| CICD-EG-004 | No critical-case set; load-bearing cases not pinned | High | Curate critical set per §3.2; require 100% pass | Product + Eval Eng |
| CICD-EG-005 | No safety-case set; safety regressions undetected | High | Curate safety set per §3.3; require 100% pass | Safety + Eval Eng |
| CICD-EG-006 | Fast subset randomly resampled per run; trend comparison impossible | Medium | Stable curated fast subset per §5.3 | Eval Eng |
| CICD-EG-007 | Full eval runs only on release; main can drift unnoticed | Medium | Nightly full eval on main per §6.2 | AI Platform |
| CICD-EG-008 | Override mechanism absent; gate failure is binary (pass or hidden) | High | Implement override file + approval per §8.2 / §8.3 | AI Platform |
| CICD-EG-009 | Overrides not audited; pattern of overrides invisible | Medium | Quarterly override audit per §8.4 | AI Platform + Eval Eng |
| CICD-EG-010 | Override "until" dates not enforced; permanent overrides accumulate | Medium | Enforce expiration in gate logic; lapsed overrides re-block | AI Platform |
| CICD-EG-011 | Cross-feature regression not checked | High | Cross-feature cases in full eval; gate per §3.6 | Eval Eng |
| CICD-EG-012 | Gate health metrics not tracked; false-positive rate unknown | Medium | Track pass-fail, false-positive, override rate per §9.2 | AI Platform |
| CICD-EG-013 | Gate doesn't re-anchor on judge / rubric change | Medium | Trigger floor re-measurement on judge / rubric / model change | Eval Eng |
| CICD-EG-014 | Cost of running eval gate not budgeted; surprise finance hits | Low | Budget eval cost quarterly; size suite accordingly | FinOps + Eval Eng |
| CICD-EG-015 | Production incidents not converted to gate test cases | High | Every production incident produces a test case added to relevant suite | Product + Eval Eng |
| CICD-EG-016 | Post-deploy gate uses same criteria as offline gate; production-mix issues missed | High | Post-deploy criteria from production telemetry per §2.3 | AI Platform + Eval Eng |
| CICD-EG-017 | Fast eval skipped on PRs that should run it (path-conditioning broken) | Medium | Verify PR-scope detection covers prompts / models / corpora / evals | AI Platform |
| CICD-EG-018 | Per-tenant divergence not gated; tenant-specific regressions ship | Medium | Per-tenant eval subsets per §5; gate on per-tenant pass rate where divergence exists | AI Platform + Eval Eng |

---

## 13. Adoption checklist

- [ ] Gate placement at all three points: PR-time fast eval, release-time full eval, post-deploy canary eval.
- [ ] Flake floor measured for both fast and full eval; documented in eval-gate-config.
- [ ] All thresholds ≥ 2× the measured flake floor.
- [ ] Critical-case set curated; gate requires 100% pass.
- [ ] Safety-case set curated; gate requires 100% pass.
- [ ] Cost-regression gate active.
- [ ] Latency-regression gate active.
- [ ] Cross-feature regression check in full eval.
- [ ] Fast subset stable across releases; reviewed quarterly.
- [ ] Full eval scheduled on every release candidate + nightly on main + weekly extended.
- [ ] Override mechanism with documented justification, approval roles, and expiration.
- [ ] Overrides logged in release artifact; audited quarterly.
- [ ] Gate health metrics tracked (pass-fail rate, false-positive, override rate, cost).
- [ ] Production incidents converted to test cases within the next sprint.
- [ ] Post-deploy gate uses production telemetry, not offline metrics.
- [ ] Per-tenant eval subsets where tenant behavior diverges.

---

## 14. References

**Internal:**

- [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) — the pipeline this gate lives inside.
- [prompt-version-pinning.md](./prompt-version-pinning.md) — the artifact discipline the gate verifies.
- [model-version-pinning.md](./model-version-pinning.md) — the model-version discipline the gate verifies.
- [release-artifacts-for-ai.md](./) — the artifact format the gate result lives in (coming).
- [canary-rollouts.md](./) — the post-deploy mechanism that hosts the third gate (coming).
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — upstream eval-side design.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — the suite definitions.
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — the case curation discipline.
- [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) — the judge the gate consumes.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — the live-judge for post-deploy.
- [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — RAG-specific eval shape.
- [model-lifecycle/model-promotion.md](../model-lifecycle/model-promotion.md) — the model promotion workflow this gate sits in.
- [model-lifecycle/ab-model-testing.md](../model-lifecycle/ab-model-testing.md) — A/B as a comparison after the gate is past.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — the cost signal source.

**Cross-repo (architecture sibling):**

- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
