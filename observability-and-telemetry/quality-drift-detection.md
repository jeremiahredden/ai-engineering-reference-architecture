# Quality Drift Detection

> **Audience.** Engineers and SREs operating AI features in production. Tech leads who have asked "is the model getting worse, or is this just variance?" **Scope.** The *engineering* practice of detecting AI quality drift in production — statistical drift detection, SLI thresholds, alert calibration, the broader drift discipline beyond per-interaction online judging. Pair with [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md), [llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md), [alerting-and-paging-design.md](./alerting-and-paging-design.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI systems drift. The same prompt with the same model on the same workload can produce slightly different output quality over weeks and months. The drift sources are many: provider-side model updates, embedding-model version shifts, corpus content changes, workload-composition shifts, judge calibration drift. Without detection, drift accumulates silently until a user-noticed regression forces investigation.

[online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) introduces the production quality SLI — judge-pass-rate on sampled production traffic. This document goes deeper on the *drift detection* discipline: how to statistically distinguish noise from signal, how to set thresholds that catch real drift without false alarms, how to integrate drift detection with the broader alerting hierarchy.

This document is opinionated about three things:

1. **Drift is statistical, not anecdotal.** A single failed interaction isn't drift; sustained baseline shifts are. The discipline distinguishes the two.
2. **Multiple drift signals are tracked.** Quality SLI is one; per-class breakdowns, latency distributions, retrieval recall trends, cost trajectories, feedback-rate shifts are all drift indicators. Cross-signal correlation is the diagnostic.
3. **Drift detection is itself eval-validated.** A drift detector that fires constantly is noise; one that never fires misses regressions. Calibrate against historical incidents.

Structure: (2) the drift signal taxonomy; (3) statistical detection methods; (4) per-class drift; (5) cross-signal correlation; (6) alert integration; (7) drift investigation workflow; (8) detection calibration; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The drift signal taxonomy

The signals that indicate quality drift.

### 2.1 The primary signals

- **Quality SLI** (per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 7). Judge-pass-rate on sampled production traffic. The direct measurement.
- **User feedback rate** — thumbs-down-to-thumbs-up ratio. Indirect but real.
- **Implicit signals** — retry rate, edit rate, abandonment rate, escalation rate. Per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 5.
- **Per-case-class pass rate** — class-specific quality.
- **Citation accuracy** — for RAG features.
- **Refusal rate** — when refusing vs answering.

### 2.2 The supporting signals

- **Retrieval recall on production sample** — for RAG; per [retrieval-observability.md](../rag-engineering/retrieval-observability.md) section 2.2.
- **Empty-retrieval rate.**
- **Per-tenant quality SLI** — some tenants may drift faster than others.
- **Cost-per-interaction** — drift up may indicate model regression.
- **Latency p95** — drift up may indicate complexity changes.

### 2.3 The signal correlation

Drift signals correlate:

- Quality SLI drops + user feedback drops + escalation rate up → strong drift signal.
- Quality SLI drops alone (no feedback shift) → may be judge calibration drift.
- Feedback drops without SLI shift → users perceive quality differently than judge measures.

The correlation pattern informs the diagnosis.

### 2.4 The signal observation horizon

Different signals have different observation horizons:

- **Quality SLI**: hourly aggregation; meaningful trends over 4-hour to 7-day windows.
- **Feedback rate**: daily aggregation; meaningful trends over 7-day to 30-day windows.
- **Per-class quality**: per-sample frequency depends on class share; longer windows for rare classes.

Each signal is plotted at its appropriate horizon; the dashboard accommodates.

### 2.5 The baseline definition

Each signal has a baseline:

- **Rolling baseline** (e.g., trailing 7-day mean). Used for short-term drift detection.
- **Stable baseline** (e.g., trailing 30-day mean). Used for medium-term drift detection.
- **Reference baseline** (the value at a specific known-good point in time). Used for long-term drift detection.

Each baseline serves a different drift-detection purpose.

---

## 3. Statistical detection methods

The methods that distinguish real drift from noise.

### 3.1 The simple-threshold method

Compare current value to baseline; alert if difference exceeds threshold.

```
If current_value < baseline - delta:
    alert
```

**When right.** Simple SLIs with stable variance. Coarse but easy to operate.

**What it misses.** Slow drift (small per-window changes accumulate over time); variance-aware patterns (high-variance metrics produce false alarms at fixed thresholds).

### 3.2 The standard-deviation method

Compare current value to baseline normalized by historical variance.

```
If (baseline - current_value) / std_dev_baseline > sigma_threshold:
    alert
```

Typical sigma_threshold: 2-3 (catches 95-99.7% of normal variation).

**When right.** SLIs with stable variance; baseline is reliable.

**What it costs.** Requires sufficient historical data to estimate variance.

### 3.3 The Cumulative Sum (CUSUM) method

Track cumulative deviations from baseline; alert when they exceed a threshold.

```
S_t = max(0, S_{t-1} + (baseline - current_value) - drift_allowance)
If S_t > h: alert
```

**When right.** Detecting slow drift that simple thresholds miss; sensitive to sustained shifts.

**What it costs.** Tuning two parameters (drift_allowance, threshold); harder to explain to non-experts.

### 3.4 The Page-Hinkley test

A change-point detection method that compares running mean to a reference.

**When right.** When the workload has clear baselines and changes are step-function shifts.

**What it costs.** Implementation complexity; tuning.

### 3.5 The week-over-week comparison

Compare this week's window to the same window last week (same-hour-of-week).

```
If this_week_value < last_week_value - delta:
    alert
```

**When right.** Seasonal workloads (clinical operating hours, weekday vs weekend patterns).

**What it costs.** Requires sufficient historical data; doesn't catch first-week regressions.

### 3.6 The method selection

| Workload | Recommended method |
|---|---|
| New (< 4 weeks data) | Simple threshold against initial baseline |
| Mature, stable variance | Standard deviation method |
| Mature, expecting slow drift | CUSUM |
| Seasonal | Week-over-week |
| Multi-method ensemble | Page-Hinkley + simple threshold |

Most teams combine 2-3 methods; each catches different patterns.

---

## 4. Per-class drift

Aggregate drift may hide per-class drift.

### 4.1 The aggregate-hides-class problem

Workload SLI:

- Day 1: clinical = 95%, general = 92%, weighted aggregate = 94%.
- Day 30: clinical = 91% (drift down 4 pts), general = 95% (drift up 3 pts), weighted aggregate = 93.5% (drift down 0.5 pts).

Aggregate looks fine; clinical class is degrading; the general class shift masks it. The team learns about the clinical issue from user complaints.

### 4.2 The per-class detection

Per-class drift detection: each class has its own baseline and threshold.

For Meridian:
- clinical: baseline 95%; alert if drops > 4 pts.
- drug-interaction: baseline 98%; alert if drops > 2 pts.
- conversational: baseline 91%; alert if drops > 5 pts.

Each class is monitored independently.

### 4.3 The class-share weighting

For aggregate alerting:

- Weight per class by traffic share.
- A class with 5% traffic share moving 10 points has less impact than a class with 60% share moving 2 points.

The weighting prevents over-alerting on rare classes; per-class alerts handle them separately.

### 4.4 The class-stratification of samples

For online judge sampling per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 3.2:

- Stratified sampling ensures each class has enough judged samples.
- Per-class SLI is statistically meaningful at appropriate sample size.

Without stratification, rare classes have too few samples for statistical detection.

### 4.5 The new-class identification

When a new question class emerges (workload composition shift):

- Per-class signals may not yet exist for it.
- The team detects the new pattern; defines a class; instruments per-class signal.

This is part of the normal workload evolution.

---

## 5. Cross-signal correlation

Drift is more confidently identified when multiple signals agree.

### 5.1 The confidence levels

| Signals agreeing | Confidence |
|---|---|
| 1 (e.g., SLI drops alone) | Suggestive; investigate |
| 2 (e.g., SLI drops + feedback drops) | Likely real drift |
| 3+ (SLI + feedback + escalation) | High confidence; respond |

The cross-signal pattern informs both detection (more confident multi-signal alerts) and investigation (which signals point at the same cause).

### 5.2 The signal patterns

Common patterns:

- **Real quality regression.** SLI drops + user feedback drops + escalation rate up + retry rate up. All signals align.
- **Judge calibration drift.** SLI drops without user feedback shift. Judge is mismeasuring, not the system.
- **Workload composition shift.** New class appears; aggregate SLI shifts; per-class SLI on existing classes is stable.
- **Provider issue.** Latency up + cost up + occasional errors; SLI relatively stable.
- **Corpus drift.** Per-RAG-class SLI drops (specifically); retrieval recall drops; embedding model unchanged.

Each pattern has a different root cause; the cross-signal pattern is the diagnostic.

### 5.3 The dashboard layout

The drift dashboard juxtaposes signals:

- Quality SLI panel.
- User feedback panel (thumbs ratio, escalation rate).
- Implicit signals panel (retry, edit, abandonment).
- Per-class breakdowns.
- Supporting signals (latency, cost, retrieval-empty rate).

Drift investigators read the dashboard top-to-bottom; cross-signal correlation is visible.

### 5.4 The automated correlation

Some teams build automated correlation:

- Multi-signal alert: alert only when N signals drift simultaneously.
- Reduces false positives.
- Tuned per workload.

The Meridian Care Coordinator uses a 2-of-4 rule: alert when 2 of (SLI, feedback, escalation, retrieval-empty) drift past their individual thresholds.

### 5.5 The signal independence

For correlation to be meaningful, signals must be sufficiently independent:

- SLI from online judge: dependent on judge quality.
- User feedback: dependent on user willingness to provide feedback.
- Implicit signals: dependent on UI behavior.

If all signals share a common bias (e.g., the judge is biased and the UI nudges users toward thumbs-up), correlation may be misleading.

The independence is checked periodically per section 8.4.

---

## 6. Alert integration

Drift detection feeds the broader alerting hierarchy.

### 6.1 The alert tiers

Per [alerting-and-paging-design.md](./alerting-and-paging-design.md):

- **Tier 1 (page).** Multi-signal drift confirmed; SLI breach exceeds Sev-1 threshold; immediate response.
- **Tier 2 (alert).** Single signal drift; SLI breach exceeds warning threshold; next-day investigation.
- **Tier 3 (dashboard).** Trend visualization; not actionable in isolation.

### 6.2 The threshold-to-alert mapping

Each drift detector maps to a specific alert:

| Detector | Threshold | Alert Tier | Runbook |
|---|---|---|---|
| SLI std-dev | > 3 sigma drop, sustained 4h | Tier 1 | quality-regression-runbook |
| SLI std-dev | 2-3 sigma drop, sustained 4h | Tier 2 | investigate-quality-drift |
| Per-class drug-interaction SLI | > 5 pts drop | Tier 1 | clinical-regression-runbook |
| Multi-signal correlation | 2-of-4 signals | Tier 1 | drift-investigation-runbook |
| Feedback rate shift | > 25% in 7 days | Tier 2 | feedback-drift-investigation |
| Retrieval empty-rate | > 2x baseline | Tier 1 | empty-retrieval-runbook |

The threshold + tier mapping is per workload; calibrated per section 8.

### 6.3 The runbook integration

Each alert has a runbook per [alerting-and-paging-design.md](./alerting-and-paging-design.md) section 5:

- Diagnostic steps for the specific drift class.
- Common causes (model update, corpus change, prompt change, etc.).
- Remediation patterns.

The runbook is what makes drift detection actionable.

### 6.4 The drift incident protocol

When drift is confirmed:

1. Acknowledge the alert.
2. Identify the affected feature / class / tenant.
3. Identify the suspected cause (cross-signal correlation, recent changes).
4. If a recent change is the cause: rollback per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) or prompt-versioning.
5. If no recent change: deeper investigation.
6. Post-incident: add regression case; calibrate detection if needed.

