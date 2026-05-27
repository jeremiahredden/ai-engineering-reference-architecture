# Synthetic Data Generation

> **Audience.** Engineers whose eval set has 50 cases and they need 500. Tech leads whose fine-tune workload is data-starved but real production data is unavailable. Anyone who has heard "LLMs can generate their own training data" and is wondering when that's actually true. **Scope.** The *engineering* practice of synthetic data generation: when synthetic data earns its place (augmenting rare-case coverage, generating adversarial cases, bootstrapping evals); the LLM-as-generator pattern; calibration discipline; failure modes. Not the labeling discipline (see [labeling-and-annotation.md](./labeling-and-annotation.md)). Not the contamination concerns (see [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Synthetic data is tempting:

- Real production data is limited.
- Labeling is expensive.
- LLMs can generate text on demand.

Generated data: free, plentiful, customizable.

The catch:

- Synthetic data teaches the model to do well on synthetic data.
- If the synthetic distribution differs from real, the model fails on real.
- Synthetic eval can be misleading.
- Synthetic fine-tune data can produce specialized failure modes.

Synthetic data has its place; the discipline is knowing when and how.

This document is opinionated about four things:

1. **Synthetic data is a useful tool, not a default.** Real data when available; synthetic when justified.
2. **Synthetic-distribution must match real-distribution where it counts.** Otherwise model learns synthetic-shaped tasks, not real ones.
3. **Synthetic eval is risky.** Real-world eval cases tell you about reality; synthetic ones tell you about your synthetic generator.
4. **Document the synthetic source per dataset.** Future debugging requires it.

Structure: (2) when synthetic earns its place; (3) the LLM-as-generator pattern; (4) calibration; (5) synthetic for evals; (6) synthetic for fine-tune; (7) failure modes; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. When synthetic earns its place

The legitimate cases.

### 2.1 Bootstrapping eval before real data exists

A new feature ships:

