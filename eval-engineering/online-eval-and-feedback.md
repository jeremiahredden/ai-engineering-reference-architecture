# Online Eval and Feedback

> **Audience.** Engineers and tech leads building the production-side eval signal — what runs against live traffic, how user feedback flows back into the eval suite, what production quality SLIs look like. **Scope.** The *engineering* practice of online eval (sampled judge runs on production traffic), explicit user feedback, implicit signals, and the closed-loop pattern back into the golden set and regression suite. Pair with [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) (the judge), [golden-set-design.md](./golden-set-design.md) (the growth target), [regression-eval-suites.md](./regression-eval-suites.md) (the regression-case capture). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Offline eval (per [eval-engineering-playbook.md](./eval-engineering-playbook.md)) tells the team about quality on the cases the team thought to test. It does not tell them about quality on the cases that surface in production unexpectedly. The gap between "the eval suite covers" and "production users encounter" is the gap online eval closes.

The online eval discipline produces three things the offline practice cannot:

1. **A real-time quality SLI.** Sampled judge runs on production interactions produce a continuously-updated pass rate. The SLI is the foundation for the quality-regression alerts per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).
2. **A feedback loop into the eval suite.** Production interactions that surface novel quality issues become candidate cases for the golden set or regression suite.
3. **User-perceived quality data.** Explicit feedback (thumbs / ratings) and implicit signals (retry, edit, abandonment, escalation) give signal that the offline judge cannot.

This document is the depth on the engineering practice. The [eval-engineering-playbook.md](./eval-engineering-playbook.md) section 7 introduces the pattern; this is the structural pattern at scale.

This document is opinionated about three things:

1. **The online judge runs continuously; sampling is calibrated.** 100% sampling is too expensive; 1% sampling produces noisy signal. 5-10% is the typical sweet spot for high-volume workloads; high-stakes workloads sample more.
2. **Feedback signals feed the eval suite.** Thumbs-down responses with free-text comments are mined for golden-set candidates. The loop is engineered, not informal.
3. **The production quality SLI is a real SLI.** It has thresholds, alerts, dashboards, runbooks. It is operated like latency and availability SLIs, not as an interesting trend line.

Structure: (2) the online judge architecture; (3) sampling strategy; (4) explicit feedback collection; (5) implicit signal instrumentation; (6) the feedback loop into the eval suite; (7) production quality SLI; (8) integration patterns; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The online judge architecture

The online judge runs against production traffic; the architecture mirrors the offline judge with adaptations for production scale.

### 2.1 The shape

```
Production interaction → response delivered to user
                              │
                              ▼ (sampled)
                       Online judge invocation
                              │
                              ▼
                       Verdict recorded
                              │
                              ▼
                       Aggregate SLI / dashboards
```

The judge runs asynchronously, after the response has been delivered to the user. The judge does not block the user-facing path; production latency is unaffected.

### 2.2 The reference-free mode

The online judge typically runs in reference-free mode (per [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) section 2.2): there is no expected_answer for production interactions (each one is novel). The judge scores against the rubric only:

- Factual correctness (is the answer plausibly correct given the apparent context?).
- Citation accuracy (do cited claims appear in cited sources?).
- Faithfulness (is the answer grounded in retrieved content, for RAG features?).
- Format adherence (does the answer follow the expected format?).
- Refusal correctness (was refusal appropriate, when refusal occurred?).

Reference-free judging is inherently noisier than reference-based; calibration accordingly is lower (typically 75-85% agreement with human review, vs 85%+ for reference-based).

### 2.3 The composition of judges

Some signals are best from specialized sub-judges:

- A citation-accuracy judge that consumes the retrieval lineage.
- A faithfulness judge that consumes retrieved chunks.
- A composite quality judge for the overall verdict.

The online judging pipeline runs each in parallel per sampled interaction; verdicts compose to an overall pass/fail.

### 2.4 The cost shape

Online judging cost scales with sample rate and traffic volume:

- Care Coordinator at ~3K interactions/day × 10% sample = 300 judged interactions/day.
- ~3 sub-judges per interaction × ~$0.012 per judge call = ~$0.036 per judged interaction.
- Daily online judging cost: ~$11.

