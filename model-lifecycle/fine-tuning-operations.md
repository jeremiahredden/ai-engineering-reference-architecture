# Fine-Tuning Operations

> **Audience.** Engineers running fine-tuning workflows. Tech leads whose team has fine-tunes in production and the operations are starting to feel heavy. Anyone whose first fine-tune is shipping and they want to know what the lifecycle actually looks like. **Scope.** The *engineering* practice of fine-tuning operations: the lifecycle phases; data pipeline for fine-tune; fine-tune-as-CI-job pattern; eval comparison against base model; version control of fine-tuned models; deprecation playbook when the base is replaced. Not the architectural decision of whether to fine-tune (see [ai-architecture-reference-architecture / multi-tenancy-and-isolation / per-tenant-fine-tuning.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-fine-tuning.md)). Not the data engineering for fine-tune data (see [data-engineering-for-ai/](../data-engineering-for-ai/)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Fine-tuning is operationally the most expensive model-lifecycle pattern:

- Per-tenant model lifecycle.
- Per-tenant eval suite.
- Re-training when base model changes.
- Monitoring drift.
- Per-tenant rollback infrastructure.

The architectural sibling document covers when fine-tuning is justified (rarely; specific cases). This document covers what the operations look like once committed.

The pattern:

- First fine-tune: feels feasible.
- 5 fine-tunes: starts to feel heavy.
- 20 fine-tunes: dominates team's time.

Without operational discipline, fine-tuning becomes the team's primary work.

This document covers the engineering.

This document is opinionated about four things:

1. **Fine-tune lifecycle is a project.** Each fine-tune is its own thing to manage; budget operationally.
2. **Data pipeline matters more than training.** Quality fine-tune depends on data quality (cross-link to [data-engineering-for-ai/](../data-engineering-for-ai/)).
3. **CI-driven fine-tune is the discipline.** Manual training doesn't scale.
4. **Base-model changes cascade.** When the base updates, fine-tunes may need re-training.

Structure: (2) the fine-tune lifecycle phases; (3) data pipeline; (4) fine-tune-as-CI-job; (5) eval vs base; (6) versioning fine-tuned models; (7) base-model-change cascade; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The fine-tune lifecycle phases

What happens from idea to retirement.

### 2.1 Phase 1: Justification

Before fine-tuning:

- Verify need (per [per-tenant-fine-tuning.md §2](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-fine-tuning.md)).
- Verify data available.
- Verify operational capacity.

Decision; not assumed.

### 2.2 Phase 2: Data preparation

For training:

- Collect data.
- Validate quality.
- Curate.
- Split (train/val/test).

Cross-link to [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md).

### 2.3 Phase 3: Training

The actual fine-tune:

- Choose hyperparameters.
- Run training.
- Monitor convergence.
- Save checkpoint.

Standard ML.

### 2.4 Phase 4: Evaluation

After training:

- Run eval suite.
- Compare to base model.
- Compare to existing fine-tune (if migrating).
- Decision: deploy or iterate.

### 2.5 Phase 5: Deployment

If approved:

- Deploy via standard release process.
- Canary / shadow / full rollout.

Cross-link to [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) *(coming)*.

### 2.6 Phase 6: Monitoring

In production:

- Quality metrics.
- Drift detection.
- Cost monitoring.

Continuous.

### 2.7 Phase 7: Re-training

Periodically:

- Workload data shifts.
- Re-train to keep up.
- Or base model upgrades; re-train against new base.

Cycle.

### 2.8 Phase 8: Deprecation

Eventually:

- Workload sunsetted.
- Or replaced by stronger general model.
- Deprecate fine-tune.

End of life.

### 2.9 The timeline per phase

```
Phase                 Duration (typical)
─────────────────────────────────────────
Justification        Days to weeks
Data preparation     Weeks (often)
Training             Hours to days
Evaluation           Days to weeks
Deployment           Days to weeks
Monitoring           Ongoing
Re-training          Quarterly or per-base-change
Deprecation          Weeks (migration)
```

Substantial cumulative time.

### 2.10 The cumulative effort

For one fine-tune:

- Initial: ~6-8 weeks.
- Annual ongoing: ~3-4 cycles (re-training, base updates).
- Total annual: ~12-16 weeks of effort.

For multiple fine-tunes: linear with count.

---

## 3. Data pipeline

The training-data side.

### 3.1 The data requirements

For fine-tune:

- Quality data (curated).
- Sufficient volume (typically 1000+ examples).
- Representative of production.
- Split: train/val/test.

Cross-link to [data-engineering-for-ai/training-eval-split-discipline.md](../data-engineering-for-ai/training-eval-split-discipline.md).

### 3.2 The data sources

For training:

- Production data (anonymized; consented).
- Labeled data (vendor or internal).
- Synthetic data (augmenting rare cases).
- Mix.

Per workload.

### 3.3 The labeling pipeline

For labeled training data:

- Rubric (cross-link to [labeling-and-annotation.md §2](../data-engineering-for-ai/labeling-and-annotation.md)).
- Annotators (calibrated).
- Quality control.

Per data source.

### 3.4 The data versioning

Per training run:

- Dataset version pinned.
- Reproducible (re-train on same data → same result).

Cross-link to [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md).

### 3.5 The contamination prevention

Before training:

- Hash check against eval.
- Semantic check against eval.

Cross-link to [data-engineering-for-ai/eval-data-contamination-prevention.md](../data-engineering-for-ai/eval-data-contamination-prevention.md).

### 3.6 The data refresh

As production shifts:

- New data added.
- Old data archived (or removed if low quality).
- Re-train on updated data.

Per refresh.

### 3.7 The "we have a smaller dataset; do we still fine-tune" question

Per data volume:

- < 100 examples: too few; use few-shot instead.
- 100-1000: borderline; experiment.
- 1000+: viable.
- 10,000+: solid fine-tune territory.

Per workload's available data.

### 3.8 The synthetic-augmentation

For rare-case coverage:

- Synthetic examples.
- Cross-link to [data-engineering-for-ai/synthetic-data-generation.md](../data-engineering-for-ai/synthetic-data-generation.md).

Augmentation; not replacement.

### 3.9 The data-quality SLO

For training data:

- Quality threshold (e.g., 95% noise-free).
- Cross-link to [data-engineering-for-ai/data-quality-for-ai.md](../data-engineering-for-ai/data-quality-for-ai.md).

Monitored.

---

## 4. Fine-tune-as-CI-job

The CI integration.

### 4.1 The CI pattern

Fine-tuning as a CI workflow:

```
Trigger (manual or scheduled) →
  Data preparation step →
    Validate data quality →
      Run training →
        Run eval →
          Compare to baseline →
            If passes → deploy candidate →
              Else → fail with report.
```

Automated; reproducible.

### 4.2 The CI inputs

- Dataset version.
- Base model version.
- Hyperparameters.
- Eval suite version.

All pinned.

### 4.3 The CI outputs

- Fine-tuned model (saved).
- Eval results.
- Comparison report.
- Versioned artifact.

For tracking.

### 4.4 The CI integration with provider

For hosted fine-tunes (Anthropic, OpenAI):

- CI submits training job.
- Polls for completion.
- Retrieves trained model.
- Tests.

API-driven.

For self-hosted:

- CI provisions GPUs.
- Runs training.
- Stores adapter.
- Tests.

GPU-cluster-driven.

### 4.5 The CI cost

Each fine-tune CI run:

- API cost (hosted): $5-100 per run.
- Self-hosted: GPU-hours.
- Engineering time: ~hours.

Budget per run.

### 4.6 The CI cadence

- On data update.
- Scheduled (quarterly).
- On base-model change.
- On hyperparameter change.

Per trigger.

### 4.7 The CI failure handling

If CI fails:

- Investigate (data? infrastructure? hyperparameters?).
- Fix.
- Re-run.

Standard.

### 4.8 The CI vs manual

Manual fine-tuning:

- Each run improvised.
- Reproducibility difficult.
- Slow.

CI:

- Reproducible.
- Faster iteration.
- Documented.

Engineering discipline.

### 4.9 The "we don't have CI for fine-tune" reality

For early-stage:

- May be manual.
- But: as fine-tunes accumulate, CI essential.

Build CI when scaling.

---

## 5. Eval comparison against base

The decision-making artifact.

### 5.1 The eval-base comparison

Per fine-tune candidate:

- Run eval suite.
- Compare to base model (no fine-tune).
- Compare to existing fine-tune (if migrating).

Decision based on results.

### 5.2 The eval metrics

Per workload:

- Pass rate (quality).
- Per-category breakdown.
- Latency (if applicable).
- Cost (per call).

Per dimension.

### 5.3 The fine-tune-improvement threshold

Before deploying fine-tune:

- Must beat base by X% on workload-relevant metrics.
- Cost must be acceptable.
- Latency must be acceptable.

Per workload threshold.

### 5.4 The "we beat base by 2%" decision

Marginal improvement:

- 2% improvement justifies fine-tune lifecycle? Probably not.
- Need 5-10% or specific capability gain.

Cost-benefit; per workload.

### 5.5 The eval-comparison report

Per fine-tune:

```yaml
fine-tune-evaluation:
  candidate: document-classification-fine-tune-v12.0.0
  base: llama-3-70b
  eval_suite: document-classification-eval-v5.4.0
  
  results:
    candidate_pass_rate: 95.2%
    base_pass_rate: 88.4%
    improvement: +6.8 percentage points
  
  per_category:
    medical_records: 96.1% vs 89.2% (+6.9)
    insurance: 94.5% vs 86.5% (+8.0)
    ...
  
  latency_p99:
    candidate: 800ms
    base: 850ms
  
  cost_per_call:
    candidate: $0.0001 (self-hosted)
    base: $0.025 (hosted alternative)
  
  decision: DEPLOY
```

Structured.

### 5.6 The "eval-comparison is the source of truth"

Decision based on:

- Documented eval.
- Not subjective.
- Reproducible.

### 5.7 The "we re-ran eval; results changed" warning

Eval results should be stable:

- If they vary between runs: eval suite has noise.
- Investigate; tighten.

Cross-link to [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md).

### 5.8 The "is the improvement statistically significant" question

For large eval suites:

- Confidence intervals.
- Statistical significance tests.

For small: harder to be confident.

Per-workload.

### 5.9 The cross-fine-tune comparison

For multiple fine-tunes:

- Compare among them.
- Best wins.

Selection.

---

## 6. Versioning fine-tuned models

The lifecycle.

### 6.1 Per fine-tune

Each fine-tune has:

- ID.
- Base model.
- Dataset version.
- Hyperparameters.
- Trained-on date.
- Eval results.

Tracked.

Cross-link to [model-registry.md](./model-registry.md).

### 6.2 The model-registry entry

```yaml
fine-tune: document-classification-v12.0.0
status: in_production
base_model: llama-3-70b
training_dataset: document-classification-fine-tune-data-v12.0.0
hyperparameters:
  learning_rate: 1e-5
  epochs: 3
  batch_size: 16
trained_at: 2026-04-15
eval_pass_rate: 95.2%
deployed_at: 2026-04-20
deprecation: null
```

Catalog.

### 6.3 The deprecation lifecycle

When a fine-tune is replaced:

```yaml
fine-tune: document-classification-v11.0.0
status: deprecated
deprecation_announced: 2026-04-01
removal_date: 2026-06-01
replaced_by: document-classification-v12.0.0
```

Per deprecation lifecycle.

Cross-link to [model-strategy/model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md).

### 6.4 The retention policy

Per fine-tune:

- Active: production.
- Deprecated: kept for rollback window (typically 30-60 days).
- Removed: hard-deleted.

Cross-link to [data-engineering-for-ai/dataset-versioning.md §7](../data-engineering-for-ai/dataset-versioning.md).

### 6.5 The "we have 10 old fine-tunes; storage is large" cleanup

For old fine-tunes:

- Retain compliance period.
- Archive long-term.
- Remove past retention.

### 6.6 The version-pin in production

Production:

- Pinned to specific fine-tune version.
- Updates via release.

Cross-link to [model-promotion.md](./model-promotion.md) *(coming)*.

### 6.7 The roll-forward / roll-back

For fine-tune migrations:

- New fine-tune: roll forward via release.
- Issue: roll back to previous fine-tune.

Standard.

### 6.8 The "the eval suite was version X; current is version Y" coherence

When eval changes:

- Fine-tune may need re-eval.
- Or: comparison no longer apples-to-apples.

Track eval suite per fine-tune.

### 6.9 The cross-fine-tune relationship

For per-tenant fine-tunes:

- Each tenant has its own fine-tune.
- Versioned independently.
- Aggregate count grows.

### 6.10 The audit-log

Per fine-tune lifecycle event:

- Training initiated.
- Training completed.
- Deployed.
- Deprecated.
- Removed.

Tracked.

---

## 7. Base-model-change cascade

When the base model updates.

### 7.1 The cascade

When base updates (e.g., Llama 3 70B → Llama 4 70B):

- Existing fine-tunes are on old base.
- Quality vs new base: uncertain.
- Decision: re-train on new base? Continue with old base?

Per-fine-tune.

### 7.2 The "re-train on new base" workflow

For each fine-tune:

1. Verify dataset compatibility with new base.
2. Run CI training on new base.
3. Eval against new base + new fine-tune.
4. Compare to previous (old base + old fine-tune).
5. Decision.

Per fine-tune; not parallel.

### 7.3 The base-deprecation cascade

If old base is deprecated:

- Forced migration.
- All fine-tunes must re-train or be deprecated.

Significant work.

### 7.4 The cost of cascade

Per fine-tune re-training:

- Engineering time: 2-4 weeks.
- Training cost.
- Re-deployment.
- Eval.

For N fine-tunes: N × cost.

### 7.5 The "we have 20 fine-tunes; new base coming" challenge

For platforms with many:

- Pipeline retraining (concurrent).
- Prioritize critical fine-tunes.
- Maybe deprecate some.

Resource management.

### 7.6 The base-pinning vs base-upgrade

Choice:

- Pin to old base indefinitely (delays cascade).
- Upgrade base; retrain (immediate work).

Per fine-tune; per base-deprecation.

### 7.7 The hosted-fine-tune cascade

For provider-hosted fine-tunes:

- Provider may force migration to new base.
- Less control.

For self-hosted: more flexibility.

### 7.8 The base-change-with-LoRA-portability

Some hope:

- LoRAs transfer to new base.
- In practice: rarely robust.

Plan for re-training.

### 7.9 The cascade as planned project

Treat as project:

- Timeline.
- Per-fine-tune sub-tasks.
- Resource allocation.

Cross-link to [model-deprecation-playbook.md](./model-deprecation-playbook.md) *(coming)*.

### 7.10 The pre-cascade strategic decision

Before next base release:

- Which fine-tunes are still needed?
- Could some be deprecated?
- Reduce cascade burden.

Strategic review.

---

## 8. Worked Meridian example

Meridian's fine-tune operations.

### 8.1 The fine-tune catalog

```
document-classification-fine-tune (v12.0.0):
  Base: Llama 3 70B
  Volume: 40k docs/day
  Quality: 95.2% pass rate (vs base 88.4%)
  Lifecycle: production
  Owner: AI platform

billing-code-fine-tune (specialty practice; v3.0.0):
  Base: Llama 3 8B + LoRA
  Tenant: 1 (specific specialty practice)
  Volume: 50k calls/month
  Quality: 99.4% structured-output reliability
  Lifecycle: production
  Owner: AI platform

meridian-classifier (v8.0.0):
  Base: Llama 3 70B
  Volume: 5k calls/day
  Quality: 96.1%
  Lifecycle: production
  Owner: AI platform
```

3 production fine-tunes.

### 8.2 The fine-tune lifecycle

For document-classification:

```
Q4 2025: justification + decision
Q4 2025 - Q1 2026: data preparation (4 weeks)
Q1 2026: training + eval (2 weeks)
Q1 2026: deployment (1 week)
Q1 2026: production
Q2 2026: monitoring; light re-tuning
Q3 2026: data refresh; re-train v12.0.0 → v12.1.0
Q4 2026: planned re-train on new base (Llama 4)
```

Per-quarter cycle.

### 8.3 The CI pipeline

```yaml
fine-tune-ci-pipeline:
  trigger: scheduled monthly (or manual)
  
  steps:
    - validate_data_quality
    - validate_contamination_check (vs eval)
    - run_training (8 GPUs; ~6 hours)
    - run_eval_suite
    - compare_to_baseline
    - generate_eval_report
    - if pass_threshold: store_candidate
    - if not: notify; investigate
```

Automated.

### 8.4 The Q1 2026 base-cascade preparation

Anticipating Llama 4 (later 2026):

- Inventory: 3 fine-tunes.
- Resource estimate: ~6-8 weeks total (engineering).
- Strategic question: do all 3 need to migrate?

Decision: migrate document-classification (highest volume); migrate meridian-classifier; defer billing-code (specialty practice may not justify re-train).

### 8.5 The eval-comparison disciplines

For each fine-tune:

- Eval suite versioned.
- Per-release: eval vs base + previous version.
- Documented decision.

### 8.6 The deprecation history

```
2024 fine-tune v1: deprecated 2025-Q1; archived
2025 fine-tune v5: deprecated 2025-Q3; archived
2025 fine-tune v8: deprecated 2026-Q1; archived
Current: v12.0.0
```

Versioning evolved.

### 8.7 The infrastructure cost

- Training infrastructure: 8 GPUs (shared with serving); ~4-6 hours per training run.
- Storage: ~10TB for current + archived versions.
- Operations: ~0.3 FTE per active fine-tune.

For 3 fine-tunes: ~1 FTE total operational.

### 8.8 The quality monitoring

Per fine-tune:

- Daily eval-suite re-run (sample).
- Quality drift detection.
- Cost monitoring.

Cross-link to [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md).

### 8.9 The decision: don't add a 4th fine-tune

A new feature considered for fine-tuning:

- Eval showed only 3% improvement.
- Operational cost not justified.
- Decision: don't fine-tune; use prompt + few-shot.

Cost-benefit per workload.

### 8.10 The lessons

- Fine-tune is operationally heavy; budget realistically.
- CI is essential at scale.
- Base-model changes cascade.
- Strategic decisions about which fine-tunes to keep matter.

---

## 9. Anti-patterns

### 9.1 The "we'll just fine-tune; how hard could it be" naive entry

**Pattern.** Fine-tune started without budgeting operations. Six months later, team overwhelmed.

**Corrective.** Budget operational cost per §2.

### 9.2 The manual training without CI

**Pattern.** Each fine-tune is improvised. Reproducibility difficult.

**Corrective.** CI per §4.

### 9.3 The training on contaminated data

**Pattern.** Eval cases in training data. Fine-tune inflated.

**Corrective.** Contamination check per §3.5.

### 9.4 The skipped eval-vs-base

**Pattern.** Fine-tune deployed without comparison to base.

**Corrective.** Per §5.

### 9.5 The deployment-without-rollback

**Pattern.** Fine-tune deployed; rollback path unclear.

**Corrective.** Version pinning + rollback per §6.7.

### 9.6 The "we'll re-train when we have time" deferral

**Pattern.** Quality drifts; re-training deferred indefinitely.

**Corrective.** Scheduled retraining per §2.7.

### 9.7 The base-change-unprepared

**Pattern.** New base released; cascade work hits team unexpectedly.

**Corrective.** Plan per §7.9.

### 9.8 The "we have 20 fine-tunes; can't keep up" overload

**Pattern.** Too many fine-tunes; operational burden exceeds team.

**Corrective.** Strategic decisions per §7.10; reduce count.

### 9.9 The dataset-not-versioned

**Pattern.** Fine-tune trained on "the data"; not specific version.

**Corrective.** Cross-link to [dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md).

### 9.10 The eval-suite-version-not-pinned

**Pattern.** Different eval runs use different eval suites; comparisons unstable.

**Corrective.** Pin per §6.8.

---

## 10. Findings (sprint-assignable)

### ML-FTO-001 — Severity: Critical
**Finding.** Fine-tune workflow manual; not CI.
**Recommendation.** Per §4.
**Owner.** AI platform, sprint N+1.

### ML-FTO-002 — Severity: Critical
**Finding.** Eval-vs-base comparison not done.
**Recommendation.** Per §5.
**Owner.** AI platform + eval, sprint N+1.

### ML-FTO-003 — Severity: Critical
**Finding.** Contamination check before training absent.
**Recommendation.** Per §3.5.
**Owner.** AI platform + data engineering, sprint N+1.

### ML-FTO-004 — Severity: High
**Finding.** Fine-tune dataset not versioned.
**Recommendation.** Per §3.4 and [dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md).
**Owner.** data engineering, sprint N+2.

### ML-FTO-005 — Severity: High
**Finding.** Fine-tunes not in registry.
**Recommendation.** Per §6.2.
**Owner.** AI platform, sprint N+2.

### ML-FTO-006 — Severity: High
**Finding.** Rollback path absent for fine-tunes.
**Recommendation.** Per §6.7.
**Owner.** AI platform, sprint N+2.

### ML-FTO-007 — Severity: High
**Finding.** Quality monitoring absent post-deployment.
**Recommendation.** Per §8.8.
**Owner.** AI platform + observability, sprint N+2.

### ML-FTO-008 — Severity: High
**Finding.** Base-model-change cascade not planned.
**Recommendation.** Per §7.9.
**Owner.** AI platform + engineering management, sprint N+2.

### ML-FTO-009 — Severity: Medium
**Finding.** Re-training cadence absent.
**Recommendation.** Per §2.7.
**Owner.** AI platform, sprint N+3.

### ML-FTO-010 — Severity: Medium
**Finding.** Per-fine-tune ownership unclear.
**Recommendation.** Per §6.10 and ownership column.
**Owner.** AI platform + engineering management, sprint N+3.

### ML-FTO-011 — Severity: Medium
**Finding.** Strategic review of fine-tune portfolio absent.
**Recommendation.** Per §7.10.
**Owner.** engineering management + AI platform, sprint N+3.

### ML-FTO-012 — Severity: Medium
**Finding.** Deprecation lifecycle for fine-tunes undefined.
**Recommendation.** Per §6.3.
**Owner.** AI platform, sprint N+3.

### ML-FTO-013 — Severity: Medium
**Finding.** Hyperparameter selection ad-hoc.
**Recommendation.** Per §4.2.
**Owner.** AI platform, sprint N+3.

### ML-FTO-014 — Severity: Medium
**Finding.** Eval suite version not pinned per fine-tune.
**Recommendation.** Per §6.8.
**Owner.** AI platform, sprint N+4.

### ML-FTO-015 — Severity: Low
**Finding.** Cross-fine-tune comparison absent.
**Recommendation.** Per §5.9.
**Owner.** AI platform, sprint N+5.

### ML-FTO-016 — Severity: Low
**Finding.** Annual fine-tune-portfolio review absent.
**Recommendation.** Per §7.10.
**Owner.** engineering management, sprint N+5.

### ML-FTO-017 — Severity: Low
**Finding.** Cost-per-fine-tune lifecycle not tracked.
**Recommendation.** Per §2.10.
**Owner.** FinOps + AI platform, sprint N+6.

### ML-FTO-018 — Severity: Low
**Finding.** Synthetic-data-augmentation discipline absent.
**Recommendation.** Per §3.8.
**Owner.** AI platform + data engineering, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Justification step per §2.1.**
- [ ] **Data pipeline per §3.**
- [ ] **CI pipeline per §4.**
- [ ] **Eval-vs-base comparison per §5.**
- [ ] **Registry integration per §6.2.**
- [ ] **Rollback path per §6.7.**
- [ ] **Quality monitoring post-deployment per §8.8.**
- [ ] **Base-cascade planning per §7.9.**
- [ ] **Strategic portfolio review per §7.10.**
- [ ] **Annual review of fine-tune count.**

---

## 12. References

**In this folder.**
- [model-registry.md](./model-registry.md) — fine-tune registry.
- [model-promotion.md](./model-promotion.md) *(coming)* — deploy workflow.
- [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) *(coming)* — rollout.
- [rollback-procedures.md](./rollback-procedures.md) *(coming)* — rollback.
- [model-deprecation-playbook.md](./model-deprecation-playbook.md) *(coming)* — deprecation.
- [distillation-operations.md](./distillation-operations.md) *(coming)* — distillation.

**Elsewhere in this repo.**
- [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md) — dataset versioning.
- [data-engineering-for-ai/training-eval-split-discipline.md](../data-engineering-for-ai/training-eval-split-discipline.md) — splits.
- [data-engineering-for-ai/eval-data-contamination-prevention.md](../data-engineering-for-ai/eval-data-contamination-prevention.md) — contamination.
- [data-engineering-for-ai/labeling-and-annotation.md](../data-engineering-for-ai/labeling-and-annotation.md) — labeling.
- [data-engineering-for-ai/synthetic-data-generation.md](../data-engineering-for-ai/synthetic-data-generation.md) — synthetic data.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — eval discipline.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — drift.

**Sibling repos.**
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / per-tenant-fine-tuning.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-fine-tuning.md) — architectural framing.
- [ai-architecture-reference-architecture / model-strategy / build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md) — fine-tune as build option.

**External.**
- LoRA paper (Hu et al., 2021).
- Provider fine-tuning documentation (Anthropic, OpenAI, AWS Bedrock).
- HuggingFace Transformers fine-tuning guides.
