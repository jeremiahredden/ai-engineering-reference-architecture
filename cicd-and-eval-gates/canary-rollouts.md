# Canary Rollouts

> **Audience.** Engineers wiring canary deployment for AI changes into the CI/CD pipeline. Platform leads whose canary is "ship to 100% and watch the dashboard." SREs whose canary criteria for code don't translate cleanly to AI workloads. **Scope.** The *engineering* pattern for canary rollout of AI changes within a CI/CD pipeline: the ramp pattern (1% → 10% → 50% → 100%), automated quality / cost / latency monitoring on the canary, automated promote-or-rollback thresholds, integration with online evals. The CI/CD-layer view of canary; the model-level mechanic lives in [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md). Pair with [shadow-traffic.md](./shadow-traffic.md) (the alternative when you cannot risk user-impacting traffic on the new version). Cross-link to [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) (the pipeline this stage sits in) and [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) (the live-judge that scores canary traffic). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Canary rollout exists because pre-production eval does not catch every production-mix issue. The eval suite is a curated set of cases the team believes are representative; the production traffic is whatever users actually send. The two distributions overlap heavily but never perfectly. The canary catches what the eval missed — before the new version reaches every user.

For code changes, canary patterns are well-understood: deploy to a small percentage of traffic, monitor error rates and latency, ramp if clean. Apply the same pattern to AI changes and most of it transfers — but the failure modes are different (quality regressions are not exception-and-error-rate failures), the monitoring signals are different (live-judge quality, not 5xx rate), and the rollback criteria are different (a quality regression of 2 points is meaningful, even if no exception was raised).

This document is the engineering pattern for AI-specific canary in the CI/CD pipeline. It is the *deployment* mechanic at the end of the pipeline, after eval / cost / latency gates have passed. The model-level mechanic (which model instances host the canary, traffic-routing logic, etc.) lives in [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md); this document is about the *pipeline-level* discipline: when canary fires, what it monitors, how it decides to ramp or rollback, who is on the hook for the decision.

The most-common failure modes I see are: (1) skipping canary because eval passed, then learning about a regression in production at 100% traffic; (2) canary criteria identical to eval criteria, so the canary catches nothing the eval missed; (3) canary windows too short to see the regressions that actually happen; (4) auto-ramp without human review on the steps that matter most (the 50% step in particular).

The honest framing: canary is the last line of defense before user-visible regression. A team that skips canary is betting their entire user base on the eval suite's coverage. A team that runs canary well operates AI changes with much smaller blast radius when something does slip through. The investment is real — canary doubles deploy-to-100% time — but the alternative is paying that time back in incident response when a missed regression goes wide.

This document is opinionated about four things:

1. **Every AI deployment goes through canary by default.** Direct-cutover deploys are emergency-hotfix mode, not the standard path.
2. **Canary criteria are distinct from eval criteria.** Eval runs on offline cases; canary runs on production traffic. The criteria must reflect production signal, not eval signal.
3. **The 50% step is human-gated by default.** 1% → 10% can be automated for low-risk changes; 50% → 100% is where humans review for high-risk changes.
4. **Canary failures are debugged, not just rolled back.** A canary that fails has produced data; treat it as a finding, not just a non-event.

Structure: (2) the canary mechanic; (3) the ramp pattern; (4) canary criteria; (5) the monitoring window; (6) the promote-vs-rollback decision; (7) automated vs human-gated steps; (8) integration with online evals; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The canary mechanic

A canary deployment runs the new version alongside the current version, routing a small percentage of traffic to the new version, monitoring metrics on the canary, and either promoting (ramping traffic) or rolling back.

### 2.1 What's running during a canary

- **Current version** (baseline): handles 99–95% of traffic, depending on canary percentage.
- **Canary version** (new): handles 1–5% of traffic.
- Both versions log to the same observability backend with version-labeled metrics so they can be compared.
- The routing layer ([model-lifecycle/canary-and-shadow-rollout.md §2](../model-lifecycle/canary-and-shadow-rollout.md)) decides per-request (or per-conversation, or per-user — see §2.4) which version handles each request.

