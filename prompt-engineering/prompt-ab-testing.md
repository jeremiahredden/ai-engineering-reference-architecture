# Prompt A/B Testing

> **Audience.** Engineers and tech leads running prompt experiments in production. Anyone whose pre-prod eval said "the new prompt is better" but production behaviour seems uncertain. Tech leads who want to prevent A/B tests from accumulating into a permanent forking jungle. **Scope.** The engineering of prompt experiments — traffic splitting, statistical-significance design for noisy quality metrics, integration with online evals, interaction with rollback, and the discipline that prevents experiment sprawl. Not the offline eval (see [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md)). Not the broader experimentation platform (see [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Pre-prod eval tells you whether a prompt change *probably* improves quality on the golden set. Production tells you what *actually* happens with real users on real inputs at real scale. The gap between the two can be substantial: a prompt that scores 4.3/5 on the golden set may score 4.1/5 on production traffic, or 4.5/5; the golden set is a proxy that doesn't perfectly track production reality.

A/B testing is the discipline that closes the gap. Two prompt versions run in parallel on a traffic split; observed outcomes are compared; the winner is promoted; the loser is retired. The team gets data on the production effect of the change before committing to it everywhere.

Done well, A/B testing turns prompt engineering from "we think this is better" to "we measured this is better." Done badly, it becomes a permanent fork: the experiment never concludes, traffic is split across N variants indefinitely, the system carries the operational complexity of many parallel paths without resolving any of them.

The discipline this document covers is the engineering practice: when to A/B test, how to design the experiment, how to read the data, when to stop, how to roll out the winner cleanly, and the disciplines that prevent experiment sprawl. The mechanics are mature (most experimentation platforms support the basics); the prompt-specific concerns are statistical power on noisy quality metrics, online eval integration, and the discipline of *closing* experiments rather than letting them fade.

This document is opinionated about four things:

1. **A/B testing is for production-relevant questions, not all prompt changes.** Most prompt changes go through pre-prod eval and ship if the eval clears. Reserve A/B testing for changes whose production effect is genuinely uncertain.
2. **Quality metrics are noisy; design for the noise.** Power analysis matters; small experiments on small samples produce misleading conclusions; the practice has to honour statistical reality.
3. **Online eval is the measurement instrument.** Pre-prod eval informs pre-launch decisions; online eval (continuous LLM-judge on production samples) is what the A/B compares. Without online eval, A/B testing has no instrument.
4. **Experiments have lifecycles. End them.** Start date, success criteria, planned end date, post-experiment review. Experiments that drift indefinitely are operational debt.

Structure: (2) when to A/B test; (3) experiment design; (4) statistical considerations for quality metrics; (5) online eval as the measurement instrument; (6) traffic splitting mechanics; (7) interaction with rollback and risk management; (8) the experiment lifecycle; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. When to A/B test

The decision is per-change, not per-team policy.

### 2.1 Cases that warrant A/B testing

- **A change with genuine production uncertainty.** Pre-prod eval was inconclusive (similar quality scores; high variance). Need production data.
- **A change that affects subjective qualities.** "Sounds more natural"; "feels less robotic" — these are hard to measure on golden sets but visible in production engagement metrics.
- **A change with cross-cutting impact.** A platform-component update affecting many features; A/B test on a sample of features verifies the impact.
- **A change with significant cost or latency implications.** Promotion based on quality but cost or latency could regress; A/B tests both dimensions.
- **A change in a high-stakes feature where regression is unacceptable.** Even if pre-prod eval is positive, A/B tests catches surprises.

### 2.2 Cases that don't warrant A/B testing

- **Small, low-risk changes that pass pre-prod eval.** Most prompt changes. Pre-prod eval clears them; ship.
- **Changes where the right answer is unambiguous.** A bug fix; a typo correction; a clear improvement.
- **Changes too small to power an experiment.** If the expected effect size is small and the call volume is low, the experiment will be underpowered; the result won't conclude either way.
- **Changes where production observation is not feasible.** No online eval; no engagement metric; nothing to measure.

### 2.3 The A/B test budget

A/B testing has cost:

- **Experiment infrastructure** (the traffic-split mechanism, the result-aggregation).
- **Engineering time** to design, monitor, conclude.
- **Risk exposure** to the variant being tested (some users get the variant; if it's bad, those users are affected).
- **Continuous-eval costs** (LLM judge on production samples for both variants).

A team running too many A/B tests in parallel has too much overhead. Cap: typically 3–5 active A/B tests per feature at any time.

### 2.4 The "promote with confidence" decision tree

Pre-prod eval clear, no genuine uncertainty → promote without A/B.

Pre-prod eval clear, but production uncertainty (sec 2.1) → A/B test.

Pre-prod eval ambiguous → either improve the pre-prod eval or run an A/B with appropriate sizing.

Pre-prod eval shows regression → don't promote; rework first.

### 2.5 The "feature flag" alternative

Some changes don't need an A/B test — they need a feature flag for gradual rollout (1% → 10% → 50% → 100% with monitoring at each step). The team observes for regressions without doing a comparative experiment.

Feature flags catch obvious regressions; A/B tests measure subtler differences. Use the right tool for the question.

---

## 3. Experiment design

A well-designed experiment is half the value.

### 3.1 The pre-experiment specification

Before launching:

- **Hypothesis.** What change is being made; what effect is expected.
- **Success criteria.** Specific, measurable: "Variant B's online-judge score is ≥ Variant A's by ≥ X% at p < 0.05."
- **Failure criteria.** What constitutes a clear loss: "Variant B's online-judge score is < Variant A's by ≥ Y%."
- **Inconclusive criteria.** "Variant B and A are within Z% at p > 0.05 after N samples."
- **Traffic split.** What % of traffic to each variant.
- **Duration.** Planned end date; can extend if inconclusive but commit to a stop.
- **Slice analyses.** Pre-specified slices (case type, persona, tenant tier) for analysis; prevents post-hoc cherry-picking.
- **Monitoring plan.** What alerts; who responds; how often the team checks.
- **Rollback plan.** What conditions trigger rollback; who has authority.

The specification is the experiment's charter. Without it, the experiment drifts and conclusions become subjective.

### 3.2 The success metric

Most prompt A/B tests use online quality eval (per [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md)) as the primary metric:

- LLM judge scores production samples from each variant.
- Aggregate (mean, ratio) per variant.
- Comparison between variants.

Secondary metrics to monitor:

- Cost per request.
- Latency p99.
- Escalation rate.
- User engagement (where available — click-through, follow-up rate, explicit feedback).

A primary metric guards against multi-metric optimisation tricks (the team picks the metric that favours their preferred variant after seeing data).

### 3.3 The sample size

Determined by power analysis. Rough rule:

- For an LLM-judge-on-5-point-scale metric with typical variance, detecting a 5% relative improvement at p < 0.05 with 80% power requires ~2,000–5,000 samples per variant.
- For larger expected effects (10%+), smaller samples suffice.
- For smaller expected effects (1–2%), much larger samples.

Underpowered experiments waste time. If the math says 5,000 samples and the team has 100/day, the experiment runs 50 days; consider whether that's worth it vs improving pre-prod eval.

### 3.4 The traffic split

Common splits:

- **50/50.** Equal power on both sides; fastest conclusion.
- **90/10.** Slow conclusion; reduced risk exposure to the variant (for high-stakes changes).
- **Holdout.** Variant is 0% (all traffic on control); used to measure the existing variant's stability over time as a baseline.

Most production A/B tests start at 50/50 (or 90/10 for cautious cases) and may rebalance as data accumulates.

### 3.5 Per-user vs per-request

Decision: should the same user always see the same variant, or can each request get a different variant?

- **Per-user (sticky).** User experience is consistent. Required when variants have different conversational patterns (the user shouldn't switch mid-conversation).
- **Per-request.** Larger effective sample size faster. Used for stateless features.

Most agent features are per-user. Most workflow features can be per-request.

### 3.6 The control variant

The control is the current production prompt. Changes accumulating during the experiment should not be made to the control (else the experiment is comparing two moving targets). Pause non-critical changes to the control while experiments run.

If a critical fix to the control is needed mid-experiment, the experiment is typically restarted from the fix.

### 3.7 The eval-set pre-check

Before launching the A/B, the variant should pass the standard pre-prod eval. The A/B is in addition to, not instead of, the pre-prod eval gate.

If the variant fails pre-prod eval, fix the variant first; don't run an A/B that's likely to confirm a bad prompt.

---

## 4. Statistical considerations for quality metrics

Quality metrics are noisier than business metrics; statistical care matters.

### 4.1 The variance challenge

Online LLM-judge scores are noisy:

- The judge's evaluation has variance (the same case may score 4.0 on one judge run, 4.2 on another).
- Different cases have different "natural" scores (some are easy, some hard).
- The mix of cases in a sample varies day-to-day.

The variance means small samples are noisy; differences need to be larger to reach significance.

### 4.2 Reducing variance — paired comparison

Where possible, compare variants on the *same inputs*. The judge scores Variant A's output and Variant B's output on the same case; the per-case difference is the comparison unit; reduces variance dramatically.

Implementation: a shadow eval where both variants run on the same input (one in production, one in shadow). Judge scores both; compares.

Per-paired comparison, sample sizes drop to hundreds rather than thousands for the same statistical power.

### 4.3 Reducing variance — judge calibration

Per [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md), judge calibration reduces inter-judge variance. A well-calibrated judge scores the same case with low variance across runs.

Calibrate the judge before the A/B; revisit during long experiments.

### 4.4 Multiple comparison correction

If the experiment examines many slices (case type, persona, tenant), the p-value threshold should be adjusted (Bonferroni or similar) to maintain the overall false-positive rate. Otherwise, with 20 slices and p < 0.05, one false positive is expected by chance.

Discipline: pre-specify the slices that matter (per section 3.1); apply correction. Don't post-hoc fish.

### 4.5 Sequential analysis

If the team monitors the experiment as it runs and stops when a threshold is hit, the p-value is inflated (peeking). Two corrections:

- **Pre-commit to a fixed sample size.** Don't stop early.
- **Sequential probability testing.** Use methods designed for repeated checking (e.g., alpha-spending functions); higher overhead but allows valid mid-experiment decisions.

For most prompt A/B tests, fixed sample size is simpler. If the experiment is long and the team needs interim decisions, sequential is the technically right choice.

### 4.6 Effect-size estimation

A finding "Variant B is better than A, p < 0.05" doesn't say by how much. Report the effect size (e.g., 3.2% relative improvement; 95% CI [1.8%, 4.6%]).

Decisions hinge on effect size:

- Large effect size, even modest significance → promote.
- Small effect size, high significance → may or may not be worth promoting (cost / risk vs benefit).
- Inconclusive — neither significant nor large.

### 4.7 The "stat sig vs practical sig" distinction

A statistically significant result may not be practically significant. A 0.5% improvement at p < 0.001 may not justify the engineering cost of the prompt change.

Pre-commit to the minimum practically-significant effect size; ignore significance below that.

---

## 5. Online eval as the measurement instrument

The A/B's measurement quality is its instrument's quality. Online eval is the instrument.

### 5.1 What online eval is

Per [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md). Summary:

- A sample of production requests is selected for evaluation.
- The judge (LLM, sometimes human, rarely rule-based) scores them.
- Scores aggregate; trends are visible.

For A/B testing, online eval is run on each variant's traffic; the variant-level aggregates feed the comparison.

### 5.2 The judge

Critical to the A/B's validity:

- Versioned (per [prompt-versioning.md](./prompt-versioning.md)); judge prompt doesn't change during the experiment.
- Calibrated against human evaluation periodically; agreement rate measured.
- Run on enough samples to be reliable (judge variance reduced by sample size).
- Blind to which variant produced the output (avoid bias).

### 5.3 The sampling rate

Higher sampling rate → more data → faster conclusion. Lower sampling rate → less cost.

For typical prompt A/B tests: 10–25% of production traffic sampled for online eval; both variants sampled at the same rate. Cost: ~$0.005–$0.02 per sample × samples per day.

### 5.4 The sampling discipline

- **Stratified sampling.** Ensure each pre-specified slice gets adequate representation. Pure random can under-sample rare slices.
- **Consistent sampling.** Don't change sampling mid-experiment; the result won't be comparable.
- **Time-of-day awareness.** Production traffic varies; sample across time of day proportionally.

### 5.5 The "what the judge can't see" problem

The judge scores outputs, not outcomes. It doesn't see:

- Whether the user was satisfied.
- Whether the user followed up.
- Whether the user's task was completed downstream.
- Whether the user came back tomorrow.

For some features, outcome data exists (engagement metrics; downstream completion); use it as a secondary signal. For features without it, the judge is the main instrument and its limitations are accepted.

### 5.6 The human eval

For high-stakes decisions, human evaluators score samples too. Slower; more expensive; higher-quality signal.

Typical use: pre-launch human eval on a smaller sample to validate the judge's accuracy; ongoing judge-based monitoring during the experiment.

### 5.7 The "judge favours one variant systematically" failure

If the judge is biased toward one variant's style or format, the A/B is invalid. Detection:

- Calibration against humans on a sample of each variant's outputs; check for systematic differences.
- Mix variant identity in the judge's input randomly so the judge can't detect; verify scoring is consistent.

Mitigation: improve the judge prompt; or accept the bias and interpret accordingly; or use multiple judges and average.

---

## 6. Traffic splitting mechanics

The engineering layer.

### 6.1 Splitting at the prompt-version-pin level

The cleanest implementation: a feature flag (or experimentation framework) selects the prompt version for each request:

```python
prompt_version = experiment_assignment.get(
    experiment_id="care-coord-v23-vs-v24",
    user_id=user_id,  # for stickiness
    variants=["v23", "v24"],
    weights=[0.5, 0.5],
)
prompt = prompt_store.get("care-coordinator-system", prompt_version)
# ... rest of the request
```

The assignment is logged (so the per-variant aggregation works). The prompt is fetched by the assigned version.

### 6.2 The experiment-platform integration

Most teams use an experimentation platform (LaunchDarkly, Statsig, Eppo, in-house). The platform:

- Manages experiment definitions (id, variants, weights, targeting).
- Assigns variants per user (consistent for sticky experiments).
- Logs assignments.
- Provides analysis tools.

The prompt-A/B integration is the platform's standard pattern; nothing prompt-specific.

### 6.3 Variant labelling

Every request is labelled with its variant assignment; the label propagates to:

- Online eval (per-variant scoring).
- Traces (per [agent-observability.md](../agent-engineering/agent-observability.md)).
- Cost attribution (per [cost-attribution.md](../cost-and-finops/cost-attribution.md)).
- Application logs.

The variant label is the join key across all observation surfaces.

### 6.4 Targeting

Most prompt A/B tests run on a subset of traffic:

- All tenants vs specific tenants.
- All users vs internal users only (early-stage).
- All case types vs specific case types.

Targeting narrows the population; speeds the experiment for relevant samples; protects unaffected users.

### 6.5 The "experiment forgot to apply" failure

A bug or misconfiguration could cause all traffic to receive the same variant despite the split being defined. The experiment runs; data accumulates; the conclusion is "no difference" — because there really was no difference (both groups got the same thing).

Mitigation: validate the assignment in CI; alert if the actual variant split deviates significantly from the configured split.

### 6.6 The "concurrent experiments" problem

When two experiments run concurrently on the same feature, they can interact. Variant A of experiment 1 + Variant X of experiment 2 may produce different results than Variant A + Variant Y.

For prompts specifically, concurrent experiments on the *same* prompt are rare (only one experiment per prompt typically). Concurrent experiments on *different* prompts in the same request path can interact; mitigate by:

- Coordinating experiment timing (don't run interacting experiments simultaneously).
- Stratifying analysis (analyse experiment 1 within each variant of experiment 2).

### 6.7 The 100% rollout (the experiment's success)

When a variant wins:

- Move traffic to 100% on the winner.
- Remove the experiment configuration.
- Update the prompt's "current" pointer to the winner version.
- Document the experiment's result in the prompt's history.

The 100% rollout is the experiment's clean conclusion. Skipping the cleanup leaves the experiment dangling.

---

## 7. Interaction with rollback and risk management

Experiments interact with the team's rollback capability.

### 7.1 Rollback during an experiment

If a variant is found to be causing problems (regression, cost spike, latency issue), rollback the experiment:

- Set traffic to 100% on the control.
- Document the rollback reason.
- Investigate; fix; consider re-experiment.

Authority to rollback: on-call has authority for safety/cost regressions; product/engineering for quality regressions.

### 7.2 The "experiment is the rollback path" pattern

If a major prompt change ships without an A/B and causes a regression, the rollback is straightforward: revert the prompt's pin. An A/B test that's already running provides a pre-built rollback: the control is still in production and can absorb 100% of traffic.

This is a benefit of running A/Bs on risky changes — the rollback is the variant that's already serving the control population.

### 7.3 Per-tenant rollback

If a regression affects only specific tenants (the variant works for most but fails for tenant X), per-tenant rollback:

- Carve out tenant X from the experiment (always serve control).
- Investigate the tenant-specific issue.
- Continue the experiment for other tenants.

Most experiment platforms support per-tenant targeting; use it.

### 7.4 The "experiment crashed" failure

If the experiment infrastructure itself fails (assignment logic broken, traffic-split wrong), the system must default to a safe state:

- Default to control (no variant).
- Alert.
- Investigate and fix; restart the experiment after fix.

The safe default protects users; the alert ensures the team knows.

### 7.5 Insurance for the variant

For risky variants, an insurance pattern:

- The variant runs at low traffic (e.g., 5%) for a "burn-in" period.
- If no regression detected after N days, increase to 50%.
- Continue increasing if no issues.

The insurance pattern reduces blast radius if the variant has unexpected issues; common for changes with high consequence-of-failure.

### 7.6 The "no rollback path" problem

If the variant has irreversible side effects (data was written using its output; emails were sent; appointments were scheduled), rollback doesn't undo the effects. Plan for this:

- Variants with irreversible side effects warrant more care (smaller traffic, longer burn-in, more monitoring).
- Compensating action if needed.

---

## 8. The experiment lifecycle

Start. Run. Conclude. Clean up.

### 8.1 Start

- Pre-experiment specification (per section 3.1) approved by relevant stakeholders.
- Pre-prod eval cleared.
- Experiment configured in the platform.
- Monitoring alerts set up.
- Team is aware; expected duration communicated.

### 8.2 Run

- Periodic check-ins (weekly typically) on accumulated data.
- Resist peeking and early termination (per section 4.5) unless serious issues emerge.
- Monitor secondary metrics (cost, latency, error rate); investigate anomalies.
- Slice-level analyses per pre-specification.

### 8.3 Conclude

When the planned sample size is reached (or the planned duration is reached):

- Final analysis.
- Determine winner per pre-specified criteria (winner / loser / inconclusive).
- Document the result.
- Plan the rollout (or rollback or re-experiment).

### 8.4 Clean up

After the experiment concludes:

- Promote the winner to 100%; retire the loser.
- Remove experiment configuration.
- Update the prompt's current-version pointer.
- Document in the prompt's history.
- Archive the experiment data (for future reference).
- Post-experiment review (what was learned; what to do differently next time).

### 8.5 The "experiment never concluded" failure

Common pattern: an experiment launches; data accumulates inconclusively; the team gets distracted; the experiment runs for months; traffic is split for no reason; nothing is learned.

Discipline: every experiment has an end date. If the end date arrives and the experiment is inconclusive, the result is "inconclusive — no change." Stop the experiment.

### 8.6 The "permanent A/B" anti-pattern

Some teams leave A/B tests running indefinitely as a way to keep the option of either variant. This creates permanent operational complexity: two prompts maintained instead of one; ambiguity about which is canonical.

The pattern is wrong. Decide; promote; clean up. Re-experiment if needed later, with a fresh specification.

### 8.7 The post-experiment review

After each experiment:

- Was the hypothesis confirmed or denied?
- Did pre-prod eval predict the production result?
- Was the experiment design appropriate (sample size, traffic split, duration)?
- Were any surprises observed (secondary metric regressions, slice-specific effects)?
- What to do next: promote, iterate, abandon, re-experiment.
- What's the team's takeaway for future experiments?

Reviews are short (15–30 minutes). They build the team's experimental practice.

---

## 9. Worked Meridian example

Meridian's prompt A/B testing practice across the care-coordinator and other features.

### 9.1 The experimentation stack

- Statsig as the experiment platform (used for both prompt and feature experiments).
- Online eval via Braintrust (continuous LLM-judge over production samples; per-variant aggregation).
- Cost attribution via the standard pipeline (per [cost-attribution.md](../cost-and-finops/cost-attribution.md)); variant label included in attribution records.

### 9.2 The experiment cadence

- ~2-3 prompt A/B tests per quarter for care-coordinator.
- ~1 per quarter for the analytics copilot.
- Patient-summary (workflow) and patient-API copilot have lower change cadence; A/B tests rare.

Total: ~6-10 prompt A/B tests per year across all features.

### 9.3 Most common experiment shape

- 50/50 traffic split.
- Sticky per session_id (multi-turn consistency).
- 7-day duration target.
- Pre-specified primary metric: LLM-judge score on production sample.
- Pre-specified slices: case type, persona, tenant tier.
- Sample size target: 2,500 per variant on the primary metric.

### 9.4 A specific experiment (Q4-25)

**Hypothesis.** A new system-prompt component reducing verbosity (added to platform-preamble v4) will improve care-coordinator user satisfaction without losing quality.

**Pre-prod eval.** v23 (control) vs v23 + verbosity component (treatment): treatment scored 4.18 vs control 4.21 — within noise. Inconclusive pre-prod.

**A/B design.** 50/50 split, sticky per user, 7 days, 2500 samples per variant. Primary: LLM-judge score. Secondary: cost, latency, user follow-up rate, escalation rate.

**Result after 7 days, 2,800 samples per variant:**
- Primary: treatment 4.26 vs control 4.19. Difference +1.7%, p = 0.03. Significant.
- Cost: treatment 8% lower (shorter outputs cost less).
- Latency: treatment 12% lower.
- User follow-up rate: treatment 4% lower (users got the answer they needed faster, didn't need follow-up).
- Escalation rate: equal.

**Conclusion:** Treatment wins. Promote v24 (the treatment) to 100%. Cleanup completed within 2 days.

**Post-review.** Pre-prod eval failed to detect the difference because the golden set was older and didn't include the verbosity-sensitive cases. Eval set updated.

### 9.5 An inconclusive experiment (Q2-25)

**Hypothesis.** Reordering the system prompt's section ordering will improve quality.

**A/B design.** Standard.

**Result after planned duration, 2,600 samples per variant:**
- Treatment 4.20 vs control 4.22. Difference -0.5%, p = 0.41. Inconclusive.
- Secondary metrics similar.

**Conclusion:** No effect detected. Treatment retired; control remains. Documented as "the reordering doesn't help; don't try this specific approach again unless conditions change."

The discipline: the team stopped, accepted the null result, didn't extend looking for significance.

### 9.6 A rolled-back experiment (Q1-25)

**Hypothesis.** A new escalation-component variant would reduce inappropriate escalations.

**A/B design.** Standard, with extra monitoring on escalation appropriateness.

**Day 3 observation:** Treatment escalation rate dropped, but spot-check of treatment escalations showed several that *should* have escalated didn't (the variant was too aggressive in handling things itself).

**Action:** Rollback. Variant returned to 0% traffic; team investigated; the variant was reworked; re-experimented later (which then passed).

The discipline: monitoring caught the regression early; rollback was fast; the experiment failed cleanly; the rework didn't ship without re-validation.

### 9.7 Experiment results summary (last 12 months)

- 9 experiments concluded.
- 4 winners (promoted).
- 3 losers (control retained).
- 2 inconclusive (no change).
- 1 rolled back mid-experiment (later re-experimented and passed).

Promotion rate: 44% of experiments produce a change. The team considers this healthy — not so high that experiments rubber-stamp predetermined conclusions; not so low that experiments aren't productive.

### 9.8 The discipline that worked

- Pre-specification before launch.
- Adherence to planned end dates.
- Documented results regardless of outcome.
- Post-experiment reviews that fed back into eval-set improvements and prompt-engineering practice.

### 9.9 What didn't work initially

- **Peeking.** Early experiments saw "p < 0.05 by day 3!" excitement and early termination. Some of those didn't replicate. Discipline tightened: pre-specified sample size, no peeking for promotion decisions.
- **Inadequate sample sizes.** Several early experiments concluded "inconclusive" because they were underpowered. Now power-analyse before launching.
- **Forgotten experiments.** Two experiments ran for 2+ months unmonitored. Cleanup process introduced after.

---

## 10. Anti-patterns

### 10.1 "A/B test everything"

Every prompt change goes through A/B. Experiment count balloons; engineering time consumed; little is learned.

**Corrective.** A/B test for production-relevant uncertainty per section 2.1; most changes ship after pre-prod eval.

### 10.2 "Peeking and early termination"

The team checks the experiment daily and stops the first day "p < 0.05" appears. Many of those don't replicate.

**Corrective.** Pre-commit to sample size per section 4.5; don't stop early without sequential design.

### 10.3 "Underpowered experiment"

Sample size insufficient for the expected effect; experiment concludes "no difference" but the difference may exist.

**Corrective.** Power analysis per section 3.3 before launching.

### 10.4 "No pre-specification"

The experiment launches without success criteria; the team interprets the result subjectively.

**Corrective.** Pre-experiment specification per section 3.1.

### 10.5 "Permanent A/B"

The experiment never concludes; traffic split persists for months; the operational complexity is permanent.

**Corrective.** Lifecycle per section 8; experiments end.

### 10.6 "Judge biased toward one variant"

The judge systematically favours one variant's style; the comparison is invalid.

**Corrective.** Bias detection per section 5.7.

### 10.7 "Online eval missing"

No measurement instrument; the A/B has nothing to compare.

**Corrective.** Online eval setup per section 5 before launching A/Bs.

### 10.8 "Promoted winner not deployed cleanly"

The winner is determined; promotion drags on; the experiment configuration persists.

**Corrective.** Clean rollout per section 6.7 and section 8.4.

---

## 11. Findings (sprint-assignable)

### PROMPT-AB-001 — Severity: Critical
**Finding.** A/B tests run without pre-experiment specification; results interpreted subjectively.
**Recommendation.** Specification template per section 3.1; mandatory before launch.
**Owner.** ai-platform-eng + feature-team, sprint N+1.

### PROMPT-AB-002 — Severity: Critical
**Finding.** Experiments run without online eval; no measurement instrument.
**Recommendation.** Online eval per [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md); deploy before running A/Bs.
**Owner.** ai-platform-eng, sprint N+1.

### PROMPT-AB-003 — Severity: Critical
**Finding.** Long-running experiments (>2 months) with no conclusion or cleanup.
**Recommendation.** Lifecycle enforcement per section 8; quarterly experiment audit.
**Owner.** ai-platform-eng + product, sprint N+1.

### PROMPT-AB-004 — Severity: High
**Finding.** Peeking and early-termination practice; p-values inflated.
**Recommendation.** Fixed-sample-size discipline per section 4.5; or sequential design if needed.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### PROMPT-AB-005 — Severity: High
**Finding.** Experiments lack power analysis; many conclude "inconclusive" because underpowered.
**Recommendation.** Power analysis per section 3.3; size accordingly.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-AB-006 — Severity: High
**Finding.** Variant label not propagated to traces and cost attribution; per-variant analysis difficult.
**Recommendation.** Variant label as standard attribute per section 6.3.
**Owner.** ai-platform-eng + ops, sprint N+2.

### PROMPT-AB-007 — Severity: High
**Finding.** Judge calibration absent; judge variance and potential bias unknown.
**Recommendation.** Calibration per section 5.7; periodic re-calibration.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-AB-008 — Severity: Medium
**Finding.** Pre-specified slices not analysed at experiment conclusion; slice-specific effects missed.
**Recommendation.** Slice analyses per spec per section 3.1; multiple-comparison correction per section 4.4.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-AB-009 — Severity: Medium
**Finding.** No rollback path documented for risky experiments.
**Recommendation.** Rollback plan per section 7; per-tenant or per-segment rollback if applicable.
**Owner.** ai-platform-eng + ops, sprint N+3.

### PROMPT-AB-010 — Severity: Medium
**Finding.** Post-experiment reviews not held; learnings not captured.
**Recommendation.** Review per section 8.7; documented; fed back into eval-set improvements.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### PROMPT-AB-011 — Severity: Medium
**Finding.** Effect-size estimation absent; only "significant / not significant" reported.
**Recommendation.** Effect-size + CI per section 4.6; informs practical-significance judgment.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-AB-012 — Severity: Medium
**Finding.** Concurrent experiments on interacting paths; results confounded.
**Recommendation.** Coordination per section 6.6; analyse stratified by concurrent variant.
**Owner.** ai-platform-eng + ops, sprint N+4.

### PROMPT-AB-013 — Severity: Medium
**Finding.** Per-user vs per-request choice not deliberate; experiment may violate user-experience continuity.
**Recommendation.** Choice per section 3.5; per-user for stateful features.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-AB-014 — Severity: Medium
**Finding.** Cost and latency not monitored as secondary metrics; A/Bs ship quality winners that regress cost or latency.
**Recommendation.** Secondary metrics per section 3.2; explicit thresholds.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-AB-015 — Severity: Low
**Finding.** Burn-in pattern not used for high-consequence variants; risk exposure higher than necessary.
**Recommendation.** Burn-in per section 7.5 for high-stakes changes.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-AB-016 — Severity: Low
**Finding.** Experiment platform integration with prompt versioning is manual; promotion can drift.
**Recommendation.** Automated pin update on experiment promotion; integration with prompt store.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-AB-017 — Severity: Low
**Finding.** Paired comparison (shadow eval) not used; sample sizes larger than necessary.
**Recommendation.** Paired comparison per section 4.2 where feasible.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-AB-018 — Severity: Low
**Finding.** Experiment data not archived; historical results not queryable.
**Recommendation.** Archive per section 8.4; queryable storage; reference for future decisions.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting prompt A/B testing:

- [ ] **Sprint 0 — online eval.** Continuous LLM-judge on production samples; per-feature; calibrated.
- [ ] **Sprint 0 — experiment platform.** Pick (or use existing); confirm assignment-logging and per-variant aggregation.
- [ ] **Sprint 1 — first experiment.** Specification template; pilot experiment on a small-risk change.
- [ ] **Sprint 1 — variant labelling.** Variant in traces, cost attribution, logs.
- [ ] **Sprint 2 — power analysis discipline.** Pre-launch sample size; documented.
- [ ] **Sprint 2 — pre-specification.** Mandatory; reviewed by relevant stakeholders.
- [ ] **Sprint 3 — secondary metrics.** Cost, latency, etc. monitored as part of every experiment.
- [ ] **Sprint 3 — lifecycle enforcement.** End-date tracking; cleanup process.
- [ ] **Sprint 4 — judge calibration.** Periodic against humans; bias detection.
- [ ] **Sprint 4 — review cadence.** Post-experiment reviews; learnings into eval and practice.
- [ ] **Ongoing — quarterly experiment audit.** Are there stale experiments? Are the team's experiments productive?

For a team retrofitting A/B discipline:

- [ ] **Sprint 0 — current state audit.** What experiments are running? How many are stale? How many concluded but not cleaned up?
- [ ] **Sprint 1 — cleanup.** End stale experiments; promote winners or retire variants.
- [ ] **Sprint 1 — specification standard.** Going forward, all experiments require specification.
- [ ] **Sprint 2 — power analysis training.** Team can size experiments appropriately.
- [ ] **Sprint 3 — improved tooling.** Variant labelling, slice analysis, secondary-metric monitoring.

A team that completes the sequence runs productive A/B tests that close cleanly. A team that doesn't accumulates experiment debt and learns less from each cycle.

---

## 13. References

- [prompts-as-code-discipline.md](./prompts-as-code-discipline.md) — prompts as versioned artefacts; variants are versions.
- [prompt-versioning.md](./prompt-versioning.md) — version management for variants and promotion.
- [prompt-libraries-and-components.md](./prompt-libraries-and-components.md) — A/B testing component changes.
- [few-shot-engineering.md](./few-shot-engineering.md) — A/B testing example changes.
- [structured-output-engineering.md](./structured-output-engineering.md) — schema changes that may warrant A/B.
- [prompt-as-api-contract.md](./prompt-as-api-contract.md) — promotion of variants is a contract change.
- [prompt-anti-patterns.md](./prompt-anti-patterns.md) — including "never-ending experiments" and "no eval."
- [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — broader eval practice.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — the measurement instrument for A/B.
- [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) — judge engineering and calibration.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — pre-prod gate that complements A/B.
- [agent-engineering/agent-evals.md](../agent-engineering/agent-evals.md) — eval surfaces for agent prompts.
- [agent-engineering/agent-observability.md](../agent-engineering/agent-observability.md) — variant label in traces.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — continuous quality monitoring overlaps with online eval.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — variant in cost records; per-variant cost analysis.
- [cicd-and-eval-gates/canary-rollouts.md](../cicd-and-eval-gates/canary-rollouts.md) — canary as adjacent pattern.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/](https://github.com/jeremiahredden/ai-architecture-reference-architecture) — architectural patterns that experiments validate.
- LaunchDarkly, Statsig, Eppo — experimentation platforms.
- "Trustworthy Online Controlled Experiments" (Kohavi, Tang, Xu) — comprehensive reference on experimentation.