The protocol is structured; first responses are fast.

### 6.5 The post-mortem

After drift incidents (per [alerting-and-paging-design.md](./alerting-and-paging-design.md) section 6.3):

- Did drift detection fire at the right time?
- Was the runbook adequate?
- What instrumentation would have caught it earlier?

Post-mortem findings update detection calibration.

---

## 7. Drift investigation workflow

The structured workflow for investigating drift signals.

### 7.1 The intake

When drift alert fires:

1. Confirm the alert: is the signal really showing drift?
2. Acknowledge in the on-call system.
3. Open the drift dashboard.

### 7.2 The cross-signal review

On the dashboard:

- Is the drift visible on multiple signals or just one?
- Is the drift in aggregate or in specific classes?
- Is the drift in specific tenants or platform-wide?

The pattern informs the next step.

### 7.3 The recent-change correlation

Check what changed in the relevant time window:

- Model deploys (per [model-registry.md](../model-lifecycle/model-registry.md)).
- Prompt deploys (per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md)).
- Corpus updates (per [ingestion-pipeline-engineering.md](../rag-engineering/ingestion-pipeline-engineering.md)).
- Configuration changes (per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md)).
- Provider-side events (provider status pages).

If a recent change correlates with drift onset, it's the likely cause.

### 7.4 The fix