### 2.2 What can be canaried

The patterns apply to:

- **Prompt changes.** The new prompt is served to canary traffic; the old prompt to baseline traffic.
- **Model-version bumps.** A new model version is the canary's model; the old model is the baseline's.
- **Fine-tune deployments.** A new fine-tune is canaried before broad rollout.
- **Retrieval-corpus updates.** New corpus version is canary; old is baseline.
- **Pipeline / orchestration changes.** The shape of the AI pipeline (number of LLM calls, tool wiring, prompt assembly logic) changes in canary first.
- **Combinations.** A release that bundles multiple changes (new prompt + new model) is canaried as one unit. The trade is that the canary cannot decompose which change caused any observed effect.

### 2.3 What cannot be canaried cleanly

Some changes have an all-or-nothing character:

- **Schema changes affecting downstream callers.** If the new prompt outputs a different schema, the downstream consumer must support both. Otherwise the canary's output corrupts the user's session.
- **Stateful migrations.** Database migrations, sequence renumbering, etc. — these don't fit the canary pattern.
- **Multi-instance coordination.** A change that requires every instance to agree on a parameter cannot live alongside instances using the old parameter.

For these, shadow traffic ([shadow-traffic.md](./shadow-traffic.md)) is sometimes the alternative; sometimes the change has to be feature-flagged with synchronized rollout instead.

### 2.4 The randomization unit

Just like A/B testing ([model-lifecycle/ab-model-testing.md §3](../model-lifecycle/ab-model-testing.md)), the canary's randomization unit must match the feature's state model:

- **Per-conversation** for chat / multi-turn features. Once a conversation lands on canary, all turns of that conversation stay on canary.
- **Per-user** for personal-assistant patterns where cross-conversation state matters.
- **Per-tenant** for B2B features where intra-tenant inconsistency is itself a harm.
- **Per-request** for stateless single-shot features.

Per-conversation is the default for Care-Coordinator-shaped systems. Routing layer reads conversation metadata and pins the conversation to its assigned version.

---

## 3. The ramp pattern

The canary is not a single percentage; it is a sequence of increasing percentages with monitoring between each.

### 3.1 The standard ramp

```
1% → 10% → 50% → 100%
```

Each step:

- Runs for a defined window (4 hours typical, longer for low-traffic features).
- Has criteria that must hold throughout the window.
- Ramps to the next step if criteria are met; rolls back if not.

The 1% step is the *smoke test*: does the canary work at all? Does the routing layer correctly send 1% of traffic? Are metrics flowing? Are there any immediate issues?

The 10% step is the *signal floor*: enough volume to detect medium-effect regressions in the canary criteria, but still small enough that an incident affects a minority.

The 50% step is the *production-mix check*: traffic mix at this level is representative of full production, so this is where production-mix-specific issues surface.

The 100% step is the *promotion*: full traffic on the new version, the old version retained for the rollback window (14 days typical).

### 3.2 Alternative ramps

- **Conservative ramp** for high-risk changes: 1% → 5% → 10% → 25% → 50% → 100%. Each step's window may also be longer.
- **Aggressive ramp** for low-risk changes (a sub-version model refresh that has cleared all eval gates and prior canaries): 5% → 50% → 100%.
- **Tenant-staged ramp** for multi-tenant systems: deploy to one tenant first, monitor, then expand.

The default ramp is 1% → 10% → 50% → 100%; deviations are documented in the release.

### 3.3 Per-step window length

The window length depends on:

- **Traffic volume.** A 1% canary on a feature with 1M requests/day = 10K canary requests/hour. A 4-hour window has 40K canary requests — statistically meaningful. A 1% canary on a feature with 1K requests/day = 0.4 canary requests/hour. The window must extend long enough to gather enough data.
- **Diurnal pattern.** Some features have very different traffic mix at different times. The window should ideally cover a representative slice; if traffic peaks at 10 AM and the canary starts at 4 AM, extend the window to cover the peak.
- **Criteria sensitivity.** A criterion measured on a noisy metric (live-judge quality) needs more data than a criterion measured on a low-noise metric (error rate).