The cost is meaningful but bounded; budgeting per section 4.6 ensures it stays in proportion to the production cost.

---

## 3. Sampling strategy

How to sample production for online judging — without breaking the budget and without missing important cases.

### 3.1 The sample rate

- **High-volume, low-stakes workloads:** 1-5% sample. Volume produces enough signal at low rates.
- **Medium-volume workloads:** 5-15% sample. Standard.
- **Low-volume, high-stakes workloads:** 50-100% sample. Volume is low enough to afford; signal density matters more.

For Meridian Care Coordinator (~3K/day, high-stakes clinical): 10% sample produces ~300 judged interactions/day. Statistically meaningful for the SLI.

### 3.2 The stratification

Random sampling biases toward common case classes. Stratified sampling ensures representation:

- Sample evenly across case classes (clinical-protocol, drug-interaction, etc.).
- Over-sample high-stakes classes (drug-interaction sampled at 30% even though it's 10% of traffic).
- Under-sample noisy classes (the trivial-acknowledgment class sampled at 1% even though it's frequent).

The stratification produces a sample that supports per-class SLI computation.

### 3.3 The tail-based augmentation

Random sampling misses the rare. Per [trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) section 5.2, tail-based augmentation keeps traces that match interesting criteria:

- Cases that triggered fallback (per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md)).
- Cases where cost circuit-breaker fired.
- Cases where agent loop ran to budget.
- Cases with thumbs-down user feedback.
- Cases where retrieval returned empty.

These criteria identify likely-failure cases. Augmenting the random sample with tail-based selection produces a signal that catches the rare cases.

### 3.4 The judge call rate limit

To keep cost bounded, the judge call rate is itself rate-limited. If the sample produces more candidates than the budget allows, the team:

- Reduces the random sample rate temporarily.
- Maintains the tail-based augmentation (the high-value samples).
- Investigates the traffic spike that caused the over-sampling.

The rate limit prevents the online judge from becoming a cost incident itself.

### 3.5 The per-tenant sampling

For multi-tenant systems, per-tenant sampling decisions:

- Premium-tier tenants may get higher sampling (their interactions are more valuable; the team wants better signal on their experience).
- Standard-tier tenants get the platform default.
- Internal-tooling tenants get higher sampling (they're the team's dogfooding).

Per-tenant sampling rates are configuration, not code.

---

## 4. Explicit feedback collection

User-provided feedback signals.

### 4.1 The signal types

- **Thumbs up / down.** Binary signal per interaction.
- **Rating (1-5 stars or similar).** Scalar signal.
- **Free-text feedback.** "What was wrong?" or "What would have been more helpful?"
- **Categorical feedback.** Multiple-choice ("Was the answer accurate? Was the tone appropriate?").
- **Structured corrections.** Some workflows let users edit the AI's output before using it; the edit is itself a signal.

Different workloads have different signal types. The Meridian Care Coordinator uses thumbs + optional free-text + categorical (clinical-accuracy / completeness / appropriate-disclaimers).

### 4.2 The collection UI

The collection UI matters:
- Easy to provide feedback (one click for thumbs).
- Not intrusive (does not block the user's workflow).
- Optional rich feedback (clicking thumbs-down opens a free-text field).
- Anonymous by default (some users won't provide feedback if their identity is recorded).

The UI design directly affects response rate; response rates below 1% are common (most users do not rate); above 5% is unusual.

### 4.3 The feedback storage

Feedback is stored alongside the interaction trace per [trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md):

- Linked by interaction ID.
- Tagged with feedback type, value, timestamp, optional free-text.
- Available for later review and golden-set mining.

### 4.4 The aggregate signal

Aggregate dashboards show:
- Thumbs-up vs thumbs-down ratio per feature per day.
- Trend lines (week-over-week).
- Per-tenant breakdown (some tenants may experience worse quality).
- Per-question-class breakdown.

Significant changes flag potential quality issues.

### 4.5 The free-text mining

Thumbs-down responses with free-text are the highest-value feedback source. Weekly:

- Pull all thumbs-down + free-text from the past week.
- Categorize: what kind of issue? (Factual wrong / off-topic / missing information / poor format / etc.)
- For each category: are there patterns? (Multiple users reporting the same kind of issue.)
- Promote high-value cases to the golden set or regression suite.

The mining process is the engineering work. Without it, the feedback accumulates without producing action.

---

## 5. Implicit signal instrumentation

User behavior signals that imply quality without explicit feedback.

### 5.1 The implicit signals

- **Retry.** User asked the same or near-same question again within a short window. Implies the first answer was insufficient.
- **Edit.** For drafting workflows, the user edited the AI's draft before using it. The amount of edit is a quality signal.
- **Abandonment.** User started but did not complete the interaction. May indicate frustration with the AI.
- **Escalation.** For workflows with a human-escalation path, the rate of escalation is a near-direct quality signal.
- **Session length / depth.** Long sessions may indicate the user is struggling; short sessions may indicate quick resolution.

### 5.2 The signal definition

Each implicit signal needs a definition:

- **Retry:** same user, same or > 80% similar question, within 5 minutes of the previous response.
- **Edit:** > 20% of the draft text was changed before sending.
- **Abandonment:** session ended within 10 seconds of the response, no follow-up action.
- **Escalation:** user clicked the "escalate to clinician" button OR the agent invoked the escalate-to-human tool.

The definitions are workload-specific. The Meridian definitions above are illustrative.

### 5.3 The instrumentation

Implicit signals require instrumentation in the application layer (not the AI layer). The application:

- Tracks user actions per session (retry detection requires comparing across responses).
- Computes implicit signals per session.
- Emits to the observability stack as session-level events.

The signals link back to the AI interactions via session ID.

### 5.4 The interpretation challenges

Implicit signals are ambiguous:

- A retry may mean "the answer was wrong" or "the user remembered an additional detail."
- An edit may mean "the AI was wrong" or "the user wanted to customize the tone."
- Abandonment may mean "I'm frustrated" or "the phone rang."

The interpretation requires context. The team treats implicit signals as *hypotheses* — a high retry rate suggests a quality issue worth investigating, but does not prove one.

### 5.5 The aggregate metrics

Implicit signals are aggregated per feature:

- **Retry rate** per feature per day.
- **Edit rate** for drafting features.
- **Abandonment rate** per feature.
- **Escalation rate** per feature.

Trends are watched. Spikes correlate with potential quality issues; investigations follow.

---

## 6. The feedback loop into the eval suite

The discipline that turns production signals into eval cases.

### 6.1 The weekly triage

Weekly, the eval-team-owner reviews:

- All thumbs-down responses with free-text from the past week.
- Implicit signal spikes (retry-rate-elevated periods).
- Online judge failures (cases the judge flagged).

Per case, the triage decision:

- **Promote to golden set.** This is a representative case the eval should cover; add as a golden-set case.
- **Promote to regression suite.** This is a known failure to prevent recurrence; add as a regression case per [regression-eval-suites.md](./regression-eval-suites.md).
- **Note but do not promote.** The case is informational but does not warrant an eval slot (low value, near-duplicate, edge case the team accepts).
- **Investigate further.** The case suggests a deeper issue; spawn an investigation ticket.

### 6.2 The growth metric

A healthy feedback loop produces a measurable case-growth rate:

- ~2-5 new golden-set cases per week from feedback (mature stage).
- ~1-3 new regression cases per week from production bugs.

Growth rates below this indicate the feedback loop is broken (cases are being missed) or the production system is so good that few cases warrant eval coverage (unusual).

### 6.3 The case-mining tools

Tooling supports the triage:

- A queue interface showing thumbs-down + free-text from the past week.
- One-click "promote to golden set" or "promote to regression suite" with template population.
- Linkage from production trace to the new eval case.

Without tools, the triage is too slow; the team falls behind.

### 6.4 The feedback-to-fix latency

A useful metric: how long between user-reported quality issue and a case being added to the eval suite?

- Same-day: ideal for high-stakes issues.
- Within a week: standard.
- Beyond a week: feedback queue is backing up.

Latency tracked; alerts on queue growth.

### 6.5 The aggregate-feedback dashboard

Beyond per-case mining, aggregate feedback patterns inform broader strategy:

- "Users are reporting issues with X question class" → focus eval expansion in that class.
- "Drafting features have higher edit rates than chat" → focus product investment.
- "One tenant has consistently lower thumbs-up rate" → customer-success conversation.

The aggregate view supports strategic decisions beyond individual case-by-case work.

---

## 7. Production quality SLI

The continuously-updated quality metric. The foundation for quality-regression alerting.

### 7.1 The SLI definition

The production quality SLI is the online judge's pass rate on sampled production interactions, in a rolling window:

- **Window:** typically 4-hour rolling.
- **Stratification:** per feature; per question class; per tenant tier.
- **Threshold:** historical baseline ± tolerance.

For Meridian Care Coordinator: judge-pass-rate on the 10%-sampled production stream, 4-hour rolling window, per case class.

### 7.2 The SLI thresholds

The threshold per case class:

| Case class | Baseline | Alert (Tier 2) | Page (Tier 1) |
|---|---|---|---|
| Clinical-protocol | 95% | drop > 5 points | drop > 8 points |
| Drug-interaction | 98% | drop > 3 points | drop > 5 points |
| Patient-education | 92% | drop > 7 points | drop > 10 points |

The thresholds are tighter for higher-stakes classes per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).

### 7.3 The SLI versus user feedback

The SLI (judge-based) and user feedback (thumbs / implicit) are complementary:

- SLI catches issues the judge can identify (factual errors, citation problems, format issues).
- User feedback catches issues the judge cannot (poor tone, missing context users wanted, expectations mismatch).

Both signals are tracked; both feed dashboards. When they diverge (SLI fine but user feedback declining; or vice versa), the divergence itself is a signal that needs investigation.

### 7.4 The SLI calibration

Per [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) section 5, the judge's calibration determines the SLI's trustworthiness:

- Judge agreement with human review > 85% → SLI is trustworthy.
- Agreement < 85% → SLI is informational only; do not gate on it.

Calibration drift events temporarily downgrade the SLI from gating to informational.

### 7.5 The SLI vs eval pass-rate

The offline eval pass rate (against the golden set) and the production SLI (against production sample) are different metrics:

- **Eval pass rate:** measures performance on the cases the team designed. Stable; gates releases.
- **Production SLI:** measures performance on what users actually ask. Continuous; reflects real-world drift.

Both matter. A team with high offline eval pass-rate but declining production SLI has cases not well-covered by the golden set.

---

## 8. Integration patterns

The online-eval-and-feedback pattern integrates with multiple components.

### 8.1 With the LLM-call wrapper

Per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md), the wrapper emits trace spans. The online judging pipeline subscribes to the trace stream and samples for judging.

### 8.2 With the alerting practice

Per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md), the production quality SLI is a Tier 1 alerting source. Threshold breaches page on-call with the runbook for quality investigation.