If a change is the cause:

- Rollback the change.
- Verify the drift resolves.
- Investigate the change's quality issue.

If no specific change:

- Investigate broader patterns (workload shift, provider drift, slow accumulation).
- May require eval analysis, sample-case review, deeper instrumentation.

### 7.5 The post-fix verification

After remediation:

- Watch the drift signals; verify they return to baseline.
- Run an eval validation against the fix.
- Document the incident.

### 7.6 The regression case

Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md):

- The drift incident produces a regression case.
- The case captures the affected pattern.
- The eval suite is updated to prevent recurrence.

---

## 8. Detection calibration

The drift detection is itself eval-validated.

### 8.1 The historical replay

Calibration uses historical data:

- Pull historical SLI trajectories.
- Mark known incidents (when did drift actually happen?).
- Run candidate detection methods against the history.
- Measure: did the detector fire at the right time? False positives? False negatives?

### 8.2 The detection-quality metrics

For a candidate detection configuration:

- **True positive rate.** Real drift incidents that fired the alert.
- **False positive rate.** Alert fires when no real drift was happening.
- **Detection latency.** Time from drift onset to alert.

Tune to maximize true positives at acceptable false-positive rate.

### 8.3 The recalibration cadence

Quarterly:

- Review the past quarter's alerts.
- Calculate detection quality on the period's incidents.
- Adjust thresholds or methods if quality has drifted.