A practical floor: 1000 canary sessions or 4 hours, whichever is longer.

### 3.4 The 50% holding window

The 50% step typically holds longer than earlier steps:

- 1% step: 4 hours (or until 1000 sessions, longer).
- 10% step: 4 hours.
- 50% step: 12–24 hours.
- 100%: indefinite (this is the promotion).

The 50% step is held longer because it's the last step before full traffic, and it covers a more representative slice of the day-of-week mix. A 24-hour 50% step covers an entire daily cycle.

---

## 4. Canary criteria

The criteria are what the canary monitors. They must be tuned to catch real regressions and not fire on noise.

### 4.1 The four criterion families

**Quality.**

- Live-judge score (per [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md)) on canary traffic vs baseline.
- User signals (thumbs-up rate, conversation continuation rate, escalation rate).
- Refusal rate.

**Cost.**

- Cost per request on canary vs baseline.
- Cost per conversation on canary vs baseline.

**Latency.**

- p50 / p95 / p99 latency on canary vs baseline.
- Time-to-first-token for streaming features.

**Reliability.**

- Error rate (5xx, timeout, exception).
- Tool-call success rate.
- Schema-compliance rate (does the structured output validate?).

### 4.2 The thresholds

| Criterion | Allowed delta from baseline | Block-deploy threshold |
|---|---|---|
| Live-judge quality | -0.5 points (small canary noise tolerance) | -1.5 points |
| Refusal rate | +1pp | +3pp |
| Error rate | +0.1pp | +0.3pp |
| Schema-compliance rate | -0.1pp | -0.5pp |
| Cost per request | +15% | +30% |
| p95 latency | +20% | +50% |
| p99 latency | +25% | +75% |

The thresholds are workload-dependent; the table is a starting point. Tune per feature.

### 4.3 The "noisy vs safety" distinction

Some criteria are noisy (live-judge quality at small N has substantial confidence intervals). Some are not (error rate is much cleaner).

For noisy criteria:

- The "small canary noise tolerance" allows modest movement before triggering rollback.
- The block-deploy threshold is tight; movement past it triggers rollback regardless of N.

For non-noisy criteria:

- Tight thresholds throughout.
- Block-deploy on any sustained deviation.

### 4.4 The pre-canary baselining

Before the canary starts, baseline metrics are measured:

- Current version's quality / cost / latency / reliability over the last 24 hours (or longer).
- The baseline is the comparison point.
- The baseline is refreshed at each ramp step; the 10% canary compares to the 90% baseline at that moment, not to a stale baseline from yesterday.

### 4.5 The safety overrides

Regardless of statistical criteria, certain conditions trigger immediate rollback:

- A safety incident traceable to canary.
- A regulatory violation traceable to canary.
- User-reported harms above a threshold.
- A specific high-priority customer experiencing degradation.

These overrides are *signals*, not thresholds; they are triggered manually by the SRE on-call or product owner, and the canary is rolled back even if the statistical criteria appear clean.

---

## 5. The monitoring window

What happens during the canary's monitoring window.

### 5.1 The dashboard

A canary dashboard shows, per criterion:

- Canary metric in real time.
- Baseline metric.
- Delta (canary minus baseline).
- Confidence interval (where applicable, especially for live-judge).
- Threshold lines (allowed delta, block-deploy delta).

The dashboard is bookmarked, on-call has access, the AI Platform team has it on a wall during business hours.

### 5.2 The monitor's responsibilities

For each canary, an *engineer is on the hook* — usually the engineer who wrote the change, with the AI Platform on-call as backup:

- Reviews the canary dashboard at each ramp step.
- Approves the ramp (manually if human-gated; via automated criteria otherwise).
- Triggers rollback if criteria are violated.
- Investigates and documents any non-trivial deltas (even if within tolerance).

### 5.3 Automated alerts

Canary monitoring triggers alerts when:

- A criterion crosses its allowed-delta line.
- A criterion approaches its block-deploy line.
- A criterion has been at an unusual value for > 30 minutes (catches drift that doesn't trip a static threshold).
- The canary's request volume is anomalously low (the canary may not be receiving traffic; routing failure).

Alerts route to the on-the-hook engineer and the AI Platform on-call.

### 5.4 The "soak time"

Between ramp steps, the canary "soaks" — runs at the current percentage for the window length. The soak time is to:

- Let metrics stabilize.
- Cover the diurnal pattern.
- Surface delayed regressions (some regressions appear hours into a session, not on the first turn).

Skipping the soak time is the most common ramp anti-pattern.

### 5.5 Concurrent canaries

What if two changes are being canaried at the same time? Two cases:

- **Independent canaries.** Two separate features each have their own canary. The canaries don't interact. This is fine.
- **Stacked canaries.** Two changes to the same feature are canaried simultaneously (an A canary at 1% and a B canary at 1% within the same feature). This is *not* fine: the canaries cannot distinguish which change caused any observed effect, and the criteria mix the two.

The rule: one canary per feature at a time. Stacked canaries are flagged at deploy gate and refused.

---

## 6. The promote-vs-rollback decision

At each ramp step, a decision is made.

### 6.1 The automated criteria check

For low-risk changes with criteria met:

- Quality delta within tolerance.
- Cost delta within tolerance.
- Latency delta within tolerance.
- Reliability delta within tolerance.
- Window duration met.
- Minimum N met.

If all hold: auto-ramp.

### 6.2 The human gate

For high-risk changes (model swap, major prompt rewrite, fine-tune deployment) or for the 50% step on any change:

- Automated criteria still calculated.
- Dashboard reviewed.
- Engineer / lead approves the ramp explicitly.
- Approval is logged.

The human gate is not a rubber stamp; it's the place where the engineer who is on the hook makes the call after looking at the data.

### 6.3 The rollback decision

If criteria are violated:

- Automated rollback for clear violations (delta past block-deploy threshold).
- Human-initiated rollback for ambiguous cases (within tolerance but engineer has concerns).
- Safety-override rollback for the safety conditions in §4.5.

Rollback restores the prior version's full pinned set (per [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md)).

### 6.4 The post-rollback investigation

A rollback is a finding, not a non-event:

- The on-the-hook engineer documents what happened.
- The eval team reviews whether an eval case should be added that would have caught it.
- The AI Platform team reviews whether the canary criteria should have caught it earlier.
- The release is held until the underlying issue is addressed.

### 6.5 The "I want to ship anyway" case

Sometimes a canary fails on a criterion that the team decides is acceptable. For instance, a deliberate quality trade for a 30% cost reduction.

The discipline:

- The override mechanic (per [eval-gate-design.md §8](./eval-gate-design.md)) applies to canary too.
- The override is documented in the release.
- Senior approval is required.
- The override has an expiration; lapsed overrides reblock.

Override is allowed; bypass is not.

---

## 7. Automated vs human-gated steps

Different ramp steps have different gate disciplines.

### 7.1 The default discipline by change type

**Low-risk changes** (sub-version model refresh that cleared eval; prompt tweaks):

- 1% step: auto-ramp on criteria.
- 10% step: auto-ramp on criteria.
- 50% step: human-gated.
- 100% promotion: human-gated.

**Medium-risk changes** (minor model version bump; new prompt; corpus refresh):

- 1% step: human-gated (read the dashboard before ramping).
- 10% step: auto-ramp on criteria.
- 50% step: human-gated.
- 100% promotion: human-gated.

**High-risk changes** (major model version; fine-tune deployment; structural pipeline change):

- 1% step: human-gated.
- 10% step: human-gated.
- 50% step: human-gated.
- 100% promotion: human-gated.

### 7.2 Who is the human

The human-gate approver depends on step and change type:

- 1% step: the on-the-hook engineer.
- 10% step: the on-the-hook engineer.
- 50% step: AI Platform lead or tech lead.
- 100% step for high-risk: AI Platform lead + product owner.

Approvals are logged in the release.

### 7.3 The off-hours discipline

For high-risk changes, the canary should not start its 50% or 100% step during off-hours (overnight, weekends) unless approved as part of the release plan. The discipline:

- Canary plans include the timing.
- Auto-ramps that would cross into off-hours are held until business hours resume.
- Emergency on-call has the authority to override and proceed.

This avoids the "promoted at 3 AM, regression detected at 9 AM" pattern.

### 7.4 The cascade-of-confidence

For repeated canaries of the same shape (e.g., the team has done 20 prompt-tweak canaries successfully), the gate discipline can relax:

- The first canary of a kind: full human gates.
- After 5 successful canaries of similar shape: automate the 1% step.
- After 20: automate through the 10% step.

The cascade is a team-level decision, documented in the canary policy.

---

## 8. Integration with online evals

The canary depends on the online-eval infrastructure to score quality on canary traffic.

### 8.1 The live-judge scores canary traffic

Per [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md), the live-judge scores conversations (or a sample of them) using a defined rubric. During canary:

- The judge scores both canary and baseline traffic.
- Scores are aggregated per arm (canary vs baseline).
- The delta is the live-judge quality criterion.

### 8.2 The judge must be stable

If the judge model or rubric changes during the canary, the comparison is invalid. The judge is pinned for the canary window.

### 8.3 Sampling rate for the judge

The judge is expensive (an LLM call per scored conversation). Strategies:

- 100% sampling on canary; 1–10% sampling on baseline (the baseline's score is more stable, so less data is needed).
- Stratified sampling if conversation types differ in importance.
- Quota-based sampling for low-volume conversation types (ensure each type gets enough samples).

### 8.4 The judge's confidence interval

The judge has its own variance. A canary delta of 0.2 points may be within the judge's confidence interval. The criterion threshold (§4.2) must account for this; thresholds set inside the judge's confidence interval will fire on noise.

### 8.5 Live-judge versus other quality signals

The live-judge is the primary quality signal but not the only one:

- User thumbs-up is direct user signal (less noisy than judge but slower-arriving).
- Conversation continuation rate is behavioral signal.
- Escalation rate captures product-level harms.

Canary criteria combine all of these. The judge is the headline; the others are confirming signals.

---

## 9. Worked Meridian example: canary of a clinical-knowledge model version bump

The Care Coordinator's clinical-knowledge model is on `claude-opus-4-7@2026-04-12`. A sub-version refresh `2026-05-15` cleared the offline eval (per [model-lifecycle/ab-model-testing.md §10](../model-lifecycle/ab-model-testing.md)'s example). The team is now canarying the refresh.

### 9.1 The canary plan

- **Change type:** medium-risk (sub-version model bump).
- **Ramp:** 1% → 10% → 50% → 100%.
- **Per-step windows:** 4 hours for 1% and 10%; 24 hours for 50%; 100% is the promotion.
- **Gates:** 1% step is human-gated (medium-risk default); 10% auto-ramp; 50% human-gated; 100% promotion is human-gated.
- **Randomization unit:** per-conversation.
- **Criteria thresholds:** standard table from §4.2.

### 9.2 The deploy

- Release `2026.05.25-r3` is built with the new model pin.
- Deploys the canary version to one instance alongside three baseline instances.
- The routing layer is configured to assign 1% of new conversations to the canary version.
- Monitoring dashboards are live.

### 9.3 The 1% step (4 hours, ~600 canary conversations)

- Live-judge quality: 7.45 (canary) vs 7.42 (baseline). Delta +0.03. Within noise.
- Refusal rate: 1.0% vs 1.1%. Within noise.
- Cost per conversation: $0.083 vs $0.082. Delta +1.2%. Within tolerance.
- p95 latency: 1.32s vs 1.34s. Delta -1.5%. Within tolerance.
- Error rate: 0.04% vs 0.04%. Unchanged.
- Schema-compliance: 99.8% vs 99.7%. Within tolerance.

**Engineer review:** delta on every criterion is within tolerance, but live-judge sample is small (~600 conversations). Engineer waits the full window before promoting; confirms metrics are stable across the four hours; approves ramp to 10%.

### 9.4 The 10% step (4 hours, ~6000 canary conversations)

- Live-judge quality: 7.44 vs 7.42. Delta +0.02.
- Refusal rate: 1.0% vs 1.1%.
- Cost per conversation: $0.083 vs $0.082. Delta +1.4%.
- p95 latency: 1.31s vs 1.34s.
- Error rate: 0.03% vs 0.04%.
- Schema-compliance: 99.9% vs 99.7%. Slight improvement.

**Auto-ramp criterion satisfied.** Auto-ramps to 50%.

### 9.5 The 50% step (24 hours, ~150K canary conversations)

The window covers a full daily cycle including the morning peak.

- Live-judge quality: 7.43 vs 7.41. Delta +0.02. Very stable across the day.
- Refusal rate: 1.0% vs 1.1%. Slightly better.
- Cost per conversation: $0.083 vs $0.082. Delta +1.5%.
- p95 latency: 1.31s vs 1.33s.
- Error rate: 0.03% vs 0.04%.
- Schema-compliance: 99.9% vs 99.7%.

A diurnal note: during the morning peak (8–10 AM local), canary's p95 latency rose to 1.45s while baseline rose to 1.51s. The canary is *better* under load; reassuring.

**Human gate at 50%.** AI Platform lead reviews the dashboard; approves promotion to 100%.

### 9.6 The 100% promotion

- All traffic moves to the new model version.
- The prior model pin is moved to "rollback target."
- The release artifact is finalized.
- The eval baseline is updated to reflect the new model's metrics.

### 9.7 Total canary duration

- 1% step: 4 hours.
- 10% step: 4 hours.
- 50% step: 24 hours.
- Total: 32 hours (one full day plus a working day).

The team accepts the 32-hour duration as the cost of confident promotion.

### 9.8 The counterfactual: a regression-detected canary

In a hypothetical where the canary detected a regression:

- During the 10% step, refusal rate on canary rises to 2.5% vs baseline 1.1%. Delta +1.4pp.
- The threshold for refusal rate is +1pp allowed, +3pp block. The delta is past the allowed line but below the block line.
- The on-the-hook engineer reviews: the canary version is over-refusing on a class of clinical queries that the baseline answered.
- The engineer triggers rollback. The 10% canary returns to 0%; the old model is the sole serving version.
- Investigation: the new sub-version has tightened refusal on a specific clinical pattern. The team determines whether the new behavior is correct (the baseline was under-refusing) or wrong (the new version is over-refusing). The decision drives whether the release is reworked or whether the refusal pattern is intentionally accepted.

### 9.9 Findings closed

- **ARCH-CARE-079** (model version bumps deployed direct-to-100%; no canary).
- **ARCH-CARE-080** (canary criteria identical to eval criteria; production-mix issues not caught).
- **ARCH-CARE-081** (canary windows too short to cover diurnal pattern; morning-peak issues missed).
- **ARCH-CARE-082** (no human gate at 50%; promotions happened off-hours unattended).
- **ARCH-CARE-083** (rollback path for canary failures not documented; rollback ad-hoc).

---

## 10. Anti-patterns

### 10.1 The skip-canary deploy

The team trusts the eval gate enough to skip canary. A regression that the eval suite didn't cover ships to 100%; users see it before the team does.

The fix: canary is the default. Direct-cutover is emergency mode, used rarely and audited.

### 10.2 The eval-criteria canary

Canary criteria are identical to eval criteria. The canary "passes" the same way the eval did; nothing new is learned. The canary is procedural, not protective.

The fix: criteria from production telemetry (live-judge, user signals, behavioral metrics), not just from offline eval.

### 10.3 The too-short window

The team runs the 1% canary for 30 minutes "to keep the deploy moving." Insufficient data to detect anything; the canary is decoration.

The fix: minimum 1000 sessions or 4 hours, whichever is longer, per step. Adjust for low-volume features.

### 10.4 The same-baseline-every-step

The canary at 10% compares against the baseline measurements taken before the 1% step started. The baseline has drifted (a different time of day, different traffic mix). The delta is misleading.

The fix: refresh baseline at each step. Compare the 10% canary against the concurrent 90% baseline.

### 10.5 The auto-ramp through everything

Every step is auto-ramped on criteria. A regression that is borderline doesn't trip the automated check but a human would have caught. The change ships; users feel it.

The fix: human gate at the 50% step minimum; more steps human-gated for higher-risk changes.

### 10.6 The off-hours auto-promotion

The canary auto-ramps to 100% at 3 AM. A regression detected at 9 AM has had 6 hours of full-traffic exposure.

The fix: high-risk canaries are held outside business hours; the auto-ramp respects an "off-hours don't promote" window.

### 10.7 The stacked canary

Two changes to the same feature are canaried simultaneously. The criteria mix both; the team cannot decompose any observed effect.

The fix: one canary per feature at a time. Deploy gate refuses overlapping canaries.

### 10.8 The "we always roll back if anything is yellow" panic

The team rolls back any canary that shows any non-zero delta on any criterion. The criteria threshold is effectively zero. Every canary fails; the team learns to bypass criteria.

The fix: set thresholds at meaningful levels, allow modest deltas without rollback, reserve rollback for sustained or material deltas.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-CR-001 | AI deployments skip canary; direct-to-100% standard | High | Make canary the default; direct-cutover is emergency-only | AI Platform + SRE |
| CICD-CR-002 | Canary criteria identical to offline eval criteria | High | Define production-telemetry criteria per §4; live-judge + user signals + behavioral | AI Platform + Eval Eng |
| CICD-CR-003 | Canary windows too short to detect production-mix issues | High | Min 1000 sessions or 4h per step; 24h at 50% per §3.3 | AI Platform |
| CICD-CR-004 | Baseline not refreshed at each ramp step | Medium | Concurrent-baseline pattern per §10.4 | AI Platform |
| CICD-CR-005 | Auto-ramp at all steps including 50%; no human review | High | Human gate at 50% minimum per §7.1 | AI Platform |
| CICD-CR-006 | Stacked canaries on same feature; criteria mix changes | Medium | One canary per feature; deploy gate refuses overlaps | AI Platform |
| CICD-CR-007 | Off-hours auto-promotion without holding window | Medium | Off-hours don't-promote window for high-risk changes | AI Platform + SRE |
| CICD-CR-008 | No on-the-hook engineer per canary; rollback decisions ad-hoc | Medium | Engineer designated per canary; AI Platform on-call backup | AI Platform |
| CICD-CR-009 | Rollback path not documented; rollback ad-hoc | High | Document rollback path per [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) | AI Platform + SRE |
| CICD-CR-010 | Canary failures not investigated; rollback treated as non-event | High | Every rollback produces a finding; eval case added if applicable | AI Platform + Eval Eng |
| CICD-CR-011 | Canary criteria thresholds set inside live-judge confidence interval | Medium | Thresholds above 2× judge noise floor per §8.4 | Eval Eng |
| CICD-CR-012 | Routing-layer issues hide canary traffic; canary receives 0 requests undetected | Medium | Monitor canary request volume; alert on anomalously low volume | SRE |
| CICD-CR-013 | Randomization unit mismatched to feature state model | Medium | Per-conversation default; document choice per §2.4 | AI Platform |
| CICD-CR-014 | Override mechanism for canary criteria not documented | Medium | Override per §6.5; documented justification + senior approval + expiration | AI Platform |
| CICD-CR-015 | Live-judge sampling rate too low at small canary percentage | Medium | 100% sample on canary, 1–10% on baseline per §8.3 | Eval Eng |
| CICD-CR-016 | Canary dashboard not on-call accessible | Low | Bookmark in on-call runbook; alerts route to on-call | SRE + AI Platform |
| CICD-CR-017 | Safety-override path absent; statistical criteria the only signal | High | Safety override per §4.5 (manual rollback trigger by SRE / product) | SRE + Product |
| CICD-CR-018 | No cascade-of-confidence for repeated successful canaries; same rigor every time | Low | Document cascade per §7.4; relax gates after demonstrated history | AI Platform |

---

## 12. Adoption checklist

- [ ] Canary is the default deployment path for AI changes; direct-cutover is emergency-only.
- [ ] Standard ramp `1% → 10% → 50% → 100%` defined; alternative ramps documented per change type.
- [ ] Per-step windows defined; minimum 1000 sessions or 4 hours per step.
- [ ] 50% step holds ≥ 24 hours to cover diurnal cycle.
- [ ] Canary criteria derived from production telemetry (live-judge, cost, latency, reliability, user signals).
- [ ] Thresholds tuned per workload; above live-judge noise floor.
- [ ] Baseline refreshed at each ramp step (concurrent-baseline pattern).
- [ ] Randomization unit chosen per feature state model; per-conversation default.
- [ ] Human gate at 50% step (minimum); more steps gated for higher-risk changes.
- [ ] One canary per feature; deploy gate refuses overlapping canaries.
- [ ] Off-hours don't-promote window enforced for high-risk changes.
- [ ] On-the-hook engineer designated per canary; AI Platform on-call backup.
- [ ] Rollback path documented and tested; restoration of full pinned set verified.
- [ ] Safety-override path for manual rollback (SRE / product) regardless of statistical criteria.
- [ ] Every canary rollback produces a finding; eval case added when applicable.
- [ ] Override mechanism for canary criteria mirrors eval-gate-override discipline.
- [ ] Live-judge runs at 100% on canary, 1–10% on baseline.
- [ ] Canary dashboard bookmarked in on-call runbook; alerts route to on-call.

---

## 13. References

**Internal:**

- [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) — the pipeline this canary stage sits in.
- [shadow-traffic.md](./shadow-traffic.md) — the alternative when canary cannot risk user-impacting traffic.
- [eval-gate-design.md](./eval-gate-design.md) — the offline gate that runs before canary.
- [release-artifacts-for-ai.md](./release-artifacts-for-ai.md) — the artifact format including canary results.
- [prompt-version-pinning.md](./prompt-version-pinning.md) — the pinned prompts the canary deploys.
- [model-version-pinning.md](./model-version-pinning.md) — the pinned models the canary deploys.
- [dataset-version-pinning.md](./dataset-version-pinning.md) — the pinned datasets the canary deploys.
- [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) — the model-level canary mechanic.
- [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) — the rollback path on canary failure.
- [model-lifecycle/ab-model-testing.md](../model-lifecycle/ab-model-testing.md) — A/B as the formal-comparison alternative to canary.
- [model-lifecycle/model-promotion.md](../model-lifecycle/model-promotion.md) — the promotion the canary gates.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — the live-judge that scores canary traffic.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — quality monitoring that may force a safety override.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — canary alert routing.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — the cost signal for canary cost criterion.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — when a canary failure becomes an incident.

**Cross-repo (architecture sibling):**

- [model-strategy/model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — architecture-side framing of model migrations canaries support.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
- [reference-systems/adoption-sequencing-across-systems.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/adoption-sequencing-across-systems.md) — adoption sequence including canary discipline.