- Real production data: zero (haven't launched).
- Need eval cases before launch.

Synthetic eval:

- LLM-generated cases.
- Used pre-launch to validate.
- Replaced / supplemented with real cases post-launch.

### 2.2 Augmenting rare-case coverage

Real production data has uneven distribution:

- Common cases: many.
- Rare cases: few.

Model trained on this distribution: good at common, bad at rare.

Synthetic for rare cases:

- Generate examples of rare scenarios.
- Augment training set.

### 2.3 Adversarial / edge cases

For robustness:

- Adversarial inputs (attacks).
- Edge cases (unusual inputs).

LLM can generate these for stress-testing.

### 2.4 Privacy-preserving substitutes

Real data may have PII:

- Synthetic substitutes preserve patterns; remove identifying info.

For training where real data has constraints.

### 2.5 Multilingual coverage

Real data in one language:

- Synthetic translations to other languages.
- Bootstrap multilingual support.

### 2.6 The "we don't have data" failure mode

Sometimes synthetic is the only option:

- No real data exists.
- Generate synthetic; ship.

But: validate against real data as soon as possible.

### 2.7 The "synthetic is faster than real" justification

For some workloads:

- Real labeling: 6 months.
- Synthetic generation: 1 week.

Time-to-market.

But: quality may differ; verify.

### 2.8 The cases where synthetic doesn't fit

Don't use synthetic for:

- Workloads where real distribution matters (specialty cases).
- Workloads with subtle distributional patterns LLMs miss.
- Workloads where synthetic teaches LLM bias (LLM-generated data has LLM-style; trains models to match).

Be selective.

---

## 3. The LLM-as-generator pattern

How to generate.

### 3.1 The basic pattern

```python
def generate_synthetic_cases(prompt_template, count):
    cases = []
    for i in range(count):
        case = llm.call(prompt_template.format(seed=i))
        cases.append(case)
    return cases
```

LLM generates from a prompt template.

### 3.2 The prompt design

```
Generate a synthetic clinical case with these properties:
- Patient demographics: <random>
- Chief complaint: <varied>
- Clinical context: realistic for the demographic
- Expected output (per rubric): specified

Output format: JSON {patient_demographics, complaint, context, expected_output}
```

Detailed; structured.

### 3.3 The seed variation

For diversity:

- Random seeds.
- Variable demographics.
- Variable scenarios.

Prevents homogeneity.

### 3.4 The provider choice

For generation:

- Different from production model? (Reduces in-distribution overfitting.)
- Or same model? (Easier; risk of overfitting if used for training.)

Per use case.

### 3.5 The quality validation

Per synthetic case:

- Validate against rubric.
- Sample SME review.

Generated isn't automatically correct.

### 3.6 The "regenerate the bad ones" iteration

Cases that fail validation:

- Discard.
- Or regenerate with adjusted prompt.

Quality filter.

### 3.7 The cost-per-synthetic-case

Generation cost:

- Per case: $0.005-0.05.
- 500 cases: $2.50-25.

Cheap relative to labeling.

### 3.8 The data-quality of synthetic

Synthetic data quality:

- Often higher than rushed labeling.
- Sometimes lower than careful labeling.

Per workload.

### 3.9 The "we have humans curate the synthetic" hybrid

For higher quality:

- LLM generates candidates.
- Human reviews/edits.
- Final cases used.

Combines speed and quality.

---

## 4. Calibration

Ensuring synthetic-distribution matches real.

### 4.1 The distribution match question

Synthetic data has its own distribution:

- Vocabulary patterns.
- Length distribution.
- Topic distribution.

Real data has its own distribution.

If they differ: model trained on synthetic doesn't generalize to real.

### 4.2 The measurement

Compute distribution metrics:

- Vocabulary distribution (TF-IDF).
- Length distribution.
- Topic distribution (LDA or embedding cluster).

Compare synthetic vs real.

### 4.3 The "synthetic distribution differs" diagnostic

If divergence > threshold:

- Generator's prompt is too constrained.
- Or seeds aren't diverse enough.

Iterate the generator.

### 4.4 The "we don't have real to compare to" bootstrap

Before launch, no real data:

- Calibrate against:
  - SME-described expectations.
  - Comparable systems' data (proxy).
  - Held-out small real sample.

Best-effort.

### 4.5 The post-launch recalibration

Once real data exists:

- Compare synthetic vs real distribution.
- Iterate synthetic; re-train.

Iterative.

### 4.6 The "synthetic distribution matches; model still fails on real" surprise

Sometimes:

- Distribution matches statistically.
- Model fails on real anyway.

Cause: subtle features not captured by metrics.

Mitigation:

- Verify with real eval cases.
- Don't rely solely on synthetic.

### 4.7 The mix-ratio decision

For training:

- 100% synthetic: risky.
- 50/50 synthetic + real: better.
- 20/80 synthetic + real: better still.

Tilt toward real where possible.

### 4.8 The synthetic-as-augmentation pattern

Common pattern:

- 80% real data.
- 20% synthetic (augmenting rare cases).
- Synthetic specifically targets coverage gaps.

Augmentation, not replacement.

### 4.9 The eval calibration

For eval (separate from training):

- 100% real preferred.
- Synthetic only when real impossible.

Eval validates model against reality.

---

## 5. Synthetic for evals

The risky case.

### 5.1 The risk

Synthetic eval tells you:

- How the model does on synthetic-generated cases.

It doesn't tell you:

- How the model does on real production cases.

The mismatch can be substantial.

### 5.2 The bootstrap case (acceptable)

Pre-launch:

- Generate synthetic eval.
- Use it for development.
- Replace / augment with real cases post-launch.

Temporary; explicit.

### 5.3 The "we never replaced synthetic eval" mistake

Months pass:

- Synthetic eval still in use.
- Production behavior differs from eval.

**Corrective.** Replace; or supplement.

### 5.4 The hybrid eval

For mature platforms:

- 80% real cases (production-sampled, labeled).
- 20% synthetic (covers rare scenarios; adversarial).

Real plus targeted synthetic augmentation.

### 5.5 The synthetic-eval validation

If synthetic eval is used:

- Periodically compare to real-data spot-checks.
- Discrepancy indicates synthetic eval is misleading.

Detection.

### 5.6 The "synthetic eval got worse; real got better" inversion

If model improves on real but synthetic eval declines:

- Synthetic eval no longer matches reality.
- Replace synthetic eval.

Eval drift.

### 5.7 The "we eval against synthetic to validate synthetic" circular trap

Don't:

- Generate synthetic.
- Eval the model on synthetic.
- Declare success.

Eval against real where possible.

### 5.8 The adversarial-synthetic-eval value

For robustness:

- Synthetic adversarial cases.
- "Can the model handle these edge cases?"
- Different purpose from general eval.

Per-purpose.

---

## 6. Synthetic for fine-tune

Training data from LLMs.

### 6.1 The motivation

For fine-tune:

- Need 1000s of (input, output) pairs.
- Real production data limited.
- Synthetic to bootstrap.

### 6.2 The risk

LLM-generated training data:

- Teaches model to mimic LLM patterns.
- May not match real production needs.

### 6.3 The pattern

Common approach:

1. Generate diverse synthetic training examples.
2. Curate (human review or filter).
3. Combine with available real data.
4. Train.

Augmentation.

### 6.4 The "we trained on 100% synthetic" failure

Model trained on synthetic only:

- Excellent on synthetic-shaped tasks.
- Poor on real production.

Cross-validate against real.

### 6.5 The provider-choice for generation

For training data:

- Different LLM than the one being trained.
- Avoid in-distribution overfitting.

If training Llama: generate with Claude.

### 6.6 The quality vs quantity

Synthetic quality:

- Hard to verify at scale.
- Spot-check.

For high-quality training data: smaller volume, more curation.

For exploratory: larger volume, less curation.

### 6.7 The "synthetic data leaked into eval" anti-pattern

If synthetic training data inadvertently overlaps with eval:

- Eval inflated.
- Cross-link to [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md).

Engineering controls.

### 6.8 The distillation as synthetic data

Distillation: train smaller model on larger model's outputs.

This is synthetic-data-generation:

- Generator: large model.
- Output: training data for smaller model.

Cross-link to [model-lifecycle/distillation-operations.md](../model-lifecycle/distillation-operations.md).

### 6.9 The "we used synthetic; fine-tune failed" lessons

Common causes:

- Synthetic too uniform (lacks diversity).
- Synthetic doesn't match real-task complexity.
- Quality filter removed too much; remaining is biased.

Iterate.

---

## 7. Failure modes

Where synthetic goes wrong.

### 7.1 The distributional mismatch

Synthetic distribution ≠ real distribution.

Model overfits to synthetic shape.

Detection: distribution metrics.

### 7.2 The LLM-style contamination

LLM-generated data has LLM-style:

- Verbose.
- Markdown-rich.
- Particular phrasings.

Model trained on this:

- Generates LLM-style outputs.
- May not match user expectations.

### 7.3 The synthetic-eval inflation

Eval scores against synthetic look great:

- Model overfits to synthetic.
- Real performance unknown.

Track real performance separately.

### 7.4 The hidden bias

LLM has biases:

- Generates biased synthetic data.
- Model learns the biases.

Diagnose: bias audits.

### 7.5 The diversity collapse

Generator produces similar outputs:

- Cases look different but cover same scenarios.
- Coverage limited.

Detection: cluster analysis.

### 7.6 The contamination back to model

If generator is the model:

- Model generates examples it can easily handle.
- Training reinforces what model already does.
- No improvement.

Different model for generation.

### 7.7 The "we relied on synthetic; real production broke us" surprise

After deployment:

- Real users behave differently.
- Model fails on cases synthetic missed.

Pre-launch synthetic + post-launch real iteration.

### 7.8 The "synthetic is good enough; we'll skip real" temptation

Synthetic is cheaper:

- Easy to lean on.
- Skip the expensive real labeling.

Eventually: production data mismatches.

### 7.9 The lifecycle drift

Initially synthetic was calibrated:

- Reality has shifted.
- Synthetic still calibrated to old reality.

Re-calibrate periodically.

---

## 8. Worked Meridian example

Meridian's synthetic data usage.

### 8.1 The synthetic deployments

```
Use case                          Synthetic role
─────────────────────────────────────────────────────
Care Coordinator eval (post-launch) 20% synthetic (rare cases)
                                    80% real (production-sampled)

Patient API chat eval                 100% real (workload has plenty of real data)

Document classification               80% real / 20% synthetic for rare categories
fine-tune data                       

Clinical decision support eval      30% synthetic (rare clinical scenarios)
                                    70% real
                                    
Billing-code training (specialty)   100% real (workflow-specific; no synthetic helpful)
```

Per workload.

### 8.2 The synthetic for Care Coordinator eval

Care Coordinator deals with many scenarios:

- Common: 80% of cases.
- Rare (renal failure + diabetes + heart failure + ...): 5%.

Real data has few of the rare cases.

Synthetic: generate cases of rare combinations.

Process:

1. SMEs identify gap scenarios.
2. LLM generates cases per scenario.
3. SMEs review and curate.
4. Cases added to eval.

50 synthetic cases added to eval; covers gap.

### 8.3 The synthetic for document classification training

Document classification has 15 categories:

- Some categories: thousands of examples.
- Rare categories: tens of examples.

Synthetic for rare:

- LLM generates examples of "consultation_report" (50 real → 200 with synthetic).
- LLM generates examples of "imaging_report" (30 real → 150 with synthetic).

Training data balanced.

### 8.4 The Q1 2026 synthetic-eval validation

Care Coordinator eval (mix of real + synthetic):

- Spot-check: real cases pass at 96%.
- Synthetic cases pass at 95%.

Distributions match; synthetic eval is calibrated.

### 8.5 The Q2 2026 synthetic-training failure

For a specialty workflow, attempted 100% synthetic training:

- 1000 synthetic examples.
- Fine-tuned Llama 8B.
- Quality on real production: 65% (below 85% threshold).

Diagnosis:

- Synthetic lacked real-world complexity.
- Real production cases had unique features synthetic missed.

Recovery:

- Collected 200 real cases.
- Mixed 50/50 with synthetic.
- Re-trained.
- Quality: 88%; acceptable.

Lesson: synthetic alone wasn't enough.

### 8.6 The synthetic-data tracking

Per synthetic dataset:

```yaml
synthetic_dataset:
  name: care-coordinator-rare-cases-eval
  generator: claude-sonnet-4-6
  generator_version: v4.6
  generation_date: 2026-01-15
  prompt_template_version: v1.2
  case_count: 50
  curation: SME-reviewed
  validation_against_real: passed (95% / 96% match)
```

Tracked alongside datasets.

### 8.7 The cost of synthetic

- Generation: ~$0.02 per case × 50 cases = $1.
- SME review: ~30 min per 10 cases.
- Curation: ~10 hours total for 50 cases.

Cost per useful synthetic case: ~$25.

vs labeling 50 real cases: ~$50.

Faster + cheaper for this use case.

### 8.8 The Q3 2026 synthetic-data drift

The Care Coordinator synthetic eval cases had been in use 6 months:

- Real workflow had evolved.
- Synthetic cases no longer matched real production.

Detected by: declining synthetic-eval correlation with real-data pass rate.

Action:

- Regenerated synthetic with updated prompt template.
- Recalibrated.

Lesson: synthetic data ages too.

### 8.9 The lessons

- Synthetic augments; doesn't replace.
- Calibration is essential.
- Mix synthetic with real where possible.
- Re-calibrate as workflows evolve.

---

## 9. Anti-patterns

### 9.1 The 100% synthetic training

**Pattern.** No real data; trained entirely on synthetic. Model overfits.

**Corrective.** Augmentation, not replacement, per §4.7.

### 9.2 The synthetic-only eval

**Pattern.** Eval cases all synthetic; production behavior unknown.

**Corrective.** Real eval cases per §5.2.

### 9.3 The uncalibrated synthetic

**Pattern.** Synthetic generated without checking against real distribution.

**Corrective.** Calibration per §4.

### 9.4 The "we'll replace synthetic later" deferral

**Pattern.** Synthetic at launch; never replaced.

**Corrective.** Post-launch real per §5.3.

### 9.5 The same-model generation + training

**Pattern.** Model generates training data for itself.

**Corrective.** Different model for generation per §6.5.

### 9.6 The undocumented synthetic source

**Pattern.** Synthetic data accumulates; nobody knows where it came from.

**Corrective.** Track per §8.6.

### 9.7 The LLM-style training corrupts model

**Pattern.** Model trained on LLM-generated data; outputs are now markdown-rich verbose; doesn't match user expectations.

**Corrective.** Style awareness in generation; diverse generators.

### 9.8 The diversity-collapse

**Pattern.** Generator produces similar cases; coverage limited.

**Corrective.** Diverse prompts + cluster analysis per §7.5.

### 9.9 The "synthetic looked good; real broke us" surprise

**Pattern.** Synthetic eval passing; production failing.

**Corrective.** Real validation per §7.7.

### 9.10 The synthetic-data drift unnoticed

**Pattern.** Synthetic data ages without re-calibration.

**Corrective.** Periodic re-calibration per §8.8.

---

## 10. Findings (sprint-assignable)

### DATA-SDG-001 — Severity: Critical
**Finding.** 100% synthetic training without validation.
**Recommendation.** Mix with real per §4.7 and §6.4.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-SDG-002 — Severity: Critical
**Finding.** Synthetic eval not validated against real.
**Recommendation.** Periodic validation per §5.5.
**Owner.** eval + AI platform, sprint N+1.

### DATA-SDG-003 — Severity: High
**Finding.** No calibration of synthetic vs real distribution.
**Recommendation.** Per §4.
**Owner.** data engineering, sprint N+2.

### DATA-SDG-004 — Severity: High
**Finding.** Generator and trained model are same.
**Recommendation.** Different model per §6.5.
**Owner.** AI platform, sprint N+2.

### DATA-SDG-005 — Severity: High
**Finding.** Synthetic data sources not documented.
**Recommendation.** Per §8.6.
**Owner.** data engineering, sprint N+2.

### DATA-SDG-006 — Severity: Medium
**Finding.** Quality validation of synthetic absent.
**Recommendation.** Per §3.5.
**Owner.** eval + data engineering, sprint N+3.

### DATA-SDG-007 — Severity: Medium
**Finding.** Synthetic re-calibration not periodic.
**Recommendation.** Per §7.9.
**Owner.** data engineering, sprint N+3.

### DATA-SDG-008 — Severity: Medium
**Finding.** Diversity not checked.
**Recommendation.** Per §7.5.
**Owner.** data engineering, sprint N+3.

### DATA-SDG-009 — Severity: Medium
**Finding.** Generator prompt versioning absent.
**Recommendation.** Versioned per §3.2 and §8.6.
**Owner.** AI platform, sprint N+3.

### DATA-SDG-010 — Severity: Medium
**Finding.** Synthetic-data audit not done before training.
**Recommendation.** Per §6 and §10.
**Owner.** eval + data engineering, sprint N+4.

### DATA-SDG-011 — Severity: Medium
**Finding.** Hybrid (LLM + human) curation not used.
**Recommendation.** Per §3.9.
**Owner.** AI platform + data engineering, sprint N+4.

### DATA-SDG-012 — Severity: Medium
**Finding.** Bias in synthetic data not audited.
**Recommendation.** Per §7.4.
**Owner.** AI platform + ethics, sprint N+4.

### DATA-SDG-013 — Severity: Medium
**Finding.** Synthetic-data-contamination of training set.
**Recommendation.** Per §6.7 and [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md).
**Owner.** AI platform, sprint N+4.

### DATA-SDG-014 — Severity: Low
**Finding.** Mix-ratio of real vs synthetic not documented.
**Recommendation.** Per §4.7.
**Owner.** data engineering, sprint N+5.

### DATA-SDG-015 — Severity: Low
**Finding.** Annual synthetic-data review absent.
**Recommendation.** Periodic re-evaluation.
**Owner.** data engineering, sprint N+5.

### DATA-SDG-016 — Severity: Low
**Finding.** Adversarial-synthetic-eval absent.
**Recommendation.** Per §5.8.
**Owner.** AI platform + security, sprint N+6.

### DATA-SDG-017 — Severity: Low
**Finding.** Privacy-preserving synthetic data not used where applicable.
**Recommendation.** Per §2.4.
**Owner.** AI platform + privacy, sprint N+6.

### DATA-SDG-018 — Severity: Low
**Finding.** Synthetic-data cost not tracked.
**Recommendation.** Per §8.7.
**Owner.** FinOps + data engineering, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Identify legitimate synthetic use cases per §2.**
- [ ] **Different generator from trained model per §6.5.**
- [ ] **Calibration vs real distribution per §4.**
- [ ] **Quality validation per case per §3.5.**
- [ ] **Mix synthetic with real per §4.7.**
- [ ] **Hybrid curation (LLM + human) per §3.9.**
- [ ] **Documentation per §8.6.**
- [ ] **Periodic re-calibration per §7.9.**
- [ ] **Real-data validation of synthetic eval per §5.5.**
- [ ] **Annual synthetic-data review.**

---

## 12. References

**In this folder.**
- [dataset-versioning.md](./dataset-versioning.md) — synthetic data versioning.
- [data-quality-for-ai.md](./data-quality-for-ai.md) — quality.
- [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md) — preventing synthetic leakage.
- [training-eval-split-discipline.md](./training-eval-split-discipline.md) — discipline.
- [labeling-and-annotation.md](./labeling-and-annotation.md) — alternative.

**Elsewhere in this repo.**
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — eval design.
- [model-lifecycle/fine-tuning-operations.md](../model-lifecycle/fine-tuning-operations.md) — fine-tune that may use synthetic.
- [model-lifecycle/distillation-operations.md](../model-lifecycle/distillation-operations.md) — distillation as synthetic data.

**Sibling repos.**
- [ai-architecture-reference-architecture / context-and-prompt-architecture / few-shot-vs-fine-tune-vs-system-prompt.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/few-shot-vs-fine-tune-vs-system-prompt.md) — synthetic for few-shot.

**External.**
- "Constitutional AI" (Anthropic) — synthetic data for alignment.
- "Self-Instruct" research.
- Distillation literature.
- Synthetic data privacy literature.