### 8.4 The signal-independence check

Periodically verify drift signals remain independent:

- Cross-correlation analysis.
- Identify cases where signals move together for non-drift reasons.
- Adjust if signals become too correlated (consider new independent signals).

### 8.5 The drift-from-the-detector

The detector itself drifts:

- Judge calibration shifts; SLI changes.
- Sampling distribution shifts; per-class detection thresholds need recalibration.
- New question classes; existing class baselines may need re-derivation.

Recalibration catches detector drift.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The drift signals

Meridian's drift signal panel:

| Signal | Source | Threshold | Tier |
|---|---|---|---|
| Online judge SLI (clinical) | online-eval sampling | > 4 pts drop 4h | T2; > 8 pts T1 |
| Online judge SLI (drug-interaction) | online-eval sampling | > 2 pts drop 4h | T1 |
| User feedback (thumbs ratio) | UI | > 25% shift 7d | T2 |
| Escalation rate | clinical UI | > 50% above baseline 24h | T2 |
| Retrieval empty-rate | retrieval pipeline | > 2x baseline 1h | T1 |
| Multi-signal correlation | 2-of-4 above | (combined) | T1 if 2 trigger simultaneously |

Each signal is independently monitored; the correlation rule adds confidence.

### 9.2 The detection methods

- **SLI**: standard-deviation method against trailing 7-day baseline.
- **Feedback**: simple-threshold against trailing 7-day rolling.
- **Escalation**: simple-threshold against trailing 7-day baseline with seasonal correction (clinic-hour vs off-hour baselines).
- **Retrieval empty-rate**: simple-threshold against 7-day baseline.

The combination catches different drift patterns.

### 9.3 The dashboard

The drift dashboard:

- Top panel: aggregate SLI with 7-day trend.
- Per-class panels: clinical, drug-interaction, conversational, refusal.
- Feedback panel: thumbs-up/down trend.
- Implicit panel: retry, edit, abandonment, escalation trends.
- Supporting panels: latency, cost, retrieval-empty rate.

Reviewed weekly per [retrieval-observability.md](../rag-engineering/retrieval-observability.md) section 5.

### 9.4 The 2026-Q2 drift incident

Walk-through of an actual drift detection:

1. **Tier 2 alert fires**: SLI dropped 5 points over 4 hours.
2. **On-call acknowledges; opens dashboard.**
3. **Cross-signal check**: Feedback rate also down 30%; escalation rate up; retrieval empty-rate stable.
4. **Multi-signal correlation: 3-of-4 signals showing drift. High confidence; auto-promote to Tier 1.**
5. **Recent changes**: prompt version `supervisor@2.4.2` deployed 6 hours prior.
6. **Hypothesis: prompt change is the cause.**
7. **Rollback** to `supervisor@2.4.1`.
8. **Verify**: signals return to baseline within 4 hours.
9. **Post-incident**: regression case added; the new prompt's eval validation gap identified.
10. **Calibration**: 4-hour rolling window proved adequate; no detection-tuning needed.

