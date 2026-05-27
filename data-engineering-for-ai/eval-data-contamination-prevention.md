# Eval Data Contamination Prevention

> **Audience.** Engineers whose eval pass rate is high but production performance is lower. Tech leads who suspect their eval cases got into training data somehow. Anyone who has never explicitly checked for contamination. **Scope.** The *engineering* controls that prevent eval cases from leaking into training, fine-tuning, few-shot examples, or retrieval: hash-based separation; time-based separation; periodic contamination audit. Not the contamination detection (see [data-quality-for-ai.md §4](./data-quality-for-ai.md), companion). Not the broader data quality (see [data-quality-for-ai.md](./data-quality-for-ai.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

If training data contains eval cases (or near-duplicates), the eval doesn't measure model quality — it measures memorization. Production performance differs from eval, sometimes dramatically.

Contamination paths:

- Direct: eval cases copy-pasted into training.
- Indirect: same source feeds both (web scrape).
- Time-based: training cut-off includes eval cases.
- Few-shot: eval-like examples in prompt.
- Retrieval: eval queries match retrieval corpus.

Without engineered prevention: contamination accumulates silently.

This document covers the prevention controls.

This document is opinionated about four things:

1. **Contamination is the default; prevention is engineering.** Assume it's there unless verified clean.
2. **Engineered separation is the answer, not goodwill.** "We won't accidentally contaminate" doesn't hold over time.
3. **Time-based separation is the most robust.** Eval from time periods after training cut-off.
4. **Audit periodically; don't trust prior verification.** New data sources, new processes; re-audit.

Structure: (2) the contamination paths; (3) hash-based separation; (4) time-based separation; (5) source-based separation; (6) periodic audit; (7) the response when contamination is found; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The contamination paths

How contamination happens.

### 2.1 Direct copy

Eval case appears verbatim in training set:

- Someone copy-pasted.
- Or scripts inadvertently included.

Hash detection catches.

### 2.2 Indirect via shared source

Both training and eval drawn from same source:

- Public web data.
- Internal database.

Some overlap inevitable.

### 2.3 Near-duplicate

Eval case paraphrased / slightly modified:

- "Same content, different words."
- Semantic similarity catches.

### 2.4 Time-based via crawl

Training data cut-off includes eval test cases:

- Web scrape post-eval creation.
- Eval cases now in training.

Time-based separation prevents.

### 2.5 Few-shot leakage

Eval-like examples used as few-shot in prompts:

- Model has seen similar at inference time.

Few-shot examples curated separately.

### 2.6 Retrieval-corpus leakage

Eval queries match retrieval-corpus documents:

- Model retrieves "the answer" from corpus.
- Test isn't really testing reasoning.

Time-based or source-based separation.

### 2.7 The accidental synthetic-leakage

Synthetic data generation:

- Generator may have been trained on eval cases.
- Outputs include eval-like content.

Different generator (or check).

### 2.8 The user-submitted-data leakage

If user-submitted data feeds back:

- Customer provides labeled examples (intentionally).
- Or user queries inadvertently match eval cases.

Audit; isolate.

### 2.9 The "we don't know how this happened" mystery

Sometimes contamination is detected without clear path:

- Investigate possibilities.
- Engineer prevention regardless.

---

## 3. Hash-based separation

Exact-match prevention.

### 3.1 The hash pattern

Maintain hashes:

- All eval cases hashed (SHA-256 or similar).
- Per training-data ingestion: check against hashes.
- Match → exclude from training.

### 3.2 The hash storage

```yaml
care-coordinator-eval-hashes:
  - sha256:a3f7b2c... # case_001
  - sha256:b8e9c1d... # case_002
  ...
```

Per eval set; maintained.

### 3.3 The training-pipeline check

In CI for training data:

```python
def filter_training_data(training_data, eval_hashes):
    filtered = []
    for item in training_data:
        if hash(item) not in eval_hashes:
            filtered.append(item)
    return filtered
```

Pre-training filter.

### 3.4 The hash precision

For exact match: any minor difference (whitespace, encoding) defeats hash.

Normalize before hashing:

- Strip whitespace.
- Lowercase.
- Normalize Unicode.

Then hash.

### 3.5 The "hash check passes but still contaminated" gap

Hash catches exact:

- Doesn't catch near-duplicates.
- Doesn't catch paraphrases.
- Combine with semantic checks per §3.6.

### 3.6 The semantic-check complement

Near-duplicate detection:

- Embed each item.
- Compare to eval embeddings.
- Similarity > 0.95 → exclude.

Cross-link to [data-quality-for-ai.md §5.3](./data-quality-for-ai.md).

### 3.7 The hash-based audit

Periodically:

- Hash check across all known training corpora.
- Compare to eval hashes.
- Detect overlaps.

Catches accumulated contamination.

### 3.8 The vector-of-hashes

For multi-shot eval:

- Hash each shot.
- All hashes excluded from training.

Comprehensive.

### 3.9 The hash-maintenance discipline

When eval changes:

- Update hashes.
- Re-distribute to training pipelines.

CI integration.

---

## 4. Time-based separation

The most robust prevention.

### 4.1 The time-based approach

Training data: only from periods <= cut-off date.

Eval cases: created after cut-off date.

Model can't have seen eval.

### 4.2 The "real production data" advantage

Production data has natural timestamps:

- User submitted Mar 15, 2026.
- Cut-off: Mar 1, 2026.
- Eval case: drawn from post-cut-off.

Clean separation.

### 4.3 The application to web data

For web-scraped training data:

- Cut-off date for crawl.
- Eval cases from sources post-crawl.

Or: eval from sources never crawled.

### 4.4 The "we use the same date as the model's training" mistake

Don't use the model provider's training cut-off:

- That data was used during model training (already).
- Use a later date for fine-tuning + eval.

### 4.5 The rolling cut-off

For continuous learning:

- Cut-off moves forward.
- Eval pool shifts; new eval cases drawn from new time period.

Maintained.

### 4.6 The "we don't have continuous data" challenge

Without continuous data:

- Manually maintain time-separation.
- Mark eval as "from period X-Y."
- Training data: "from period A-B" where A and B don't overlap with X-Y.

Engineering discipline.

### 4.7 The "we created eval; it might be in training" suspicion

If unsure:

- Audit: are eval cases in training?
- Hash + semantic check.

Cross-link to §3.6 and §3.7.

### 4.8 The "eval data is old; need newer" maintenance

Over time:

- Production shifts.
- Eval drawn from older period may not match current production.

Add newer eval cases per period.

### 4.9 The combined time + hash

For full protection:

- Time-based primary.
- Hash check as backup.

Defense in depth.

---

## 5. Source-based separation

Different data sources.

### 5.1 The "eval source is distinct" pattern

Eval cases come from:

- Source A (e.g., "eval-only feedback log").

Training data comes from:

- Source B (e.g., "production data").

Never overlap.

### 5.2 The eval source design

Set up:

- Customer feedback channel → eval-only.
- Production data → training-only.

Engineering separation by source.

### 5.3 The "production data into eval" rule

Production data:

- May go into training.
- Does NOT go into eval.

Exception: explicit "real production eval samples" separate from training (typically a holdout split).

### 5.4 The "feedback into training" rule

Customer feedback / labels:

- Go into training.
- Don't go into eval (would create confusion).

### 5.5 The retrieval-corpus separation

Eval queries:

- Don't match retrieval corpus exactly.
- Or: retrieval corpus excludes documents that match eval queries.

### 5.6 The source-tag

Per data item, tag:

```yaml
item:
  content: "..."
  source: "production_data"  # or "eval_feedback", "labeling_vendor", etc.
  ingested_at: "..."
```

Tags determine destination (training / eval / etc.).

### 5.7 The "we mixed sources by accident" recovery

If sources mixed:

- Audit by source tag.
- Re-separate.
- May need to re-label data with proper source.

### 5.8 The cross-source dedup

If sources may overlap:

- Cross-source hash check.
- Document the resolution.

### 5.9 The "we accept some cross-source overlap" risk

If overlap is unavoidable:

- Document.
- Accept limitations.
- Re-evaluate workflows.

---

## 6. Periodic audit

Verification.

### 6.1 The audit cadence

Quarterly (or per-major-release):

- Hash check across training and eval.
- Semantic similarity check.
- Time-based verification.

### 6.2 The hash audit

```python
def audit_hashes(training_corpora, eval_sets):
    eval_hashes = set()
    for eval_set in eval_sets:
        eval_hashes.update(hash(item) for item in eval_set)
    
    contaminations = []
    for corpus in training_corpora:
        for item in corpus:
            if hash(item) in eval_hashes:
                contaminations.append((corpus, item))
    return contaminations
```

Exact-match check.

### 6.3 The semantic audit

```python
def audit_semantic(training_corpora, eval_sets, threshold=0.95):
    contaminations = []
    eval_embeddings = {...}  # pre-computed
    
    for corpus in training_corpora:
        for item in corpus:
            item_embedding = embed(item)
            for eval_id, eval_emb in eval_embeddings.items():
                if cosine_similarity(item_embedding, eval_emb) > threshold:
                    contaminations.append((corpus, item, eval_id))
    return contaminations
```

Near-duplicate check.

### 6.4 The audit-results action

If contaminations found:

- Quantify (how many; what fraction).
- Diagnose (how did it happen).
- Remediate (remove from training / re-train / regenerate eval).

### 6.5 The "low contamination is acceptable" calibration

Some contamination is unavoidable:

- < 0.1%: probably fine; document.
- 0.1-1%: investigate; mitigate.
- > 1%: significant; eval may be unreliable.

Per-organization threshold.

### 6.6 The cross-corpus audit

For multi-feature platforms:

- Multiple training corpora.
- Multiple eval sets.

Audit each pair: training corpus × eval set.

### 6.7 The audit log

Per audit:

```yaml
audit:
  date: 2026-05-15
  training_corpora: [list]
  eval_sets: [list]
  hash_contaminations: 0
  semantic_contaminations: 3 (similarity 0.96-0.97; investigated; flagged for review)
  remediation: removed flagged items from training; re-trained
```

Documented; tracked.

### 6.8 The audit-vs-iterate cycle

Detect → remediate → re-audit:

- Contamination found.
- Removed.
- Audit again to verify.

Iterative.

### 6.9 The "we audit but never act" risk

Audit without remediation:

- Same contamination cycles.

Each finding requires action.

---

## 7. The response when contamination is found

What happens when audit detects contamination.

### 7.1 The severity classification

- **Low.** Few cases (< 0.1%); semantic-similarity-only.
- **Medium.** 0.1-1%; exact-match or high-similarity.
- **High.** > 1%; significant overlap.

Per finding.

### 7.2 The low-severity response

- Document the finding.
- Sample-check for validity.
- Continue if acceptable.

### 7.3 The medium-severity response

- Remove contaminated items from training.
- Re-train model.
- Re-eval.
- Document.

Standard cycle.

### 7.4 The high-severity response

- Treat as eval integrity issue.
- Eval may be unreliable.
- Consider regenerating eval entirely.
- Don't trust prior eval results.

Major correction.

### 7.5 The "we found contamination 6 months in" recovery

If long-running contamination:

- Past eval results may be invalidated.
- Re-eval against clean dataset.
- Compare claims to reality.

May reveal that quality has been over-reported.

### 7.6 The customer-facing communication

If contamination affected production:

- "Re-evaluating performance with cleaner data."
- "Findings shared with stakeholders."

Honest.

### 7.7 The "we're not certain it caused issues" subtlety

Sometimes contamination found but production seemed fine:

- May not have affected production behavior much.
- Or: production was already strong despite eval inflation.

Document; remain vigilant.

### 7.8 The contamination-prevention strengthening

Post-finding:

- Why did this happen?
- What engineering control would have prevented?
- Implement.

Iterative improvement.

### 7.9 The "this is the third time" pattern

If contamination recurs:

- Process gap; not just engineering.
- Audit + culture change.

Root cause.

---

## 8. Worked Meridian example

Meridian's contamination prevention practice.

### 8.1 The eval and training separation

```
Care Coordinator eval golden set:
  Source: SME-curated from real cases (anonymized).
  Cut-off: cases from 2025-Q1 (pre-fine-tune period).
  Hash-tracked.

Document classification training:
  Source: production-ingested + vendor-labeled.
  Cut-off: only items pre-2026 used.
  
Pre-train check:
  Hash check against Care Coordinator eval: 0 matches.
  Semantic check (similarity > 0.95): 2 items found; investigated; paraphrases of eval cases.
  Removed from training.
  Re-trained.
```

Defense in depth.

### 8.2 The quarterly audit (Q2 2026)

Audit ran:

- All training corpora.
- All eval sets.
- Hash and semantic checks.

Results:

- Hash contamination: 0.
- Semantic contamination: 12 items above 0.95 (across all training and all evals).
- 12 investigated; 12 confirmed paraphrases (not direct copies).

Action: removed 12 items from training; re-trained affected model; verified clean.

### 8.3 The Q1 2026 contamination scare

Earlier audit (Q1):

- Found 12 cases above 0.95 similarity (different from Q2).
- Initial concern: high.
- Investigation: paraphrases, but importantly different in content.
- Decision: not removed (not actual contamination).

Iterative: threshold may need tuning.

### 8.4 The contamination-prevention process

For each ML pipeline:

```
Training data ingestion →
  Hash check against eval hashes →
    Semantic check against eval embeddings →
      Items above threshold flagged for review →
        Approved items removed; logged →
          Cleaned training data proceeds.
```

CI integration; automatic.

### 8.5 The source-based separation

Meridian's data tagged:

- production_data → training-eligible.
- eval_feedback → eval-only.
- labeling_vendor_output → training-eligible.
- synthetic_eval → eval-only.

Source tag determines destination.

### 8.6 The Q3 2026 retrieval-contamination concern

For Care Coordinator:

- Eval cases ask "what's the dosing for X drug?"
- Retrieval corpus includes formulary with same drug.

Is this contamination?

- Strict interpretation: yes (information available in retrieval).
- Pragmatic: no (workflow tests retrieval ability).

Decision: keep; document as "tests retrieval, not memorization."

### 8.7 The "we discovered older contamination" Q1 2026 incident

A new eval was created in Q1; included some cases that turned out to overlap with old training data.

Specifically: 4 cases were paraphrases of customer-provided labeled data.

Investigation:

- Cases removed from eval.
- Re-eval cleaner.
- Score declined 1-2 points (within statistical variance).

Lesson: vet new eval cases against existing training data.

### 8.8 The cross-team alignment

Multiple teams produce data:

- Engineers ingest production data.
- Data team labels.
- Eval team curates.
- Synthetic team generates.

Each team aware of cross-contamination risk; tagging in place.

### 8.9 The infrastructure cost

- Hash storage / management: ~$50/month.
- Semantic embedding store for contamination checks: ~$200/month.
- Engineering for pipeline checks: built once; minimal ongoing.
- SME audits: ~5 hours/quarter.

Total: ~$500/quarter; high value.

### 8.10 The benefits

- Confidence in eval results.
- Production performance matches eval.
- Compliance / audit readiness.
- Cross-team awareness.

### 8.11 The lessons

- Audit periodically; not just at first.
- Hash + semantic checks combined.
- Source-tag everything.
- Threshold tuning: 0.95 is conservative; some items need review.

---

## 9. Anti-patterns

### 9.1 The "we just won't be careless" trust

**Pattern.** No engineered separation; rely on individual carefulness. Contamination inevitable.

**Corrective.** Engineered controls per §3, §4, §5.

### 9.2 The hash-only check

**Pattern.** Only exact-match check; misses paraphrases.

**Corrective.** Combine with semantic per §6.3.

### 9.3 The "we audited once; it's fine" complacency

**Pattern.** First audit clean; never re-audited.

**Corrective.** Quarterly per §6.1.

### 9.4 The "low overlap is fine" rationalization

**Pattern.** Found 2% contamination; "probably fine"; never removed.

**Corrective.** Quantify; act per §6.5 and §7.

### 9.5 The synthetic-eval contamination

**Pattern.** Synthetic eval generated by same model being trained.

**Corrective.** Different generator per [synthetic-data-generation.md §6.5](./synthetic-data-generation.md).

### 9.6 The shared-source-no-separation

**Pattern.** Training and eval both from same source; overlap.

**Corrective.** Source-based separation per §5.

### 9.7 The "eval is from internet; training is from internet" web-leakage

**Pattern.** Both from web scrapes; overlap likely.

**Corrective.** Time-based separation per §4 or domain-specific source.

### 9.8 The "we found contamination; we'll fix later" deferral

**Pattern.** Audit found contamination; remediation deferred. Cleanup never happens.

**Corrective.** Remediation per §7.

### 9.9 The audit-without-remediation

**Pattern.** Audit results filed; no action.

**Corrective.** Each finding → action per §6.9.

### 9.10 The "eval drift is normal" misattribution

**Pattern.** Eval scores drifting; team assumes natural drift; actually contamination accumulating.

**Corrective.** Audit when scores drift unexpectedly.

---

## 10. Findings (sprint-assignable)

### DATA-EDC-001 — Severity: Critical
**Finding.** No contamination check before training.
**Recommendation.** Hash + semantic check per §3.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-EDC-002 — Severity: Critical
**Finding.** No periodic audit.
**Recommendation.** Quarterly per §6.1.
**Owner.** data engineering + eval, sprint N+1.

### DATA-EDC-003 — Severity: Critical
**Finding.** Source tagging absent.
**Recommendation.** Per §5.6.
**Owner.** data engineering, sprint N+1.

### DATA-EDC-004 — Severity: High
**Finding.** Time-based separation not enforced.
**Recommendation.** Per §4.
**Owner.** data engineering, sprint N+2.

### DATA-EDC-005 — Severity: High
**Finding.** Hash check only exact-match.
**Recommendation.** Combine with semantic per §6.3.
**Owner.** data engineering, sprint N+2.

### DATA-EDC-006 — Severity: High
**Finding.** Synthetic eval generated by same model.
**Recommendation.** Different generator per §9.5.
**Owner.** AI platform, sprint N+2.

### DATA-EDC-007 — Severity: High
**Finding.** Audit log absent.
**Recommendation.** Per §6.7.
**Owner.** data engineering, sprint N+2.

### DATA-EDC-008 — Severity: Medium
**Finding.** Remediation workflow absent post-detection.
**Recommendation.** Per §7.
**Owner.** AI platform + data engineering, sprint N+3.

### DATA-EDC-009 — Severity: Medium
**Finding.** Cross-source dedup absent.
**Recommendation.** Per §5.8.
**Owner.** data engineering, sprint N+3.

### DATA-EDC-010 — Severity: Medium
**Finding.** Threshold for semantic similarity untuned.
**Recommendation.** Per §6.5 and §8.3.
**Owner.** AI platform + eval, sprint N+3.

### DATA-EDC-011 — Severity: Medium
**Finding.** New-eval-vetting absent.
**Recommendation.** Per §8.7.
**Owner.** eval, sprint N+3.

### DATA-EDC-012 — Severity: Medium
**Finding.** Retrieval-corpus contamination not considered.
**Recommendation.** Per §2.6.
**Owner.** AI platform, sprint N+3.

### DATA-EDC-013 — Severity: Medium
**Finding.** Cross-team awareness absent.
**Recommendation.** Per §8.8.
**Owner.** data engineering + AI platform, sprint N+4.

### DATA-EDC-014 — Severity: Medium
**Finding.** Customer-data feedback could contaminate eval.
**Recommendation.** Source-tag customer data per §5.6.
**Owner.** AI platform + product, sprint N+4.

### DATA-EDC-015 — Severity: Low
**Finding.** Contamination metrics not tracked over time.
**Recommendation.** Per §6.7.
**Owner.** observability + data engineering, sprint N+5.

### DATA-EDC-016 — Severity: Low
**Finding.** Pre-deployment contamination check absent.
**Recommendation.** CI gate; release blocked if contamination > threshold.
**Owner.** AI platform, sprint N+5.

### DATA-EDC-017 — Severity: Low
**Finding.** Annual contamination review absent.
**Recommendation.** Annual audit.
**Owner.** data engineering, sprint N+6.

### DATA-EDC-018 — Severity: Low
**Finding.** Customer-facing communication of contamination findings undocumented.
**Recommendation.** Per §7.6.
**Owner.** product + customer success, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Implement hash check per §3.**
- [ ] **Implement semantic check per §6.3.**
- [ ] **Source tagging per §5.6.**
- [ ] **Time-based separation per §4.**
- [ ] **Pre-training audit per §3.7.**
- [ ] **Quarterly audit per §6.1.**
- [ ] **Audit log per §6.7.**
- [ ] **Remediation workflow per §7.**
- [ ] **Cross-team awareness per §8.8.**
- [ ] **New-eval-vetting per §8.7.**

---

## 12. References

**In this folder.**
- [data-quality-for-ai.md](./data-quality-for-ai.md) — broader contamination detection.
- [dataset-versioning.md](./dataset-versioning.md) — versioning eval sets.
- [training-eval-split-discipline.md](./training-eval-split-discipline.md) — split discipline.
- [synthetic-data-generation.md](./synthetic-data-generation.md) — synthetic eval risk.
- [labeling-and-annotation.md](./labeling-and-annotation.md) — labeled data discipline.

**Elsewhere in this repo.**
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — eval design.
- [model-lifecycle/fine-tuning-operations.md](../model-lifecycle/fine-tuning-operations.md) — fine-tune that depends on clean data.

**Sibling repos.**
- [ai-architecture-reference-architecture / data-architecture-for-ai / data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) — data contracts architecture.

**External.**
- Contamination research in NLP / ML.
- "Memorization" research (memorize vs generalize).
- Hash and semantic dedup techniques.