### 8.3 With the eval gate

Per [eval-gate-architecture.md](./eval-gate-architecture.md), online eval is independent of the offline eval gate. The gate is offline; the SLI is online. They produce complementary signals.

### 8.4 With the regression suite

Per [regression-eval-suites.md](./regression-eval-suites.md) section 8.2, production failures detected by online eval feed the regression workflow.

### 8.5 With the golden set

Per [golden-set-design.md](./golden-set-design.md) section 7.2, online feedback (thumbs-down + free-text) is a growth channel for the golden set.

### 8.6 With the lineage system

Per [lineage-and-provenance.md](../../ai-architecture-reference-architecture/data-architecture-for-ai/lineage-and-provenance.md), the lineage supports the online judge's citation-accuracy and faithfulness scoring.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The online judging architecture

Meridian's online judging:

- 10% random sample + tail-based augmentation (fallback fires, circuit breakers, thumbs-down).
- Stratified to maintain proportional class coverage.
- Drug-interaction class sampled at 30%.
- Patient-education class sampled at 5% (low stakes, high volume).
- 3 sub-judges per sampled case (composite quality, citation accuracy, faithfulness).
- Daily cost: ~$15.

The judging runs continuously; verdicts feed the quality SLI dashboard.

### 9.2 The user feedback collection

