# Distillation Operations

> **Audience.** Engineers whose AI bill grew faster than expected and self-hosted open-weight inference is on the table. Tech leads whose latency requirements demand a smaller model than frontier provides. Anyone whose hosted frontier model is great for product quality but expensive at scale. **Scope.** The *engineering* practice of distilling a larger frontier model into a smaller open-weight model: when distillation is worth it; the data-collection pattern (frontier-as-teacher); the eval pattern that validates the distillate; the redistill-on-model-update lifecycle. Not the broader fine-tuning ops (see [fine-tuning-operations.md](./fine-tuning-operations.md), companion — distillation is a flavor of fine-tuning). Not the build-vs-buy decision (see [ai-architecture-reference-architecture / model-strategy / build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Distillation is a specific kind of fine-tuning:

- Frontier model (large; capable; expensive) generates training data.
- Smaller open-weight model trains on that data.
- Result: smaller model that approaches frontier quality on the specific workload.

The trade-off:

- Smaller model: faster, cheaper.
- Lower quality than frontier (usually).
- Operational ownership.

For some workloads: distillation is the right answer. For most: it isn't.

This document covers the engineering when it is.

This document is opinionated about four things:

1. **Distillation is for cost and latency, not capability.** It buys cost/latency at the price of capability ceiling.
2. **Production-traffic-driven distillation is the discipline.** Generate training data from real production patterns.
3. **Eval validates the distillate; doesn't skip.** Distillation can produce a model that does great in synthetic but fails in production.
4. **Re-distill when the teacher changes.** Frontier model updates cascade.

Structure: (2) when distillation fits; (3) the teacher-student pattern; (4) data collection; (5) training; (6) eval; (7) redistill lifecycle; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. When distillation fits

The legitimate cases.

### 2.1 The cost-at-scale case

Workload high volume:

- Frontier: $X per call × volume → significant spend.
- Distillate: $0.0001 per call (self-hosted) × volume → minimal.

Cost crossover at volume.

Cross-link to [ai-architecture-reference-architecture / cost-and-performance-architecture / gpu-strategy-for-self-hosted.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/cost-and-performance-architecture/gpu-strategy-for-self-hosted.md).

### 2.2 The latency-critical case

Workload needs sub-second TTFT:

- Frontier: 1-2s TTFT.
- Smaller model: 200-500ms.

For voice / real-time interfaces.

### 2.3 The on-device case

For deployment to edge:

- Frontier API: requires network.
- Local model: works offline.

For mobile, embedded.

### 2.4 The specific-domain case

Workload narrow:

- Frontier has general knowledge.
- Smaller model trained on domain.
- Sufficient for the workload.

For specialized workloads.

### 2.5 The privacy case

For workloads requiring on-prem:

- Frontier: data goes to provider.
- Local model: data stays.

Regulatory / compliance.

### 2.6 The "frontier is overkill" case

For workloads with clear, bounded scope:

- Frontier capability not fully utilized.
- Smaller model sufficient.

Per-workload analysis.

### 2.7 The cases distillation doesn't fit

- Workloads where capability ceiling matters.
- Workloads with shifting requirements.
- Low-volume workloads.
- Workloads where frontier costs are acceptable.

Don't distill these.

### 2.8 The decision matrix

```
Workload                Distillation?  Why
─────────────────────────────────────────────────────
High-volume routine     Yes            Cost
Latency-critical        Yes            Latency
Bounded domain          Yes            Sufficient
Frontier-capability     No             Need full capability
Rapidly changing        No             Re-train churn
Low volume              No             Not worth ops cost
```

Per workload.

### 2.9 The cost-benefit analysis

Per candidate workload:

```yaml
distillation-analysis:
  current_cost: $X/month (frontier hosted)
  projected_distilled_cost: $Y/month (self-hosted)
  savings_per_month: $X - $Y
  
  distillation_cost_per_year:
    data_generation: ~$Z
    training_runs: ~$W
    operational: 0.3 FTE allocation
  
  payback_period: months
  
  capability_risk: estimate (eval-driven)
```

Per workload.

---

## 3. The teacher-student pattern

The core mechanic.

### 3.1 The teacher

Frontier model:

- Generates outputs for inputs.
- "Demonstrates" the task.

### 3.2 The student

Smaller open-weight model:

- Learns to mimic teacher's outputs.

### 3.3 The pattern

```
For each input in distillation dataset:
  teacher_output = teacher.generate(input)
  store(input, teacher_output)

Train student on (input, teacher_output) pairs.

Result: student that approaches teacher quality.
```

Supervised learning with teacher providing labels.

### 3.4 The advantage over labeling

vs human labeling:

- Faster (teacher generates many examples).
- Cheaper (per case: ~$0.01-0.10 vs $5+).
- More uniform (no labeling drift).

vs labeling: distillation is much more efficient.

### 3.5 The student's quality ceiling

Student can approach teacher but not exceed:

- Teacher's mistakes propagate.
- Teacher's biases inherited.
- Capability ceiling = teacher's capability.

Choose teacher carefully.

### 3.6 The teacher selection

Common choices:

- Claude Sonnet / Opus.
- GPT-4 family.
- Gemini.

For specific domain: domain-specific frontier.

### 3.7 The student selection

Common choices:

- Llama 3 8B / 70B.
- Mistral 7B.
- Qwen variants.
- Phi variants.

Per workload's latency / cost targets.

### 3.8 The "same model family" pattern

For inheritance:

- Llama teacher (70B) → Llama student (8B).
- Same architecture.

Or different families (Claude → Llama). Both work.

### 3.9 The combined-teacher

For some workloads:

- Multiple teachers vote.
- Student learns from consensus.

More complex; better quality.

---

## 4. Data collection

The dataset for distillation.

### 4.1 The production-traffic source

Best data source:

- Real production inputs.
- Anonymized (cross-link to [data-engineering-for-ai/data-contracts-for-ai.md §5.2](../data-engineering-for-ai/data-contracts-for-ai.md)).
- Representative.

Production patterns; real-world distribution.

### 4.2 The teacher-generation

For each input:

- Send to teacher (frontier).
- Collect output.
- Store (input, output) pair.

Per-call cost: $0.02-0.10 (teacher generation).

### 4.3 The quality filter

Some teacher outputs are bad:

- Refusals.
- Errors.
- Hallucinations.

Filter:

- Schema validation.
- Quality judge.
- Sample human review.

Discard bad outputs.

### 4.4 The diversity check

Ensure dataset covers:

- All workload categories.
- Edge cases.
- Rare scenarios.

Imbalanced data → biased student.

### 4.5 The data volume

For effective distillation:

- 10,000+ examples typical.
- More is better.

Volume permits.

### 4.6 The data-cost calculation

```
volume = 10,000
teacher_cost_per_call = $0.05
total_data_cost = $500

Plus filtering / processing: ~$100
Total: ~$600 for initial dataset.
```

Cheaper than labeling.

### 4.7 The "we generated and trained; quality was poor" failure

Often:

- Data was uniform (similar inputs).
- Student didn't learn diversity.

Diversify inputs.

### 4.8 The "we generated against eval cases" contamination

If teacher generates against eval cases:

- Student learns eval cases.
- Eval inflated.

Cross-link to [data-engineering-for-ai/eval-data-contamination-prevention.md](../data-engineering-for-ai/eval-data-contamination-prevention.md).

### 4.9 The augmentation strategy

For coverage gaps:

- Generate synthetic inputs (separate from teacher generation).
- Or augment with adversarial cases.

Cross-link to [data-engineering-for-ai/synthetic-data-generation.md](../data-engineering-for-ai/synthetic-data-generation.md).

### 4.10 The dataset versioning

Per distillation dataset:

- Version.
- Generated date.
- Teacher used.
- Filters applied.

Cross-link to [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md).

---

## 5. Training

The distillation training process.

### 5.1 The training framework

Standard ML:

- HuggingFace Transformers + Trainer.
- PyTorch Lightning.
- Custom.

Choose per team's familiarity.

### 5.2 The training infrastructure

For Llama 8B:

- 1-2 GPUs.
- Training run: hours.

For Llama 70B:

- 4-8 GPUs.
- Training run: hours to a day.

Per model size.

### 5.3 The hyperparameters

Standard fine-tuning hyperparameters:

- Learning rate (e.g., 1e-5).
- Batch size (memory-dependent).
- Epochs (3-5 typical).
- Warmup schedule.

Cross-link to [fine-tuning-operations.md §3](./fine-tuning-operations.md).

### 5.4 The training-time iteration

For tuning:

- Initial training.
- Eval.
- Adjust hyperparameters.
- Re-train.
- Iterate.

Multiple training runs.

### 5.5 The CI integration

Per [fine-tuning-operations.md §4](./fine-tuning-operations.md):

- Training as CI job.
- Reproducible.

### 5.6 The training-data preparation

Beyond raw teacher outputs:

- Format (prompt structure).
- Truncation (token limits).
- Padding.

Standard training data prep.

### 5.7 The checkpoint saving

Per training:

- Save best model (lowest eval loss).
- Save final model.
- Versioned.

### 5.8 The "we trained 5 times; pick best" iteration

Standard:

- Multiple training runs.
- Choose best on eval.

### 5.9 The "we don't iterate" rush

If first training is shipped:

- Sub-optimal.
- Iteration improves.

Budget time for iteration.

---

## 6. Eval

The validation.

### 6.1 The eval suite

Per workload:

- Eval set (held-out; cross-link to [data-engineering-for-ai/training-eval-split-discipline.md](../data-engineering-for-ai/training-eval-split-discipline.md)).
- Test cases representative.

Same eval as fine-tuning.

### 6.2 The teacher vs student eval

Compare:

- Teacher on eval cases (baseline).
- Student on eval cases.
- Gap measurement.

Per case.

### 6.3 The "student matches teacher" criterion

Distillation is successful when:

- Student approaches teacher quality.
- Acceptable gap (e.g., 95% of teacher's performance).

Per workload.

### 6.4 The "student exceeds teacher" anomaly

Sometimes:

- Student does better than teacher on eval.
- Likely overfit to distillation data.
- Investigate (not real improvement).

### 6.5 The production-validation

Beyond eval:

- Production canary.
- Monitor real metrics.

Cross-link to [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md).

### 6.6 The "we deployed; production failed" surprise

If eval passed but production fails:

- Distillation data didn't represent production.
- Re-distill with more representative data.

### 6.7 The per-category eval

For workloads with sub-categories:

- Per-category performance.
- Student may fail on rare categories.

Targeted improvement.

### 6.8 The latency / cost validation

Beyond quality:

- Student latency: better than teacher.
- Student cost: much lower.

Confirm.

### 6.9 The "distillation gap is too large" decision

If student quality gap > acceptable:

- Investigate (data; hyperparameters).
- Iterate.
- Or: stick with teacher; accept costs.

### 6.10 The eval-result documentation

Per distillation:

```yaml
distillation-eval:
  teacher: claude-sonnet-4-6
  student: llama-3-70b-distilled-v1.0
  
  teacher_pass_rate: 96.2%
  student_pass_rate: 91.4%
  gap: 4.8 percentage points
  
  per_category:
    common: 96.5% (vs 96.8%; -0.3)
    edge_cases: 82.1% (vs 92.4%; -10.3) ⚠️
  
  latency_improvement: 5x faster
  cost_improvement: 50x cheaper
  
  decision: deploy with caveat; common cases excellent; flag edge cases for monitoring
```

Documented.

---

## 7. Redistill lifecycle

When to re-train.

### 7.1 The redistill triggers

- Teacher model upgraded.
- Workload shifted (new patterns).
- Quality drift detected.
- Data refresh.

Per trigger.

### 7.2 The teacher-upgrade cascade

When teacher updates:

- Generate fresh distillation data.
- Re-train student.
- Eval new student vs old.
- Decision: deploy new or keep old.

### 7.3 The "we shipped a distillate; never updated" risk

Without re-distillation:

- Student lags as teacher improves.
- Quality drifts.

Periodic refresh.

### 7.4 The "is the new student better than old" decision

Compare:

- Old student vs new student.
- If new ≥ old + margin: deploy.
- If similar: keep old (less disruption).

Per evaluation.

### 7.5 The refresh cadence

For most workloads:

- Quarterly or semi-annually.
- Aligned with teacher model release cycle.

### 7.6 The "we refresh too often" instability

Constant refresh:

- Student keeps changing.
- Production instability.

Stable cadence.

### 7.7 The deprecation of old students

After new student deployed:

- Old: deprecated.
- Retained for rollback (30-60 days).
- Removed.

Lifecycle.

### 7.8 The "what if new teacher is worse" anomaly

Rare; teacher might regress on workload.

If so:

- Keep old distillate.
- Or re-distill with different teacher.

### 7.9 The distillation-lifecycle as project

Like model migration:

- Each redistill is a project.
- Schedule.
- Owners.

Cross-link to [model-deprecation-playbook.md](./model-deprecation-playbook.md).

### 7.10 The infrastructure for re-distill

- Training infrastructure (GPU).
- Data infrastructure.
- Eval infrastructure.

Maintained.

---

## 8. Worked Meridian example

Meridian's distillation deployments.

### 8.1 The document classification distillate

Teacher: Claude Sonnet 4.6.

Student: Llama 3 70B fine-tuned (essentially a distillate).

Volume: 40k documents/day.

```
Distillation data:
  Inputs: production documents (40k samples).
  Teacher outputs: Sonnet classifications.
  Filtered: quality verified (95% pass filter).
  Dataset version: v12.0.0.

Training:
  Infrastructure: 4 H100 GPUs.
  Duration: 6 hours.
  Hyperparameters: lr=1e-5, epochs=3, batch=16.

Eval:
  Teacher (Sonnet): 96.2% pass rate.
  Student (Llama 70B): 95.2% pass rate.
  Gap: 1.0 percentage points (acceptable).
  Latency: 5x faster.
  Cost: 50x cheaper.

Deployment: production.
```

Successful.

### 8.2 The cost analysis

```
Pre-distillation:
  40k docs/day × $0.025 (Sonnet) = $1000/day = $30k/month.

Post-distillation:
  Self-host Llama 70B: 8 GPUs × $5/hour × 24 × 30 = $28,800/month effective.
  Wait — that's not much cheaper.
  But: 8 GPUs serve multiple workloads (not just document classification).
  Allocated to document classification: ~$5k/month + ops.

Savings: $25k/month.
```

Substantial.

### 8.3 The Q1 2026 redistill (teacher update)

Sonnet 4.5 → Sonnet 4.6 update:

- Decision: re-distill student?
- Eval test: Sonnet 4.6 + same prompt → 96.8% (vs 96.2%).
- Sonnet 4.6 as teacher → re-distill.

New student:

- Trained on 4.6 outputs.
- Eval: 95.8% pass rate (vs 95.2% old).
- Decision: deploy new.

Migration via canary.

### 8.4 The Q2 2026 quality-drift detection

Student-classification quality drifted:

- Eval pass rate dropped from 95.8% to 94.3%.
- Detected by quality-drift signal.

Cause investigation:

- Production data had shifted (new document types).
- Distillation data was 3 months old.

Action:

- Generated fresh distillation data.
- Re-trained student.
- Quality recovered to 95.5%.

Re-distill cadence: tightened to monthly.

### 8.5 The infrastructure cost

- Training: 8 GPUs × hours/training-run × runs/quarter.
- Data generation (teacher): ~$500 per training run.
- Engineering: ~0.3 FTE for distillation operations.

Total: ~$4k/month.

vs savings: ~$25k/month.

Net positive.

### 8.6 The decision: not all workloads distill

Considered distillation for other workloads:

- Care Coordinator: rejected (quality regression unacceptable; clinical safety).
- Patient API chat: rejected (workload too varied; capability needed).
- Document classification: yes (structured task; bounded scope).
- Billing-code: yes (different distillate; specific workflow).

Per-workload decision.

### 8.7 The "we tried; quality wasn't enough" learning

For analytics warehouse copilot:

- Tried distillation Q2 2026.
- Student quality: 88% (vs teacher 96%).
- Gap too large.
- Reverted to hosted Sonnet.

Lesson: not all workloads suit distillation.

### 8.8 The infrastructure

```
Self-hosted Llama cluster:
  8 H100 80GB GPUs.
  Models loaded:
    - Llama 3 70B (base inference).
    - Distilled-document-classifier (LoRA).
    - Distilled-billing-code (LoRA).
    - Embedding models.
  Serving: vLLM with continuous batching.
  Auto-scaling: 4-12 GPUs.
```

Shared infrastructure.

### 8.9 The lessons

- Distillation effective for bounded workloads with high volume.
- Re-distill cadence needs to match teacher updates.
- Quality drift requires monitoring.
- Not all workloads suit.

---

## 9. Anti-patterns

### 9.1 The "we'll just distill" enthusiasm

**Pattern.** Distillation tried without analysis. Quality gap unacceptable. Reverted.

**Corrective.** Cost-benefit analysis per §2.9.

### 9.2 The synthetic-data-only distillation

**Pattern.** Distill on synthetic inputs (not real). Student fails on real production.

**Corrective.** Production-traffic source per §4.1.

### 9.3 The unfiltered-teacher-output

**Pattern.** Teacher's mistakes propagate to student.

**Corrective.** Quality filter per §4.3.

### 9.4 The never-redistill student

**Pattern.** Initial distillate; never refreshed. Quality drifts.

**Corrective.** Periodic refresh per §7.5.

### 9.5 The capability-ceiling assumption

**Pattern.** Distillation chosen for workloads needing frontier capability.

**Corrective.** Per §2.7.

### 9.6 The "student exceeds teacher" misinterpretation

**Pattern.** Student passes eval at higher rate than teacher; declared "we beat the teacher."

**Corrective.** Likely overfit; investigate per §6.4.

### 9.7 The single-training-run rush

**Pattern.** First training is shipped without iteration.

**Corrective.** Multiple runs per §5.8.

### 9.8 The eval-set-leak in distillation

**Pattern.** Teacher generates against eval cases.

**Corrective.** Contamination check per §4.8.

### 9.9 The redistill-too-often instability

**Pattern.** Re-distill every week; production unstable.

**Corrective.** Stable cadence per §7.5.

### 9.10 The "we don't monitor distillate quality" gap

**Pattern.** Student deployed; quality not monitored; drift undetected.

**Corrective.** Quality drift detection per §8.4.

---

## 10. Findings (sprint-assignable)

### ML-DO-001 — Severity: Critical
**Finding.** Distillation tried without cost-benefit analysis.
**Recommendation.** Per §2.9.
**Owner.** AI platform + FinOps, sprint N+1.

### ML-DO-002 — Severity: High
**Finding.** Distillation on synthetic data (not production).
**Recommendation.** Per §4.1.
**Owner.** AI platform + data engineering, sprint N+2.

### ML-DO-003 — Severity: High
**Finding.** Teacher outputs not quality-filtered.
**Recommendation.** Per §4.3.
**Owner.** AI platform, sprint N+2.

### ML-DO-004 — Severity: High
**Finding.** Eval-set contamination not checked in distillation.
**Recommendation.** Per §4.8.
**Owner.** AI platform + data engineering, sprint N+2.

### ML-DO-005 — Severity: High
**Finding.** Student quality not validated against teacher pre-deployment.
**Recommendation.** Per §6.2.
**Owner.** AI platform + eval, sprint N+2.

### ML-DO-006 — Severity: High
**Finding.** Quality drift not monitored post-deployment.
**Recommendation.** Per §8.4.
**Owner.** AI platform + observability, sprint N+2.

### ML-DO-007 — Severity: Medium
**Finding.** Distillation dataset not versioned.
**Recommendation.** Per §4.10.
**Owner.** data engineering, sprint N+3.

### ML-DO-008 — Severity: Medium
**Finding.** Per-category eval not done.
**Recommendation.** Per §6.7.
**Owner.** AI platform + eval, sprint N+3.

### ML-DO-009 — Severity: Medium
**Finding.** Refresh cadence undefined.
**Recommendation.** Per §7.5.
**Owner.** AI platform, sprint N+3.

### ML-DO-010 — Severity: Medium
**Finding.** Distillation infrastructure ad-hoc.
**Recommendation.** Per §5.2 and §7.10.
**Owner.** AI platform + infra, sprint N+3.

### ML-DO-011 — Severity: Medium
**Finding.** Per-distillation cost not tracked.
**Recommendation.** Per §4.6.
**Owner.** FinOps + AI platform, sprint N+4.

### ML-DO-012 — Severity: Medium
**Finding.** Diversity in distillation data not checked.
**Recommendation.** Per §4.4.
**Owner.** data engineering, sprint N+4.

### ML-DO-013 — Severity: Medium
**Finding.** Combined-teacher pattern not explored.
**Recommendation.** Per §3.9.
**Owner.** AI platform, sprint N+4.

### ML-DO-014 — Severity: Low
**Finding.** Teacher-upgrade cascade not anticipated.
**Recommendation.** Per §7.2.
**Owner.** AI platform, sprint N+5.

### ML-DO-015 — Severity: Low
**Finding.** Latency / cost improvement not documented.
**Recommendation.** Per §6.8.
**Owner.** observability + FinOps, sprint N+5.

### ML-DO-016 — Severity: Low
**Finding.** Workload-suitability assessment not periodic.
**Recommendation.** Annual.
**Owner.** AI platform, sprint N+5.

### ML-DO-017 — Severity: Low
**Finding.** Distillation results across workloads not compared.
**Recommendation.** Per §8.6.
**Owner.** AI platform, sprint N+6.

### ML-DO-018 — Severity: Low
**Finding.** Cross-team distillation patterns not shared.
**Recommendation.** Internal documentation.
**Owner.** engineering management, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Cost-benefit analysis per §2.9.**
- [ ] **Production-traffic data collection per §4.1.**
- [ ] **Quality filter per §4.3.**
- [ ] **Contamination check per §4.8.**
- [ ] **Diversity check per §4.4.**
- [ ] **Eval discipline per §6.**
- [ ] **Versioned dataset per §4.10.**
- [ ] **Periodic refresh cadence per §7.5.**
- [ ] **Quality drift detection per §8.4.**
- [ ] **Teacher-upgrade cascade plan per §7.2.**

---

## 12. References

**In this folder.**
- [fine-tuning-operations.md](./fine-tuning-operations.md) — distillation is a flavor of fine-tuning.
- [model-promotion.md](./model-promotion.md) — promotion of distillate.
- [model-registry.md](./model-registry.md) — registry includes distillates.

**Elsewhere in this repo.**
- [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md) — versioning.
- [data-engineering-for-ai/synthetic-data-generation.md](../data-engineering-for-ai/synthetic-data-generation.md) — synthetic data alternative.
- [data-engineering-for-ai/eval-data-contamination-prevention.md](../data-engineering-for-ai/eval-data-contamination-prevention.md) — contamination.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — drift detection.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md) — distillation as build option.
- [ai-architecture-reference-architecture / cost-and-performance-architecture / gpu-strategy-for-self-hosted.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/cost-and-performance-architecture/gpu-strategy-for-self-hosted.md) — self-hosting context.

**External.**
- Knowledge distillation literature (Hinton et al., 2015 — foundational; many subsequent).
- Self-distillation techniques.
- DistilBERT, TinyBERT papers.
- "Constitutional AI" (Anthropic) — relates to teacher-driven training.
