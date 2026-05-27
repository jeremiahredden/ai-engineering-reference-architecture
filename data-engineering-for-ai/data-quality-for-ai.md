# Data Quality for AI

> **Audience.** Engineers whose AI model quality is mysteriously degrading and they suspect (correctly) that the data has shifted. Tech leads whose retrieval corpus has accumulated duplicates and stale content. Anyone whose eval suite is producing inconsistent results between runs. **Scope.** The *engineering* practice of data quality for AI: label noise quantification; distribution-drift detection; contamination detection (training data leaked into eval); deduplication; near-duplicate handling; data-quality dashboard. Not the labeling discipline (see [labeling-and-annotation.md](./labeling-and-annotation.md), companion). Not the dataset versioning (see [dataset-versioning.md](./dataset-versioning.md), companion). Not the synthetic data generation (see [synthetic-data-generation.md](./synthetic-data-generation.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI quality problems trace to data more often than to models. The model is mostly fine; the data is silently degrading:

- Label noise: some labels are wrong; model learns the noise.
- Distribution drift: the data the model trained on no longer matches production.
- Contamination: eval cases ended up in training; eval is meaningless.
- Duplicates: same content appears multiple times; signal diluted.

This document covers the engineering practice of detecting and managing each.

The general data-quality canon (great expectations, dbt tests, etc.) applies to AI but with AI-specific overlays:

- Label noise is unique to labeled-data workloads.
- Distribution drift detection for AI needs embedding-based methods.
- Contamination is specifically about eval cases leaking.
- Deduplication needs near-duplicate detection (semantic), not just exact-match.

This document is opinionated about four things:

1. **Data quality is engineering's problem.** Not data team's; not "data scientists' notebook problem." Engineering hygiene.
2. **Label noise is real.** Even good annotators have 1-5% noise; track it.
3. **Eval contamination is the silent killer.** Test for it.
4. **Deduplication needs near-duplicate awareness.** Exact-match isn't enough.

Structure: (2) label noise quantification; (3) distribution drift detection; (4) contamination detection; (5) deduplication and near-duplicate handling; (6) data-quality dashboard; (7) the response when quality drops; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. Label noise quantification

How much noise is in the labels.

### 2.1 The noise model

Some fraction of labels are wrong:

- Annotators make mistakes.
- Rubric ambiguity causes inconsistency.
- Edge cases are arbitrary.

Even good labeling: 1-5% noise.

### 2.2 The noise detection methods

**Method A: SME audit.** Sample cases; SME labels independently; compare to existing labels.

- Coverage: 100-500 cases.
- Output: noise rate estimate.

**Method B: Inter-annotator agreement.** Two annotators on same cases; disagreement rate = noise estimate.

- Cross-link to [labeling-and-annotation.md §4](./labeling-and-annotation.md).

**Method C: Model-vs-label disagreement.** Model trained on the data; cases where model strongly disagrees with label are candidates for noisy.

- More indirect; useful at scale.

**Method D: Snorkel / weak supervision.** Multiple labeling functions vote; disagreement signals noise.

- For specific workflows.

### 2.3 The "what noise rate is acceptable" decision

Per workload:

- Clinical labeling: <1% noise required.
- Customer support classification: 5-10% noise acceptable.
- Casual sentiment: 10-15% noise acceptable.

Higher-stakes = lower tolerance.

### 2.4 The "we have noise; what do we do" response

If noise > threshold:

- Re-review affected cases (or sample).
- Re-label.
- Diagnose source (rubric? annotator?).
- Address root cause.

Cross-link to [labeling-and-annotation.md §3](./labeling-and-annotation.md).

### 2.5 The noise tracking

Per dataset version:

- Noise rate estimate.
- Tracked over time.
- Alert on drift.

### 2.6 The "noise in eval set" sensitivity

For eval sets, noise is especially bad:

- Eval is the ground truth.
- 10% noise = 10% of "ground truth" is wrong.
- Model that does great on real cases may be marked "failing" by bad labels.

Eval golden sets get extra scrutiny.

### 2.7 The active-learning use of noise

For labels with detected noise:

- Re-label those.
- Or flag for SME review.

Iterative quality improvement.

### 2.8 The noise-vs-cost trade-off

Lower noise = more expensive (more SME review, calibration, etc.).

Per-workload: how much noise is worth tolerating.

### 2.9 The cross-annotator noise distribution

If 5% noise:

- May be 5% across all annotators (uniform).
- Or 20% from one annotator + 1% from others.

Per-annotator noise tracking helps.

---

## 3. Distribution drift detection

When the data shape changes.

### 3.1 The drift problem

Production data shape shifts:

- New types of inputs appear.
- Vocabulary evolves.
- User behavior changes.

Model trained on old shape may degrade on new.

### 3.2 The drift signals

**Signal A: Input feature distribution shift.**

- Length of inputs increased.
- Vocabulary distribution shifted.
- Class distribution different.

**Signal B: Embedding distribution shift.**

- Production query embeddings differ from training embeddings.
- Measured via KL divergence, Maximum Mean Discrepancy, etc.

**Signal C: Performance degradation.**

- Model accuracy dropping.
- May or may not be drift; investigation needed.

### 3.3 The drift detection method

```python
def detect_drift(reference_embeddings, current_embeddings, threshold=0.1):
    # Compute distribution similarity.
    distance = embedding_distribution_distance(reference_embeddings, current_embeddings)
    return distance > threshold
```

Periodic check.

### 3.4 The "we have drift; what to do" response

Options:

- Re-train (with current data).
- Add current data to training set; re-train.
- Investigate (is drift driven by abuse / new use cases?).

### 3.5 The drift dashboard

Per data source:

- Embedding-distribution distance over time.
- Vocabulary stats.
- Length stats.

Visualized; reviewed.

### 3.6 The "drift is normal" vs "drift is concerning"

Some drift is expected:

- Seasonal patterns.
- Growing user base.

Some drift is concerning:

- Sudden shift (likely abuse or feature change).
- Sustained trend (workload shift).

Diagnose.

### 3.7 The drift-alert tuning

- False positives: too sensitive; ignore.
- False negatives: not sensitive; miss real drift.

Tune; per-workload.

### 3.8 The retrieval-corpus drift

For retrieval:

- Corpus drifts (documents added, removed).
- Embedding distribution drift means retrieval quality may shift.

Cross-link to [retrieval-corpus-engineering.md](./retrieval-corpus-engineering.md).

### 3.9 The "model is fine; drift is in eval" surprise

Sometimes eval shows degradation but production is fine:

- Eval doesn't reflect production distribution.
- Eval needs updating, not the model.

Diagnose carefully.

---

## 4. Contamination detection

Training data leaking into eval.

### 4.1 The contamination problem

If training data and eval data overlap:

- Eval is testing on data the model has seen.
- Eval score is inflated.
- Production performance won't match.

Insidious; hard to detect.

### 4.2 The hash-based detection

For each eval case:

- Hash the content.
- Check if hash appears in training set.

Catches exact duplicates.

### 4.3 The semantic-similarity detection

For near-duplicates:

- Embed eval case.
- Compare to training-set embeddings.
- High similarity → potential contamination.

Threshold: similarity > 0.95.

### 4.4 The "we found contamination" recovery

Affected cases:

- Remove from training (re-train).
- Or remove from eval (eval is no longer trustworthy).
- Future: prevent recurrence (engineering controls).

### 4.5 The "eval cases inadvertently included" scenarios

How contamination happens:

- Web scrape included data sources that eval samples.
- Fine-tune data overlaps with eval set.
- Few-shot examples include eval cases.
- Customer-submitted data includes eval-style cases.

Each is a path; each can be prevented.

### 4.6 The engineered separation

Cross-link to [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md):

- Eval cases time-stamped after training cut-off.
- Eval cases from explicit "eval-only" sources.
- Training-data filter excludes known eval IDs.

Engineered prevention.

### 4.7 The periodic audit

Quarterly:

- Check training and eval sets for overlap.
- Hash + semantic similarity.
- Address findings.

Ongoing.

### 4.8 The "we never check" risk

Without checking:

- Contamination accumulates.
- Eval becomes meaningless.

Default belief: there is contamination unless verified.

### 4.9 The reverse contamination (eval → training)

Sometimes eval cases get added to training:

- Synthetic data generation includes eval cases.
- User-submitted similar cases.

Same problem; same detection.

---

## 5. Deduplication and near-duplicate handling

Cleaning duplicates.

### 5.1 The duplicate problem

Duplicates in:

- Training data: model over-weights repeated content.
- Eval set: cases counted multiple times; metric biased.
- Retrieval corpus: queries return multiple copies; user experience degraded.

### 5.2 The exact-match deduplication

For exact duplicates:

- Hash each item.
- Remove items with duplicate hash.

Simple; catches obvious.

### 5.3 The near-duplicate detection

For near-duplicates (paraphrases, minor edits):

- Embed each item.
- Cluster by similarity.
- Items with similarity > 0.95 are near-duplicates.

Cross-link to [retrieval-corpus-engineering.md §4](./retrieval-corpus-engineering.md).

### 5.4 The "keep one" policy

For near-duplicates:

- Keep highest-quality version.
- Or most-recent.
- Or canonical.

Per-workload policy.

### 5.5 The near-duplicate threshold tuning

Too tight (0.95): some near-dups missed.
Too loose (0.85): different content marked as duplicates.

Per-workload tuning.

### 5.6 The training-data deduplication

For training:

- Heavy deduplication preferred.
- Repeated content can hurt training.

Aggressive thresholds.

### 5.7 The retrieval-corpus deduplication

For retrieval:

- Aggressive: removes redundancy from queries.
- Less aggressive: preserve nuance.

Balance.

### 5.8 The "dedup removed legitimate distinct content" risk

Near-duplicate detection can be wrong:

- Two policy documents that are similar but distinct.
- Dedup might remove one.

Mitigations:

- Threshold tuning.
- Human review of dedup decisions.
- Conservative approach.

### 5.9 The cross-corpus dedup

For multi-source corpora:

- Documents from different sources may overlap.
- Dedup across sources.

### 5.10 The dedup pipeline integration

Dedup as part of:

- Initial corpus build.
- Ongoing updates.
- Pre-training pipeline.

Standard.

---

## 6. Data-quality dashboard

The visibility layer.

### 6.1 The dashboard panels

Per dataset:

- Label noise rate.
- Distribution shape over time.
- Drift signal.
- Dedup statistics.
- Contamination check status.

Visualized.

### 6.2 The aggregate quality metrics

Across all datasets:

- Total noise.
- Total drift.
- Total contamination events.

Trend.

### 6.3 The alert design

- Noise rate exceeds threshold → alert.
- Drift signal > threshold → alert.
- Contamination detected → alert.

Per dataset; per metric.

### 6.4 The quality SLO

Cross-link to [ai-engineering-reference-architecture / reliability-engineering / fault-budgets-for-ai.md](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/fault-budgets-for-ai.md):

- Per-dataset quality SLO (e.g., noise < 3%).
- Burn rate; alerts.

Treats data quality as a SLO.

### 6.5 The "we have a dashboard but never check" gap

Without engagement:

- Dashboard exists; nobody acts.

Monthly review per dataset.

### 6.6 The "we found drift; what now" workflow

- Investigate cause.
- Plan response (re-train? update eval?).
- Implement.
- Verify drift addressed.

Lifecycle.

### 6.7 The drilldown capability

When dashboard surfaces issue:

- Drill into specific cases.
- Sample inspection.
- SME review if needed.

Investigation tools.

### 6.8 The "we have many datasets; can't watch all" reality

For many datasets:

- Prioritize high-stakes.
- Sample monitoring of low-stakes.

Per importance.

---

## 7. The response when quality drops

What happens when quality issues surface.

### 7.1 The detection-to-response time

Goal:

- Critical quality drop (e.g., model in production failing): same-day response.
- Drift signal: within-week response.
- Routine noise check: monthly cycle.

Per severity.

### 7.2 The mitigation options

Per issue:

- Re-train.
- Re-label.
- Re-deduplicate.
- Update eval.
- Update training data.
- Or: accept and document.

Per-issue decision.

### 7.3 The incident-class quality issues

Major issues:

- Cross-link to [ai-engineering-reference-architecture / reliability-engineering / incident-response-for-ai.md §2.2](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/incident-response-for-ai.md).
- Quality incident class.

Same framework.

### 7.4 The cross-dataset impact

A quality issue in one dataset affects multiple things:

- Retrieval-corpus issue: affects retrieval and downstream LLM.
- Eval-set issue: affects all features tested by it.

Multi-system response.

### 7.5 The communication of quality issues

To stakeholders:

- "We detected data quality issue; investigating."
- "Resolution: ..."
- "Lessons learned."

Same as incident comms.

### 7.6 The "we caught it before production" preventive case

If quality issue caught pre-production:

- Don't ship.
- Address.
- Document.

Better.

### 7.7 The chronic-quality-issues pattern

Some datasets have chronic quality issues:

- Constant noise above target.
- Constant drift.

Diagnose: is the rubric / workflow / source flawed?

Root-cause; not tactical.

### 7.8 The post-incident review

For quality incidents:

- Same template as other incidents.
- Particular attention: did our quality monitoring catch this?
- Improve monitoring if not.

---

## 8. Worked Meridian example

Meridian's data quality discipline.

### 8.1 The quality monitoring per dataset

```
care-coordinator-eval-golden-set:
  noise: 0.4% (verified by SME audit)
  drift: stable
  dedup: cleaned
  contamination: none detected

document-classification-fine-tune-data:
  noise: 2.1% (vendor + SME review)
  drift: minor seasonal pattern
  dedup: 0.3% duplicates per quarter
  contamination: none detected

clinical-corpus:
  drift: ~5%/month new content (expected)
  dedup: 1.5% near-duplicates removed
  contamination: n/a (retrieval, not training)
```

Tracked.

### 8.2 The Q1 2026 noise audit

Quarterly noise audit on document-classification-fine-tune-data:

- 500 cases SME-reviewed.
- Findings: 2.1% noise overall.
- Per-annotator: ranged 0.5% to 4.5%.
- High-noise annotator identified; re-calibrated.

Quality maintained.

### 8.3 The Q1 2026 drift incident

Patient API chat embedding distribution shifted (~15% from baseline):

- Detected by embedding-distribution alert.
- Investigation: new patient demographic onboarded; different communication patterns.
- Decision: not concerning; just new users.
- Drift signal threshold tuned to account for this kind of normal shift.

### 8.4 The Q2 2026 contamination scare

Audit found 12 eval cases similar to training cases:

- Hash check: 0 exact matches.
- Semantic check: 12 cases with similarity > 0.95.
- Investigation: these were paraphrases; not exact duplicates.
- Decision: remove the 12 cases from eval (couldn't determine if model had memorized).
- Updated eval set.

Caught early; small impact.

### 8.5 The retrieval-corpus dedup

Q2 2026: clinical-corpus had 1.5% near-duplicates.

- Identified by semantic-similarity clustering.
- Sample review confirmed: paraphrased policies, slightly different versions.
- Kept canonical version; removed duplicates.

Reduced retrieval noise.

### 8.6 The data-quality dashboard

```
Per-dataset panel:
  Quality score (composite).
  Trend.
  Alert state.
  
Aggregate platform:
  Average noise across datasets.
  Drift incidents per quarter.
  Contamination incidents per quarter.
```

Reviewed monthly in data-engineering meeting.

### 8.7 The cross-team workflow

When quality issues arise:

- AI platform notified.
- Affected feature team notified.
- SMEs reviewed if needed.
- Resolution tracked.

### 8.8 The cost of the discipline

- Engineering: ~2 weeks initial; ongoing ~5% of platform team time.
- Tooling: ~$500/month (storage, dashboards).
- SME time: ~10 hours/month for audits.

Total: ~$10k/month.

### 8.9 The avoided cost

Estimated quality issues caught:

- Q1 noise drift: would have reduced classification accuracy 3-5% if uncaught.
- Q1 drift incident: misleading conclusions avoided.
- Q2 contamination: false-positive eval results prevented.

Several incidents per quarter; each meaningful.

### 8.10 The lessons

- Data quality is monitor-able; alert-able; SLO-able.
- Per-dataset tracking captures different patterns.
- SME audit is essential (no substitute).
- Dedup affects retrieval quality.

---

## 9. Anti-patterns

### 9.1 The unmeasured noise

**Pattern.** Datasets labeled; nobody measures noise.

**Corrective.** Audit per §2.

### 9.2 The "we'll catch drift in production" reactivity

**Pattern.** No drift detection; first signal is model regression.

**Corrective.** Periodic drift check per §3.

### 9.3 The contamination-not-checked assumption

**Pattern.** "Our training and eval are separate" without verification.

**Corrective.** Periodic audit per §4.7.

### 9.4 The exact-match-only dedup

**Pattern.** Hash dedup; near-duplicates remain.

**Corrective.** Semantic dedup per §5.3.

### 9.5 The dashboard-never-checked

**Pattern.** Quality dashboard exists; nobody reviews.

**Corrective.** Monthly review per §6.5.

### 9.6 The "noise is fine; we have lots of data" misconception

**Pattern.** Large dataset; assume noise washes out.

**Corrective.** Noise affects model; quantify per §2.

### 9.7 The drift-not-investigated dismissal

**Pattern.** Drift signal fires; "probably normal"; not investigated.

**Corrective.** Investigation per §3.6.

### 9.8 The "we re-deduplicated once" one-shot mentality

**Pattern.** Dedup at corpus build; not maintained.

**Corrective.** Ongoing dedup per §5.10.

### 9.9 The "label noise comes from one bad annotator; just remove them" reflex

**Pattern.** Quick fix; doesn't address rubric / training issues.

**Corrective.** Root-cause analysis per §2.4.

### 9.10 The chronic-quality-issue tolerance

**Pattern.** Quality issues persist; team accepts.

**Corrective.** Root-cause per §7.7.

---

## 10. Findings (sprint-assignable)

### DATA-DQ-001 — Severity: Critical
**Finding.** Noise quantification absent.
**Recommendation.** Per §2.
**Owner.** data engineering + AI platform, sprint N+1.

### DATA-DQ-002 — Severity: Critical
**Finding.** Drift detection absent.
**Recommendation.** Per §3.
**Owner.** data engineering + observability, sprint N+1.

### DATA-DQ-003 — Severity: Critical
**Finding.** Contamination check not performed.
**Recommendation.** Per §4.7.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-DQ-004 — Severity: High
**Finding.** Near-duplicate detection absent.
**Recommendation.** Per §5.3.
**Owner.** data engineering, sprint N+2.

### DATA-DQ-005 — Severity: High
**Finding.** Quality dashboard absent.
**Recommendation.** Per §6.
**Owner.** observability + data engineering, sprint N+2.

### DATA-DQ-006 — Severity: High
**Finding.** Quality SLO not defined.
**Recommendation.** Per §6.4.
**Owner.** AI platform + data engineering, sprint N+2.

### DATA-DQ-007 — Severity: High
**Finding.** Per-annotator noise tracking absent.
**Recommendation.** Per §2.9.
**Owner.** data engineering, sprint N+2.

### DATA-DQ-008 — Severity: Medium
**Finding.** Quality issues lack response workflow.
**Recommendation.** Per §7.
**Owner.** AI platform + data engineering, sprint N+3.

### DATA-DQ-009 — Severity: Medium
**Finding.** Dedup pipeline absent.
**Recommendation.** Per §5.10.
**Owner.** data engineering, sprint N+3.

### DATA-DQ-010 — Severity: Medium
**Finding.** Drift-alert tuning ad-hoc.
**Recommendation.** Per §3.7.
**Owner.** observability + data engineering, sprint N+3.

### DATA-DQ-011 — Severity: Medium
**Finding.** Eval-set noise not specially audited.
**Recommendation.** Per §2.6.
**Owner.** eval + data engineering, sprint N+3.

### DATA-DQ-012 — Severity: Medium
**Finding.** Cross-dataset quality interaction unconsidered.
**Recommendation.** Per §7.4.
**Owner.** AI platform, sprint N+3.

### DATA-DQ-013 — Severity: Medium
**Finding.** SME audit cadence absent.
**Recommendation.** Per §2.2.
**Owner.** data engineering + SMEs, sprint N+4.

### DATA-DQ-014 — Severity: Medium
**Finding.** Quality incident review absent.
**Recommendation.** Per §7.8.
**Owner.** SRE + data engineering, sprint N+4.

### DATA-DQ-015 — Severity: Low
**Finding.** Reverse contamination (eval→training) not checked.
**Recommendation.** Per §4.9.
**Owner.** data engineering, sprint N+5.

### DATA-DQ-016 — Severity: Low
**Finding.** Dedup decisions not reviewed for false positives.
**Recommendation.** Per §5.8.
**Owner.** data engineering, sprint N+5.

### DATA-DQ-017 — Severity: Low
**Finding.** Chronic-quality-issue tolerance.
**Recommendation.** Root-cause per §7.7.
**Owner.** data engineering + AI platform, sprint N+6.

### DATA-DQ-018 — Severity: Low
**Finding.** Cross-team quality issues lack workflow.
**Recommendation.** Per §8.7.
**Owner.** AI platform + data engineering, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Implement noise quantification per §2.**
- [ ] **Implement drift detection per §3.**
- [ ] **Implement contamination audit per §4.7.**
- [ ] **Implement dedup pipeline per §5.10.**
- [ ] **Build quality dashboard per §6.**
- [ ] **Define quality SLOs per §6.4.**
- [ ] **Set alert thresholds per §6.3.**
- [ ] **Monthly quality review per §6.5.**
- [ ] **Quality incident response per §7.**
- [ ] **Cross-team workflow per §8.7.**

---

## 12. References

**In this folder.**
- [labeling-and-annotation.md](./labeling-and-annotation.md) — labels generate noise (companion).
- [dataset-versioning.md](./dataset-versioning.md) — versioning enables quality history.
- [retrieval-corpus-engineering.md](./retrieval-corpus-engineering.md) — corpus quality.
- [synthetic-data-generation.md](./synthetic-data-generation.md) — synthetic data quality.
- [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md) — engineered separation.

**Elsewhere in this repo.**
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — model quality drift.
- [reliability-engineering/fault-budgets-for-ai.md](../reliability-engineering/fault-budgets-for-ai.md) — quality SLO.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — quality incident class.

**Sibling repos.**
- [ai-architecture-reference-architecture / data-architecture-for-ai / data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) — architectural data contracts.

**External.**
- Great Expectations documentation.
- Drift-detection literature.
- Data-quality canonical references.