Care Coordinator chat panel includes:
- Thumbs up / down per response.
- Optional free-text on thumbs-down ("What could be more helpful?").
- Optional category selection (factual accuracy / completeness / appropriate disclaimers / clinical relevance).

Response rate: ~3.5% of interactions receive thumbs (thumbs-up: 2.8%; thumbs-down: 0.7%). Free-text rate on thumbs-down: ~30%.

### 9.3 The implicit signals

Tracked per session:
- Retry rate (same clinician, similar question within 5 minutes): currently ~4% of interactions.
- Edit rate (for drafts the clinician modifies before sending): ~22% of drafts edited.
- Abandonment rate (session ended within 10s of response, no follow-up): ~6%.
- Escalation rate (clinician used escalate-to-human button): ~1.2%.

Baselines are tracked; spikes trigger investigation.

### 9.4 The feedback loop

Weekly: clinical-knowledge-engineering reviews:
- All thumbs-down + free-text from the past week.
- Online judge failures from the past week.
- Implicit signal spikes.

Recent triage (week of 2026-05-19):
- 47 thumbs-down responses; 14 with free-text.
- Triage outcomes: 3 promoted to golden set; 4 promoted to regression suite; 5 informational; 2 investigation tickets.
- Online judge failures: 12; 8 already covered by existing cases; 2 promoted as new regression cases; 2 spurious (judge error).

