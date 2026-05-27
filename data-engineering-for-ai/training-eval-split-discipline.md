# Training-Eval Split Discipline

> **Audience.** Engineers running fine-tune workloads with train/validation/test splits. Tech leads whose test set has been touched dozens of times during iteration. Anyone whose test scores look great but production performance lags. **Scope.** The *engineering* split discipline: train / validation / test for fine-tune workloads; the held-out test set touched only at release; time-based splits for temporal workloads; split-leakage prevention. Not the broader contamination prevention (see [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md), companion). Not the data quality discipline (see [data-quality-for-ai.md](./data-quality-for-ai.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Fine-tune workloads need split data:

- Train: what the model learns on.
- Validation: what guides hyperparameters during training.
- Test: what measures final quality.

If these aren't properly separated:

- Test scores inflated.
- Production performance differs.
- Researchers wonder why the model is great in eval but mediocre in production.

The discipline:

- Test is held out; touched only at release.
- Validation can be used during training; doesn't determine release.
- Train and validation don't include test data.
- Per data type, appropriate split (random, time-based, group-based).

Without engineering discipline:

- Test gets "leaked" through iterative use.
- Researcher decisions are validated on test, biasing measurement.

This document covers the discipline.

This document is opinionated about four things:

1. **The test set is sacrosanct.** Touch only at release. Otherwise it's just an inflated validation set.
2. **Validation is the iteration target, not test.** Tune against validation; measure on test at the end.
3. **Time-based splits for temporal data.** Production data has implicit time ordering; respect it.
4. **Group-based splits for related data.** Don't put related items in different splits.

Structure: (2) the three splits explained; (3) the held-out discipline; (4) split strategies; (5) leakage prevention; (6) the validation-as-iteration-target; (7) post-release re-split; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The three splits

What each does.

### 2.1 The train set

What the model learns from:

- Largest split (~70-80% of total).
- Diverse, representative.

The model sees this many times.

### 2.2 The validation set

What guides training decisions:

- Used during training (to detect overfitting).
- Used for hyperparameter tuning.
- Used during iteration.

The model "sees" this in the sense that decisions are based on it.

Typical: 10-15% of total.

### 2.3 The test set

The final measurement:

- Touched only at release.
- Not used for training or tuning.
- The performance measure.

Sacrosanct: don't iterate against test.

Typical: 10-15% of total.

### 2.4 The "we have lots of data; can we split 90/5/5" question

For very large datasets:

- 5% test still gives meaningful sample.
- More data for training is good.

For small datasets:

- May need 70/15/15 or similar.
- Smaller splits, larger relative noise.

### 2.5 The "we don't have enough data for three splits" challenge

For data-starved workloads:

- Cross-validation may help (train and validation rotate).
- Held-out test still needed.

Cross-validation doesn't replace test set; complements train + validation.

### 2.6 The "validation and test are the same" mistake

Treating validation as test:

- Iterating against "test" inflates the score.
- Production behavior differs.

Maintain separation.

### 2.7 The split-ratio per workload

```
Workload                 Train   Val   Test
─────────────────────────────────────────────
General fine-tune        70%     15%   15%
Data-starved             60%     20%   20%
Data-abundant            85%     5%    10%
Very-data-abundant       90%     5%    5%
```

Per workload's volume.

### 2.8 The split-recording

Per split:

- Items in each split documented.
- Versioned (cross-link to [dataset-versioning.md](./dataset-versioning.md)).
- Reproducible.

Deterministic splits.

---

## 3. The held-out discipline

How to keep test sacrosanct.

### 3.1 The "touch only at release" rule

Test set:

- Locked at the start of a release cycle.
- Not used until release.
- Re-locked after release.

### 3.2 The "we iterate against validation" workflow

```
Train on train set.
Tune on validation set.
Iterate iteratively.
When done (or near release): run test set ONCE.
Record results.
Don't re-iterate based on test.
```

If test reveals issues: that's the signal to release later (after re-training).

### 3.3 The test-set access control

Physically separate:

- Test set in restricted location.
- Engineers can't accidentally see during iteration.
- Run only by release process.

Engineering discipline.

### 3.4 The "we ran test 5 times" leakage

Each run against test contributes to optimization (even unconsciously):

- After 5 runs: test is no longer trustworthy.
- The "implicit overfit."

Cap test runs per period (e.g., max 2 per quarter).

### 3.5 The "test data leaked into training accidentally" recovery

If detected:

- Test no longer trustworthy.
- Regenerate test (different data).
- Re-eval against new test.

Cross-link to [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md).

### 3.6 The "we never use the test set" deferral

If test is never touched:

- Equally useless.
- The release confidence absent.

Touch once per release; not zero times.

### 3.7 The "we share test set with multiple models" risk

If multiple models use same test:

- Comparing models may be valid.
- But: comparing many models inflates apparent best-of-N performance.

Limit cross-model test sharing for high-stakes comparisons.

### 3.8 The "post-release we re-shuffle" pattern

After release:

- Test set may have leaked through repeated use.
- Re-shuffle: new test from current data.
- Old test can be retired (or used for other purposes).

Refresh.

---

## 4. Split strategies

How to split.

### 4.1 The random split

Default:

- Random sampling.
- Each item assigned to train / val / test.

Simple; works for many workloads.

### 4.2 The time-based split

For temporal data:

- Train: items before time T1.
- Validation: items between T1 and T2.
- Test: items after T2.

Cleaner for production-like data:

- Model is trained on the past.
- Tested on the future.

Models naturally face this in production.

### 4.3 The group-based split

For related items:

- Cases by patient → all patient's cases in one split.
- Avoids leakage (model may "memorize" the patient).

For some workflows, items are clustered by group; split by group.

### 4.4 The stratified split

For class-imbalanced data:

- Maintain class distribution in each split.
- Class A 80%, B 15%, C 5% in train; same in val/test.

Statistical correctness.

### 4.5 The "we mixed strategies" combination

For real workflows:

- Time-based for primary split.
- Stratified within each time block.

Combinable.

### 4.6 The split-strategy per workload

```
Workload                 Strategy
─────────────────────────────────
General fine-tune        Random
Production-like data     Time-based
Per-patient workflows    Group-based (by patient)
Class-imbalanced         Stratified
Time-series              Time-based + walk-forward
```

Per workload.

### 4.7 The "we always use random" simplification

Random works for many cases.

For temporal data, group data, or rare-class data: deeper strategy needed.

### 4.8 The split validation

Per split:

- Verify size proportions.
- Verify class balance (if stratified).
- Verify time boundaries (if time-based).
- Verify group separation (if group-based).

Documented validations.

### 4.9 The cross-fold for validation

Cross-validation:

- Train and validation rotate (k folds).
- More robust hyperparameter tuning.
- Test still separate.

For small validation sets.

### 4.10 The test-set "drawn fresh" pattern

For continuous workflows:

- Each release: new test from production data.
- Avoids same test reuse.

Refresh discipline.

---

## 5. Leakage prevention

Stopping cross-split contamination.

### 5.1 The hash-check between splits

Per item:

- Hash; check across all three splits.
- Same hash in multiple splits → error.

Cross-link to [eval-data-contamination-prevention.md §3](./eval-data-contamination-prevention.md).

### 5.2 The semantic-similarity check

For near-duplicates:

- Embed; compare across splits.
- High similarity in different splits → suspicious.

Cross-link to [data-quality-for-ai.md §5](./data-quality-for-ai.md).

### 5.3 The group-leakage check

For group-based splits:

- Verify no group appears in multiple splits.
- Patients, accounts, devices, etc.

Per workflow.

### 5.4 The time-leakage check

For time-based splits:

- Verify train items predate validation items predate test items.

### 5.5 The CI-gate for splits

Before release:

- Splits validated.
- Hash check.
- Semantic check.
- Time / group check.

Automated.

### 5.6 The "leakage in production data" subtlety

Production data:

- Same user across requests.
- Same case across iterations.

Group by user / case before splitting.

### 5.7 The "we split before knowing about groups" failure

If you split randomly without considering groups:

- User X has 10 cases.
- 3 in train, 4 in val, 3 in test.
- Model "memorizes" user X.

Group-based split prevents.

### 5.8 The "we created groups but didn't enforce" gap

If group-based but not enforced:

- Split says "by patient."
- Implementation has bugs.
- Groups split.

Validate explicitly.

### 5.9 The cross-split label-leakage

For some workflows:

- Test label leaks into training (e.g., feature engineering used test labels).
- Subtle.

Audit features for downstream use of label.

---

## 6. The validation-as-iteration-target

The iteration discipline.

### 6.1 The training loop

```
While iterating:
  Train model on train set.
  Evaluate on validation set.
  Tune hyperparameters / prompt / examples.
  Iterate.

At release:
  Evaluate on test set.
  Record.
  Don't iterate based on test.
```

### 6.2 The "validation perfectly matches test" assumption

Sometimes wrong:

- Validation overfitted to.
- Test reveals different reality.

Don't assume.

### 6.3 The hyperparameter tuning on validation

For tuning:

- Try N hyperparameter combinations.
- Validate each on validation set.
- Pick best.
- Test ONCE.

If test is bad, you can't pick again. The first test result stands.

### 6.4 The "we used test for hyperparameter tuning" mistake

If test was used:

- Test is no longer trustworthy.
- Hyperparameters fit test.

Re-split with new test.

### 6.5 The validation-vs-test divergence diagnostic

If validation says 95% but test says 80%:

- Overfit to validation.
- Implicit leakage.

Investigate.

### 6.6 The "we ran one test; it was bad; we re-tested" failure

Re-running test for the same release cycle inflates:

- First test was the measurement.
- Second is research; for next cycle's release.

Discipline: results from first test stand for this release.

### 6.7 The "we tested multiple models; picked best" inflation

If 10 models tested against test set; best picked:

- Effectively over-fitting through model selection.
- Apparent best > actual best.

Test fewer models; or test against held-out (held-out test).

### 6.8 The validation-as-development-tool

Validation is engineering's tool:

- Iterate freely.
- Track validation scores.
- Improve.

Test is the measurement at the end.

### 6.9 The "validation degraded during iteration" alarm

If validation drops:

- Likely overfit to train.
- Or other issue.

Investigate.

---

## 7. Post-release re-split

After release.

### 7.1 The "test set served its purpose" outcome

After release:

- Test was the measurement.
- Performance recorded.

Test set may be retired (or kept for historical comparison).

### 7.2 The next-release test

For next release:

- New test set drawn (refresh).
- Old test set can be used as additional validation.

Refresh.

### 7.3 The "test data drift" reality

Production shifts:

- Old test may not match current production.
- New test from current data more representative.

### 7.4 The held-out for compliance

Some workflows:

- Retain old test for compliance / audit.
- Even though no longer used for measurement.

Long retention.

### 7.5 The "test set drift" indicator

If test set performance changes over time without retraining:

- Test set itself has drifted (cases re-evaluated by different judges).
- Diagnose.

### 7.6 The cross-release test comparison

If you keep the same test set:

- Comparing across releases is direct.
- But: same test "leaks" through repeated use.

Trade-off.

### 7.7 The "we never refresh test" complacency

Static test:

- Each release run against same test.
- Over time, test no longer represents production.

Refresh periodically.

### 7.8 The "we have many old test sets" archive

Per release:

- Test set archived.
- Useful for retrospective analysis.

Storage.

---

## 8. Worked Meridian example

Meridian's split discipline for fine-tune workloads.

### 8.1 The splits per fine-tune workload

```
document-classification-fine-tune:
  Dataset: 25,000 labeled documents.
  Split:
    Train: 80% (20,000)
    Validation: 10% (2,500)
    Test: 10% (2,500)
  Strategy: stratified by category (15 categories; balanced)
  Locked test until release.

billing-code-fine-tune (specialty practice):
  Dataset: 25,000 (input, code-list) pairs.
  Split:
    Train: 70% (17,500)
    Validation: 15% (3,750)
    Test: 15% (3,750)
  Strategy: time-based (training cases from 2024; eval from 2025-2026)
  Group: by patient (no patient in multiple splits)
  Locked test until release.
```

Per workload.

### 8.2 The held-out discipline

Test sets:

- Stored in restricted location.
- Access via release-tool only.
- Released results documented.

Test runs:

- Per release: ONCE.
- Logged.
- Cap: 2 runs per quarter for test.

### 8.3 The Q1 2026 leak (billing-code workflow)

A feature was iterated against test:

- Engineer ran test 4 times during iteration.
- Test scores improved.
- Production deployment showed scores 6% lower.

Investigation: implicit overfitting to test through repeated use.

Recovery:

- Re-split data (new test from recent production).
- Re-evaluated.
- Scores aligned (close to actual production).

Lesson: enforce "test runs per release" cap.

### 8.4 The CI gate for splits

For each release:

```yaml
release_check:
  splits_validated: yes
  hash_overlap_check: passed (0 overlaps)
  semantic_overlap_check: passed (0 items > 0.95)
  group_separation_check: passed
  time_separation_check (if applicable): passed
  test_set_run_count_this_quarter: 1 (limit 2)
  
status: approved
```

Automated.

### 8.5 The split-refresh

Quarterly:

- Production data added.
- New test cases drawn from latest production.
- Old test archived.

Refresh cadence.

### 8.6 The "we have 2 test sets" parallel test pattern

For some workflows:

- Locked test (from start of release cycle).
- Always-fresh test (from latest production; rolling).

Locked: official release measurement.
Fresh: ongoing health metric.

Combined.

### 8.7 The split-leak prevention

Cross-link to [eval-data-contamination-prevention.md §3.6](./eval-data-contamination-prevention.md):

- Hash + semantic + group checks.
- Pre-deployment.

Integrated.

### 8.8 The lessons

- Test set discipline is hard but essential.
- Validation is where iteration happens.
- Refresh test set periodically.
- CI gate enforces splits.

---

## 9. Anti-patterns

### 9.1 The validation-as-test

**Pattern.** Validation called "test"; iterated against. Inflated.

**Corrective.** Three separate splits per §2.

### 9.2 The "we never refresh test" stagnation

**Pattern.** Static test; drift.

**Corrective.** Refresh per §4.10.

### 9.3 The "test is touched whenever convenient" laxity

**Pattern.** Test run repeatedly; not capped.

**Corrective.** Strict cap per §3.4.

### 9.4 The random-split-for-temporal-data

**Pattern.** Production data; random split; data leaks across time.

**Corrective.** Time-based per §4.2.

### 9.5 The patient-leakage

**Pattern.** Same patient's cases in multiple splits; model memorizes.

**Corrective.** Group-based per §4.3.

### 9.6 The unstratified-class-imbalanced

**Pattern.** Rare classes under-represented in some splits.

**Corrective.** Stratified per §4.4.

### 9.7 The "many models tested; best picked" inflation

**Pattern.** N models against test; best released; effectively over-fits.

**Corrective.** Limit tests per §6.7.

### 9.8 The retest-after-bad-result

**Pattern.** First test bad; iterate; re-test. Inflated.

**Corrective.** First test stands per §6.6.

### 9.9 The label-leakage through features

**Pattern.** Feature engineering uses label info; test corrupts.

**Corrective.** Audit features per §5.9.

### 9.10 The split-not-versioned

**Pattern.** Splits change over time; old splits unrecoverable.

**Corrective.** Versioning per §2.8 and [dataset-versioning.md](./dataset-versioning.md).

---

## 10. Findings (sprint-assignable)

### DATA-TES-001 — Severity: Critical
**Finding.** Validation and test are the same.
**Recommendation.** Separate per §2.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-TES-002 — Severity: Critical
**Finding.** Test set run multiple times per release.
**Recommendation.** Strict cap per §3.4.
**Owner.** AI platform, sprint N+1.

### DATA-TES-003 — Severity: High
**Finding.** Split strategy not workload-appropriate.
**Recommendation.** Per §4.6.
**Owner.** AI platform + data engineering, sprint N+2.

### DATA-TES-004 — Severity: High
**Finding.** Group-leakage in production data splits.
**Recommendation.** Group-based per §4.3.
**Owner.** data engineering, sprint N+2.

### DATA-TES-005 — Severity: High
**Finding.** Test set not refreshed.
**Recommendation.** Periodic refresh per §4.10.
**Owner.** data engineering, sprint N+2.

### DATA-TES-006 — Severity: High
**Finding.** No CI gate for split validation.
**Recommendation.** Per §5.5.
**Owner.** AI platform + data engineering, sprint N+2.

### DATA-TES-007 — Severity: High
**Finding.** Hash check across splits absent.
**Recommendation.** Per §5.1.
**Owner.** data engineering, sprint N+2.

### DATA-TES-008 — Severity: Medium
**Finding.** Semantic-similarity check absent.
**Recommendation.** Per §5.2.
**Owner.** data engineering, sprint N+3.

### DATA-TES-009 — Severity: Medium
**Finding.** Time-based separation not enforced for temporal data.
**Recommendation.** Per §4.2.
**Owner.** data engineering, sprint N+3.

### DATA-TES-010 — Severity: Medium
**Finding.** Test runs not logged.
**Recommendation.** Per §3.4 and §8.2.
**Owner.** data engineering, sprint N+3.

### DATA-TES-011 — Severity: Medium
**Finding.** Splits not versioned.
**Recommendation.** Per §2.8.
**Owner.** data engineering, sprint N+3.

### DATA-TES-012 — Severity: Medium
**Finding.** Stratified split absent for class-imbalanced data.
**Recommendation.** Per §4.4.
**Owner.** data engineering, sprint N+4.

### DATA-TES-013 — Severity: Medium
**Finding.** Cross-validation for small validation sets absent.
**Recommendation.** Per §4.9.
**Owner.** AI platform + data engineering, sprint N+4.

### DATA-TES-014 — Severity: Medium
**Finding.** Multi-model test inflation.
**Recommendation.** Per §6.7.
**Owner.** AI platform, sprint N+4.

### DATA-TES-015 — Severity: Low
**Finding.** Label-leakage through features.
**Recommendation.** Audit features per §5.9.
**Owner.** data engineering, sprint N+5.

### DATA-TES-016 — Severity: Low
**Finding.** Test set archive absent.
**Recommendation.** Per §7.8.
**Owner.** data engineering, sprint N+5.

### DATA-TES-017 — Severity: Low
**Finding.** Cross-release test comparison documented but discipline not enforced.
**Recommendation.** Per §7.6.
**Owner.** AI platform, sprint N+6.

### DATA-TES-018 — Severity: Low
**Finding.** Parallel-test (locked + fresh) not used.
**Recommendation.** Per §8.6.
**Owner.** AI platform + data engineering, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Define three-split structure per §2.**
- [ ] **Choose split strategy per workload per §4.**
- [ ] **Implement CI gate for split validation per §5.5.**
- [ ] **Hash + semantic checks per §5.**
- [ ] **Strict test-set discipline per §3.**
- [ ] **Test-run logging per §8.2.**
- [ ] **Test set refresh per §4.10.**
- [ ] **Group-based split for grouped data per §4.3.**
- [ ] **Time-based split for temporal data per §4.2.**
- [ ] **Stratified split for imbalanced classes per §4.4.**

---

## 12. References

**In this folder.**
- [dataset-versioning.md](./dataset-versioning.md) — split versioning.
- [data-quality-for-ai.md](./data-quality-for-ai.md) — quality.
- [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md) — contamination prevention.
- [synthetic-data-generation.md](./synthetic-data-generation.md) — synthetic data splits.
- [data-contracts-for-ai.md](./data-contracts-for-ai.md) — data contracts.

**Elsewhere in this repo.**
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — eval (test set) design.
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — regression eval.
- [model-lifecycle/fine-tuning-operations.md](../model-lifecycle/fine-tuning-operations.md) — fine-tune workflow.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — release pinning.

**External.**
- Stratified-sampling literature.
- Time-series cross-validation literature.
- k-fold cross-validation references.
- Machine-learning hyperparameter-tuning best practices.