The cycle from alert to fix: ~6 hours. Detection caught it; runbook accelerated; rollback was clean.

### 9.5 The judge calibration drift incident (per llm-as-judge-patterns.md)

Per [llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) section 9.7:

- Quarterly calibration showed judge agreement dropped from 89% to 79%.
- SLI had been gradually shifting; the team had attributed it to workload changes.
- Calibration drift was actually the cause; the judge was over-strict.

The detection: scheduled calibration caught what the day-to-day drift detectors missed.

### 9.6 The pediatric-CHF drift

Per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 9.6:

- New class (pediatric-CHF) emerged.
- Aggregate SLI stayed stable (the new class was small share).
- User feedback identified the issue.
- Per-class detection was inadequate (no class defined yet).

The lesson: new-class identification per section 4.5 is part of the discipline.

### 9.7 The platform discipline

- Multiple detection methods (statistical, threshold, correlation).
- Per-class signals tracked.
- Cross-signal correlation rule for high-confidence alerts.
- Quarterly recalibration.
- Calibration drift detection separate from day-to-day drift.

---

## 10. Anti-patterns

### 10.1 "Single signal monitoring"

Only the aggregate SLI is monitored; per-class drift hidden by aggregate.

**Corrective.** Per-class signals per section 4.

### 10.2 "Anecdotal drift response"

One user complaint triggers a drift investigation; investigation finds no statistical signal; team chases noise.

**Corrective.** Statistical detection per section 3; anecdotes inform feedback queue but don't trigger investigations.

### 10.3 "Drift detection without runbook"

Alert fires; on-call doesn't know what to do.

**Corrective.** Runbook per section 6.3.

### 10.4 "Cross-signal correlation skipped"

Single-signal alerts fire constantly (false positives); team learns to ignore.

**Corrective.** Multi-signal correlation per section 5.4.

### 10.5 "Threshold from intuition"

Thresholds set by guess; high false-positive rate or missed real incidents.

**Corrective.** Calibration against historical data per section 8.1.

### 10.6 "Judge calibration drift undetected"

SLI drops; team chases system causes; the real cause is the judge.

**Corrective.** Scheduled judge calibration per [llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) section 5.

### 10.7 "New-class blind spot"

Workload composition shifts; new class emerges; existing per-class detection doesn't cover; quality issues hide.

**Corrective.** New-class identification per section 4.5.

### 10.8 "Drift detection ignores seasonality"

Daytime-vs-nighttime, weekday-vs-weekend patterns trigger false positives; team disables drift detection.

**Corrective.** Week-over-week method per section 3.5 or seasonal baselines.

---

## 11. Findings (sprint-assignable)

### DRIFT-001 — Severity: Critical
**Finding.** No drift detection on production quality SLI; regressions discovered by users.
**Recommendation.** SLI threshold detection per section 3.2; alert integration.
**Owner.** ai-platform-eng + sre, sprint N+1.

### DRIFT-002 — Severity: Critical
**Finding.** Per-class drift detection absent; aggregate hides class-specific regressions.
**Recommendation.** Per-class detection per section 4.
**Owner.** ai-platform-eng + observability-eng, sprint N+1.

### DRIFT-003 — Severity: High
**Finding.** Drift alerts fire without runbooks.
**Recommendation.** Per-class runbooks per section 6.3.
**Owner.** ai-platform-eng + sre, sprint N+2.

### DRIFT-004 — Severity: High
**Finding.** Multi-signal correlation absent; single-signal alerts produce false positives.
**Recommendation.** Cross-signal correlation per section 5.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### DRIFT-005 — Severity: High
**Finding.** Drift thresholds set by intuition; high false-positive or false-negative rate.
**Recommendation.** Calibration against historical data per section 8.1.
**Owner.** ai-platform-eng, sprint N+2.

### DRIFT-006 — Severity: High
**Finding.** Judge calibration drift not detected; SLI shifts attributed to system causes.
**Recommendation.** Scheduled judge calibration per [llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md); separate from system drift detection.
**Owner.** ai-platform-eng, sprint N+2.

### DRIFT-007 — Severity: High
**Finding.** New-class blind spot; workload composition shifts surface quality issues without detection.
**Recommendation.** New-class identification per section 4.5; periodic class-review.
**Owner.** ai-platform-eng + product, sprint N+3.