Weekly growth: ~6-8 new cases from the feedback loop.

### 9.5 The production quality SLI

The SLI dashboard:

- Per-feature judge-pass-rate (4-hour rolling, hourly samples).
- Per-class breakdown.
- Per-tenant tier breakdown.
- Trend lines (24-hour, 7-day, 30-day).

Recent SLI values:
- Clinical-protocol: 94.8% (baseline 95.2%; within tolerance).
- Drug-interaction: 98.4% (baseline 98.1%; healthy).
- Patient-education: 91.5% (baseline 92%; within tolerance).

Tier 1 alerts have fired twice in the past quarter (both real quality regressions, addressed via prompt rollback).

### 9.6 The 2026-Q2 feedback-loop story

In 2026-Q2, the feedback loop caught a pattern:

- Multiple thumbs-down responses related to handling pediatric patients (CHF in pediatric is rare but exists).
- The team's golden set had no pediatric-CHF cases.
- The clinical-knowledge worker was handling pediatric questions with the standard-adult protocol, producing incorrect dosing.
- Mining the feedback surfaced this gap.

The response:
- 5 pediatric-specific cases added to the golden set.
- Supervisor prompt updated to recognize pediatric context and dispatch differently.
- Eval-validated; deployed; subsequent pediatric questions handled correctly.

Without the feedback loop, the gap could have persisted for months until a more serious incident.

### 9.7 The platform discipline

- Online judging runs continuously at 10% sample.
- User feedback (thumbs + free-text) collected in the chat panel.
- Implicit signals instrumented and tracked.
- Weekly triage; promotion workflow.
- SLI is a real SLI (alerts, dashboards, runbooks).
- Quarterly review of feedback loop effectiveness (case-growth rate, latency, divergence between SLI and user feedback).

---

## 10. Anti-patterns

### 10.1 "Offline eval only"

The team has a thorough offline eval but no production-side signal. Quality regressions in production are discovered by user complaints.

**Corrective.** Online judging per section 2; SLI per section 7.

### 10.2 "User feedback not collected"

The product has no thumbs / rating UI. The team has no signal on user-perceived quality beyond support tickets.

**Corrective.** Feedback collection per section 4; even basic thumbs-up/down provides meaningful signal.

### 10.3 "Feedback collected but not triaged"

Thumbs-down feedback accumulates in a database; nobody reviews it. Cases that should be in the eval suite are missed.

**Corrective.** Weekly triage per section 6.1; promotion workflow.

### 10.4 "Implicit signals uninstrumented"

The team relies on explicit feedback only. Most users do not provide explicit feedback; the implicit signals (retry, edit, abandonment) are richer but missed.

**Corrective.** Implicit signal instrumentation per section 5.

### 10.5 "100% sampling"

The team samples 100% of production for judging. Cost is multi-hundreds-of-dollars per day for high-volume features; the team eventually backs off.

**Corrective.** Calibrated sampling per section 3.1; stratification and tail-based augmentation produce signal at lower cost.

### 10.6 "SLI is informational"

The judge-pass-rate is on a dashboard but not alerted. Quality regressions visible to the team go unnoticed.

**Corrective.** SLI with thresholds and alerts per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).

### 10.7 "Aggregate-feedback dashboard absent"

Per-tenant or per-class feedback patterns are not visible. The team cannot tell which segments are experiencing worse quality.

**Corrective.** Aggregate dashboards per section 4.4.

### 10.8 "Online judge uncalibrated"

The online judge runs in reference-free mode but has never been calibrated against human review. The SLI's correlation with reality is unknown.

**Corrective.** Calibration discipline per [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) section 5; reference-free judges are noisier; calibrate to that mode.

---

## 11. Findings (sprint-assignable)

### ONLINE-001 — Severity: Critical
**Finding.** No online judging; production quality is unmeasured continuously.
**Recommendation.** Implement online judging per section 2; sampled production runs.
**Owner.** ai-platform-eng + observability-eng, sprint N+1.

### ONLINE-002 — Severity: Critical
**Finding.** No user feedback collection; user-perceived quality is invisible.
**Recommendation.** Thumbs / rating UI per section 4; basic collection ships fast.
**Owner.** ai-platform-eng + product, sprint N+1.

### ONLINE-003 — Severity: Critical
**Finding.** Production quality SLI is not surfaced; quality regressions are detected by user complaints.
**Recommendation.** Judge-pass-rate as SLI per section 7; integrate with alerting.
**Owner.** ai-platform-eng + sre, sprint N+2.

### ONLINE-004 — Severity: High
**Finding.** Feedback is collected but not triaged; cases that should enter the eval suite are missed.
**Recommendation.** Weekly triage per section 6.1; promotion workflow.
**Owner.** ai-platform-eng + clinical-knowledge-eng, sprint N+2.

### ONLINE-005 — Severity: High
**Finding.** Implicit signals (retry, edit, abandonment, escalation) are uninstrumented.
**Recommendation.** Instrument per section 5.3.
**Owner.** ai-platform-eng + product, sprint N+2.

### ONLINE-006 — Severity: High
**Finding.** Online judging samples at 100% (or uncalibrated rate); cost is unsustainable.
**Recommendation.** Calibrated sampling per section 3.1; stratification and tail-based augmentation.
**Owner.** ai-platform-eng + finops, sprint N+2.

### ONLINE-007 — Severity: High
**Finding.** Tail-based augmentation is absent; rare cases (failures, fallbacks, circuit-breaker fires) are missed by random sampling.
**Recommendation.** Tail-based criteria per section 3.3.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ONLINE-008 — Severity: High
**Finding.** Online judge is uncalibrated in reference-free mode; SLI correlation with reality is unknown.
**Recommendation.** Calibration discipline per section 7.4; periodic re-validation.
**Owner.** ai-platform-eng + clinical-knowledge-eng, sprint N+3.

### ONLINE-009 — Severity: Medium
**Finding.** Free-text mining is informal; high-value cases are missed by reviewer fatigue.
**Recommendation.** Tools per section 6.3; queue interface; one-click promote.
**Owner.** ai-platform-eng, sprint N+3.

### ONLINE-010 — Severity: Medium
**Finding.** Aggregate-feedback dashboards are absent; per-tenant or per-class patterns invisible.
**Recommendation.** Dashboards per section 4.4 and 5.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ONLINE-011 — Severity: Medium
**Finding.** Per-class SLI is not separately tracked; per-class quality regressions hide in aggregate.
**Recommendation.** Per-class stratified SLI per section 3.2 and 7.1.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ONLINE-012 — Severity: Medium
**Finding.** Feedback-to-eval-case latency is not tracked; queue can back up unnoticed.
**Recommendation.** Latency metric per section 6.4.
**Owner.** ai-platform-eng team lead, sprint N+4.