### DRIFT-008 — Severity: High
**Finding.** Seasonal patterns produce false-positive drift alerts.
**Recommendation.** Week-over-week or seasonal-baseline methods per section 3.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### DRIFT-009 — Severity: Medium
**Finding.** Slow drift undetected; small per-window changes accumulate without alerting.
**Recommendation.** CUSUM or Page-Hinkley methods per section 3.3-3.4.
**Owner.** ai-platform-eng, sprint N+3.

### DRIFT-010 — Severity: Medium
**Finding.** User feedback as drift signal unused; only judge SLI tracked.
**Recommendation.** Feedback rate as a tracked signal per section 5.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### DRIFT-011 — Severity: Medium
**Finding.** Drift dashboard absent; signals scattered across multiple views.
**Recommendation.** Drift dashboard per section 9.3.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### DRIFT-012 — Severity: Medium
**Finding.** Signal independence not checked; correlated false-positives.
**Recommendation.** Periodic independence check per section 8.4.
**Owner.** ai-platform-eng, sprint N+4.

### DRIFT-013 — Severity: Medium
**Finding.** Drift detector recalibration not scheduled.
**Recommendation.** Quarterly recalibration per section 8.3.
**Owner.** ai-platform-eng team lead, sprint N+4.

### DRIFT-014 — Severity: Medium
**Finding.** Per-tenant drift not tracked; tenant-specific regressions hide in aggregate.
**Recommendation.** Per-tenant signals for premium tier; aggregate for standard tier with tenant outlier detection.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### DRIFT-015 — Severity: Medium
**Finding.** Drift incident post-mortems don't update detection calibration.
**Recommendation.** Calibration review included in post-mortem template per section 6.5.
**Owner.** ai-platform-eng + sre, sprint N+4.

### DRIFT-016 — Severity: Low
**Finding.** Detection latency not measured.
**Recommendation.** Track per section 8.2; aim for minutes-to-hours, not days.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### DRIFT-017 — Severity: Low
**Finding.** Drift detection documentation thin; new engineers don't understand the methods.
**Recommendation.** Documentation alongside the dashboard.
**Owner.** ai-platform-eng, sprint N+5.

### DRIFT-018 — Severity: Low
**Finding.** Multi-method ensemble not used; single method may have blind spots.
**Recommendation.** Combine methods (e.g., simple threshold + CUSUM) per section 3.6.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team adding drift detection:

- [ ] **Sprint 0 — design.** Choose primary detection methods; identify per-class signals.
- [ ] **Sprint 1 — SLI threshold detection.** Simple-threshold against baseline; per-class.
- [ ] **Sprint 1 — alert integration.** Tier 1 / Tier 2 mapping; runbooks.
- [ ] **Sprint 2 — multi-signal correlation.** 2-of-N rule for high-confidence alerts.
- [ ] **Sprint 2 — drift dashboard.** Unified view of all drift signals.
- [ ] **Sprint 3 — calibration.** Run historical replay; tune thresholds.
- [ ] **Sprint 3 — seasonality handling.** Week-over-week or seasonal baselines if needed.
- [ ] **Sprint 4 — advanced methods.** CUSUM for slow drift; Page-Hinkley for change-points.
- [ ] **Sprint 4 — per-tenant signals.** For multi-tenant systems.
- [ ] **Sprint 5 — recalibration cadence.** Quarterly review; signal-independence check.
- [ ] **Ongoing — discipline.** Drift incident post-mortems; detection calibration evolution.

A team that completes this sequence catches drift before users do. A team that doesn't has drift as an emergency-only discipline.

---

## 13. References

- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — trace structure.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — call instrumentation.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](./alerting-and-paging-design.md) — alert hierarchy.
- This repo: [observability-and-telemetry/cost-dashboards.md](./cost-dashboards.md) — cost trend dashboards.
- This repo: [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — quality SLI source.
- This repo: [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) — judge calibration.
- This repo: [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — regression case workflow.
- This repo: [rag-engineering/retrieval-observability.md](../rag-engineering/retrieval-observability.md) — retrieval-specific drift signals.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](../rag-engineering/rag-failure-modes-and-debugging.md) — drift investigation overlaps.
- SPC (statistical process control) literature for CUSUM and Page-Hinkley.
- Page-Hinkley test: Page, E. S. (1954). *Continuous Inspection Schemes*.
- Google SRE Book chapter on monitoring for SLI / SLO discipline.