### ONLINE-013 — Severity: Medium
**Finding.** Divergence between SLI and user feedback is not investigated.
**Recommendation.** Quarterly review of divergence; investigate when SLI and feedback disagree.
**Owner.** ai-platform-eng + product, sprint N+4.

### ONLINE-014 — Severity: Medium
**Finding.** Implicit signal definitions are informal; "retry" and "edit" have inconsistent thresholds across the team.
**Recommendation.** Document definitions per section 5.2.
**Owner.** ai-platform-eng + product, sprint N+4.

### ONLINE-015 — Severity: Medium
**Finding.** Online judge cost is not tracked separately; eval-cost line in FinOps is hidden.
**Recommendation.** Per-feature cost attribution; online-judging as its own line.
**Owner.** ai-platform-eng + finops, sprint N+4.

### ONLINE-016 — Severity: Low
**Finding.** Per-tenant sampling rate is uniform; high-value tenants get no extra signal.
**Recommendation.** Per-tier sampling per section 3.5.
**Owner.** ai-platform-eng + customer-success, sprint N+5.

### ONLINE-017 — Severity: Low
**Finding.** Judge call rate limit is not configured; traffic spikes can produce cost-incident-from-judging.
**Recommendation.** Rate limit per section 3.4.
**Owner.** ai-platform-eng + finops, sprint N+5.

### ONLINE-018 — Severity: Low
**Finding.** Categorical feedback (multiple-choice quality dimensions) is not collected; only binary thumbs.
**Recommendation.** Add categorical options per section 4.1.
**Owner.** ai-platform-eng + product, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting from offline-eval-only:

- [ ] **Sprint 0 — design.** Decide online judge architecture, sampling rate, feedback UI.
- [ ] **Sprint 1 — judge runs.** Online judging at low sample rate; verdicts to dashboard (not yet alerting).
- [ ] **Sprint 1 — feedback collection.** Thumbs UI in the product; storage alongside traces.
- [ ] **Sprint 2 — implicit signals.** Instrument retry, edit, abandonment, escalation.
- [ ] **Sprint 2 — sampling calibration.** Stratification; tail-based augmentation; rate limit.
- [ ] **Sprint 3 — feedback triage.** Weekly review; promotion workflow; tooling for one-click promote.
- [ ] **Sprint 3 — SLI.** Judge-pass-rate as SLI; thresholds; alerts integrate with on-call.
- [ ] **Sprint 4 — dashboards.** Per-class SLI; aggregate-feedback dashboards.
- [ ] **Sprint 4 — calibration.** Online judge calibration discipline; periodic re-validation.
- [ ] **Sprint 5 — refinement.** Per-tenant sampling; latency metrics; divergence reviews.
- [ ] **Ongoing — discipline.** Weekly triage; quarterly calibration; case-growth tracking.

A team that completes this sequence has the closed-loop quality discipline that turns production into a source of continuous eval improvement. A team that skips it accumulates production-only knowledge that never makes it back into the eval suite.

---

## 13. References

- This repo: [eval-engineering/eval-engineering-playbook.md](./eval-engineering-playbook.md) — the broader practice (section 7).
- This repo: [eval-engineering/golden-set-design.md](./golden-set-design.md) — growth target.
- This repo: [eval-engineering/llm-as-judge-patterns.md](./llm-as-judge-patterns.md) — the judge that scores online.
- This repo: [eval-engineering/regression-eval-suites.md](./regression-eval-suites.md) — regression-case capture.
- This repo: [eval-engineering/eval-gate-architecture.md](./eval-gate-architecture.md) — offline gate; complement to online SLI.
- This repo: [observability-and-telemetry/trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) — the trace stream the online judge subscribes to.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alerts on the SLI.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — the wrapper that produces trace spans.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
