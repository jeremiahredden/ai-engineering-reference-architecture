# Reranking Engineering

> **Audience.** Engineers deciding whether to add reranking to their RAG pipeline, or tuning an existing reranker. Tech leads who suspect reranking is "adding cost without lifting quality." **Scope.** The *engineering* practice of reranker selection, placement, calibration, eval-based justification. Pair with [retrieval-engineering.md](./retrieval-engineering.md), [eval-of-rag.md](../eval-engineering/eval-of-rag.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Reranking is the easiest-to-add RAG component that most readily adds cost without lifting quality. The team adds a reranker because the reference architecture had one; the reranker runs on every query; cost increases; quality may or may not improve; the team doesn't measure.

The discipline this document codifies: reranking is eval-validated; the reranker earns its cost or it doesn't ship. Reranker choice, parameter tuning, threshold setting, conditional invocation — all are engineering decisions backed by measurement.

This document is opinionated about three things:

1. **Reranking is justified by measured lift, not by reference-architecture convention.** Many workloads benefit measurably; some don't. The team measures.
2. **The right reranker depends on the workload.** Cohere Rerank is good for many cases; BGE / Voyage rerankers compete; in-context LLM reranking is an option for specific patterns. Pick on signal.
3. **Conditional reranking is a real option.** Reranking everything is the default; reranking only when needed (high candidate-set ambiguity, low retrieval confidence) is sometimes the right pattern.

Structure: (2) what reranking does; (3) reranker options; (4) placement in the pipeline; (5) calibration and threshold setting; (6) the eval validation; (7) cost / latency / quality tradeoffs; (8) conditional reranking; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. What reranking does

Reranking re-orders candidate chunks based on a model's query-document relevance score.

### 2.1 The role

After hybrid retrieval (per [retrieval-engineering.md](./retrieval-engineering.md)):

- Retrieval produces a candidate set (top-50 or so) ordered by retrieval scores.
- Reranker scores each candidate against the query using a query-document relevance model.
- Reranker returns the top-K (typically top-5 or top-10) by rerank score.

The retrieval scores are based on embedding similarity (semantic) and BM25 (lexical); the rerank scores are based on a model that's been trained specifically for query-document relevance. Different model class; different signal.

### 2.2 Why it helps (when it does)

- **Retrieval is recall-oriented.** Maximizes top-K recall; precision at top-K is secondary.
- **Reranking is precision-oriented.** Re-orders to put the most relevant chunks first.

If retrieval recall@50 is high (the right chunk is in the top-50) but precision@10 is low (the right chunk is at rank 23), reranking lifts precision@10 by moving the right chunk up.

### 2.3 When it doesn't help

- **Retrieval is already precision-perfect.** If precision@10 is already 95%, the reranker has little room to improve.
- **The reranker shares the retrieval's bias.** A reranker trained on similar data may make similar errors.
- **The candidate set is small.** If top-50 is the corpus's relevant subset, reranking just re-orders the small set.
- **Workload is highly homogeneous.** Documents in the corpus are similar; relative scoring doesn't differentiate well.

The team measures whether reranking helps on its specific workload.

### 2.4 The lift measurement

Per [eval-of-rag.md](../eval-engineering/eval-of-rag.md):

- Run the eval suite without rerank; measure precision@K.
- Run with rerank; measure precision@K.
- The difference is the rerank lift.

A meaningful lift (5+ percentage points on precision@10) justifies the reranker's cost. A marginal lift (1-2 points) is borderline. No lift = remove the reranker.

---

## 3. Reranker options

Multiple reranker choices; each has trade-offs.

### 3.1 Cohere Rerank

**Versions.** Rerank-3.5 (current production), Rerank-3.0 (older), Rerank-2.0 (legacy).

**Strengths.** Strong general-purpose performance; well-documented; mature API; latency reasonable.

**Cost.** ~$0.001-0.005 per query depending on candidate count and version.

**When right.** Most production RAG workloads. The default rerank choice.

### 3.2 BGE Reranker

**Versions.** BGE-Reranker-Large, BGE-Reranker-Base.

**Strengths.** Open-weight; self-hosted (no external dependency); good performance on English content.

**Cost.** Self-hosted compute cost (GPU instance).

**When right.** Workloads that need self-hosting (data residency, vendor avoidance); teams with GPU infrastructure.

### 3.3 Voyage Rerank

**Versions.** Voyage Rerank-2.

**Strengths.** Strong performance on technical / domain-specific content (per their published benchmarks).

**Cost.** Per-API-call pricing similar to Cohere.

**When right.** Domain-specific workloads where the team validates Voyage outperforms Cohere on their eval.

### 3.4 In-context LLM rerank

**Pattern.** Use an LLM (typically Haiku or Sonnet) to rank candidates by relevance.

**Strengths.** Flexible (can use any LLM); customizable prompt for workload-specific criteria; no separate reranker dependency.

**Cost.** ~$0.005-0.02 per query depending on candidate count.

**When right.** Workloads where ranking criteria are nuanced and benefit from a prompt-tunable model; teams already using an LLM gateway.

### 3.5 Cross-encoder reranking

**Pattern.** Custom-trained cross-encoder (e.g., fine-tuned sentence-transformer).

**Strengths.** Optimized for the specific workload; potentially highest performance.

**Cost.** Training and serving infrastructure; engineering work.

**When right.** Workloads with large training data and clear ROI on training a specialized reranker. Rare in production.

### 3.6 The selection workflow

For a new workload:

1. **Default to Cohere Rerank** (most production workloads).
2. **Measure lift** on the eval suite (per section 6).
3. **If lift is marginal or absent**: investigate why; consider alternatives or no rerank.
4. **If specific concerns** (data residency, vendor avoidance): consider BGE / self-hosted.
5. **If domain-specific outperformance is plausible**: A/B against Voyage or alternative.
6. **Custom cross-encoder**: only when other options exhaust and ROI is clear.

The Meridian default is Cohere Rerank-3.5.

---

## 4. Placement in the pipeline

Reranker placement matters.

### 4.1 The standard placement

```
Retrieve (top-50) → Filter → Rerank (top-10) → Return
```

After retrieval and filtering; before the final return. The reranker sees a filtered candidate set.

### 4.2 The pre-filter vs post-filter rerank

- **Pre-filter rerank.** Rerank then filter. The reranker sees more candidates; some may be filtered out.
- **Post-filter rerank.** Filter then rerank. The reranker sees only candidates passing the filter.

Post-filter is more efficient (reranker processes fewer candidates); the standard pattern.

### 4.3 The candidate count

The reranker receives N candidates and returns K:

- N (candidate count): top-50 is common. Larger N = more potential for the reranker to find a great chunk; larger latency and cost.
- K (return count): top-5 to top-10 typically.

Tuning N: shadow-test with different values; measure precision@K and latency; choose.

### 4.4 The conditional rerank pattern

The reranker doesn't have to run on every query. Conditional patterns:

- **High-confidence retrieval skips rerank.** If the retrieval's top-K scores are well-separated (clear winners), skip rerank.
- **Low-confidence retrieval triggers rerank.** If top-K scores are clustered or tied, rerank to disambiguate.
- **Workload-specific skip.** Some workloads benefit; others don't; the pipeline routes accordingly.

Per section 8, conditional rerank can reduce cost while maintaining quality.

### 4.5 The integration with multi-stage retrieval

For multi-stage retrieval (per [retrieval-engineering.md](./retrieval-engineering.md) section 6), reranking is one stage. The pattern is recall-then-precision: retrieve broadly, rerank to precision.

### 4.6 The post-rerank processing

After reranking, additional processing may apply:

- **Threshold-based cutoff.** Only return candidates with rerank score above a threshold.
- **Diversity selection.** From the top-N reranked, select diverse chunks (one per section).
- **Context-fit selection.** Choose chunks that fit the LLM's context budget.

These are workload-specific.

---

## 5. Calibration and threshold setting

The reranker has tunable parameters that need calibration.

### 5.1 The scoring scale

Reranker scores vary by reranker:

- Cohere Rerank: 0.0-1.0; higher = more relevant.
- BGE Reranker: typically -10 to +10; higher = more relevant.
- LLM-as-reranker: 1-5 or 0-100; depends on the prompt.

Production code shouldn't assume a specific scoring scale; the wrapper normalizes if cross-reranker comparison is needed.

### 5.2 The threshold-based cutoff

For some workloads, only return candidates with high enough rerank scores:

- Threshold 0.7 (for Cohere): only return chunks the reranker considers high-relevance.
- Below threshold: return empty or fewer chunks.

The threshold-based pattern reduces low-relevance noise in the context but may produce empty results when the corpus has no high-relevance content.

Calibration: shadow-test different thresholds; measure precision and empty-rate.

### 5.3 The score interpretation

The team interprets scores carefully:

- A 0.8 score doesn't mean "80% relevant"; it means "more relevant than a 0.7-scored chunk."
- Cross-query comparison of absolute scores can mislead (one query's 0.8 may be another's 0.5).
- Use scores for ranking within a query; not for absolute "relevance" judgments.

### 5.4 The conditional-skip threshold

If retrieval scores are well-separated, reranking adds little. Detection:

- Compute the score gap between retrieval rank 1 and rank K+1 (e.g., rank 11 if returning top-10).
- If the gap is large: top-K is clear; skip rerank.
- If the gap is small: top-K is ambiguous; rerank to disambiguate.

Threshold: workload-specific; calibrated by eval.

### 5.5 The model-version pinning

Reranker model versions matter:

- Cohere Rerank-3.5 is different from Rerank-3.0; scores and rankings differ.
- Per [model-registry.md](../model-lifecycle/model-registry.md), the reranker version is pinned.

Reranker version changes are migration events (eval-validate before adoption).

---

## 6. The eval validation

Reranking decisions are eval-validated.

### 6.1 The lift measurement

Per [eval-of-rag.md](../eval-engineering/eval-of-rag.md):

- For each case in the eval set: run retrieval; capture top-50.
- Run rerank against the top-50; capture top-10 reranked.
- Compare: did the gold-source chunks (per `expected_sources`) end up in the top-10 reranked?
- If the reranked top-10 has higher recall than the retrieval top-10: rerank helped.

### 6.2 The per-case-class breakdown

Reranking may help differently per case class:

- Some classes (where retrieval already has high precision): rerank adds little.
- Other classes (where retrieval needs precision lift): rerank helps substantially.

Per-class measurement informs whether to use conditional rerank per section 8.

### 6.3 The lift threshold for adoption

How much lift justifies reranking?

- Strong lift (10+ points precision@10): clear adoption.
- Moderate lift (5-10 points): adopt; cost is usually worth it.
- Marginal lift (1-5 points): borderline; consider cost; consider conditional rerank.
- No lift: don't adopt.

Meridian's clinical workload sees ~7-12 points lift; adopted.

### 6.4 The cost trade-off

The lift is balanced against:

- Cost: ~$0.001-0.01 per query.
- Latency: ~200-500ms per query.
- Operational complexity: another component to operate, monitor, update.

The trade-off favors rerank for high-stakes workloads (clinical, financial, legal); marginal for low-stakes workloads.

### 6.5 The re-evaluation triggers

The reranker decision is re-evaluated when:

- A new reranker version lands (re-measure lift).
- The retrieval pipeline changes (better retrieval may reduce the rerank lift).
- The corpus changes substantially.
- New eval cases surface workload patterns the original evaluation didn't cover.

Annual re-evaluation as a baseline cadence.

---

## 7. Cost / latency / quality tradeoffs

The three-dimensional optimization.

### 7.1 The cost dimension

Reranker cost per query:

- Cohere Rerank-3.5: $0.001-0.005 per query (depends on candidate count).
- BGE self-hosted: GPU instance cost amortized across queries.
- LLM-as-reranker: $0.005-0.02 per query.

For high-volume features: the per-query cost multiplies. Care Coordinator at ~3K interactions/day with rerank = ~$5-15/day in rerank cost.

### 7.2 The latency dimension

Rerank latency:

- Cohere Rerank: 200-500ms per query.
- BGE self-hosted: depends on GPU; typically 50-200ms.
- LLM-as-reranker: 500-2000ms (LLM-call latency).

Latency is part of the broader retrieval budget. If the retrieval budget is 500ms and the reranker takes 400ms, the rest of retrieval has 100ms.

### 7.3 The quality dimension

The lift in retrieval precision (or downstream answer quality) per section 6.1.

### 7.4 The joint optimization

The team's choice optimizes the joint:

- High-stakes workload (clinical, financial): quality dominates; cost and latency acceptable.
- High-volume, low-stakes: cost dominates; reranking justified only at high lift.
- Latency-critical (real-time chat): latency dominates; quality lift must justify added latency.

The Meridian Care Coordinator (high-stakes, moderate-volume, moderate-latency): rerank earns its cost and latency at the quality lift.

### 7.5 The cost-reduction options

If rerank is justified but cost is too high:

- Reduce candidate count (rerank smaller N).
- Conditional rerank per section 8.
- Cheaper reranker (Cohere Rerank-3.0 instead of 3.5).
- Cache rerank results for repeated queries.

### 7.6 The latency-reduction options

If rerank is justified but latency is too high:

- Parallel execution: run rerank in parallel with other work.
- Reduce candidate count.
- Cheaper reranker.
- Streaming pattern: start LLM generation with retrieval results while rerank finishes in background (only for some patterns).

---

## 8. Conditional reranking

Reranking on every query is the default; reranking only when needed is sometimes better.

### 8.1 The conditional patterns

**Score-gap-based skip.**

```
Compute the score gap between retrieval rank 1 and rank top_K + 1.
If gap > threshold (top-K is clearly the best): skip rerank.
Else (top-K is ambiguous): rerank.
```

Works when the gap measurement correlates with rerank lift.

**Query-class-based dispatch.**

```
Classify the query into classes that benefit from rerank vs not.
For benefit classes: rerank.
For non-benefit classes: skip.
```

Requires a query classifier; works when classes have stable rerank-lift characteristics.

**Confidence-based.**

```
If the retrieval's combined score on top result is high (>threshold): skip rerank.
If low: rerank.
```

Threshold-based pattern; calibrated per workload.

### 8.2 The cost savings

If 60% of queries don't benefit from rerank:

- Without conditional rerank: 100% of queries pay rerank cost. ~$10/day in rerank.
- With conditional rerank: 40% of queries pay; 60% skip. ~$4/day. 60% savings.

Cost savings are workload-dependent.

### 8.3 The quality trade-off

Conditional rerank only works if the conditional-skip cases truly don't benefit:

- Calibration must be accurate; skipping a case that would have benefited produces a quality regression.
- Per [eval-of-rag.md](../eval-engineering/eval-of-rag.md), eval validates the conditional logic.

### 8.4 The implementation

```python
def maybe_rerank(query, retrieved, top_k):
    if should_skip_rerank(query, retrieved):
        return retrieved[:top_k]
    return rerank(query, retrieved, top_k)

def should_skip_rerank(query, retrieved):
    # Score-gap-based:
    if len(retrieved) > top_k:
        gap = retrieved[0].score - retrieved[top_k].score
        if gap > SKIP_THRESHOLD:
            return True
    return False
```

The conditional logic is itself a versioned artifact; calibration is part of its lifecycle.

### 8.5 The observability

The pipeline records whether rerank was applied per query:

- `ai.retrieval.rerank_applied: true | false`
- `ai.retrieval.rerank_skipped_reason: high_confidence | query_class | cost_limit`

Per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md). The metric supports tuning the threshold.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The reranker choice

Meridian uses Cohere Rerank-3.5 for the Care Coordinator clinical retrieval.

- Selected over Rerank-3.0 (3.5 outperforms on clinical eval by ~3 points; cost similar).
- Selected over BGE (BGE self-hosting was operationally expensive for the team's capacity).
- Selected over LLM-as-reranker (per-query cost was ~5x).

The choice was validated by eval comparison; documented.

### 9.2 The placement

```
Hybrid retrieval (top-50)
    │
    ▼ (tenant + content-type filter applied during retrieval, not separately)
Cohere Rerank-3.5
    │
    ▼ (top-10)
Return to clinical-knowledge worker
```

Standard post-filter, pre-context placement.

### 9.3 The lift measurement

Eval comparison (Care Coordinator clinical golden set):

- Retrieval (top-10) without rerank: precision@10 = 71%.
- Retrieval (top-50) → rerank (top-10): precision@10 = 88%.
- Lift: 17 points. Strong adoption signal.

### 9.4 The cost / latency profile

- Per-query cost: ~$0.0025.
- Per-query latency: ~260ms.
- Care Coordinator daily volume: ~3K interactions, ~6K retrieval calls (some interactions have multiple retrieval calls in the agent loop).
- Daily rerank cost: ~$15.
- Latency: rerank is the largest single component of retrieval latency (~57% of total).

### 9.5 The threshold setting

The team doesn't use a strict threshold cutoff (returns top-10 regardless of rerank score). Considered but rejected: clinical workload values having all 10 candidates for the model to reason over.

### 9.6 The conditional rerank consideration

Conditional rerank was evaluated:

- Score-gap-based skip: would have saved ~30% of rerank cost.
- Quality on skipped cases: dropped 4 points on the eval subset.
- Net: cost savings didn't justify the quality drop.

Decision: rerank everything. Re-evaluate if the cost line grows.

### 9.7 The version-pinning

Cohere Rerank-3.5 is pinned per [model-registry.md](../model-lifecycle/model-registry.md). When Cohere released Rerank-4.0 (hypothetically), the team would:

1. Pre-migration eval on sample.
2. If quality matches or exceeds: shadow + cutover.
3. If quality regresses: stay on 3.5.

### 9.8 The 2026-Q1 incident

In 2026-Q1, rerank latency spiked. Investigation:

- Cohere API latency had increased (provider-side issue).
- Pipeline latency exceeded the budget.
- Workaround: reduced candidate count from 50 to 30 (slightly less rerank input).
- Latency normalized.
- Lift dropped from 17 points to 14 points (acceptable trade-off).

After Cohere resolved the latency issue, the team returned to candidate count 50.

### 9.9 The platform discipline

- Reranker pinned per registry.
- Lift measured quarterly; cost vs quality trade-off documented.
- Conditional-rerank evaluated; not adopted (yet).
- Per-query trace records reranker version and scores.
- Re-evaluate when new versions land.

---

## 10. Anti-patterns

### 10.1 "Reranker without measurement"

Team added reranker because reference architecture had one; never measured lift. Cost added without verified quality benefit.

**Corrective.** Eval measurement per section 6.1; adopt only if lift justifies.

### 10.2 "Reranker version unpinned"

Reranker referenced by alias; provider updates change ranking behavior; team can't reproduce.

**Corrective.** Pin per section 5.5.

### 10.3 "Wrong candidate count"

Too few candidates (top-10): reranker has nothing to work with.
Too many (top-200): rerank latency is unjustified.

**Corrective.** Tune per section 4.3; eval-based.

### 10.4 "Pre-filter rerank when post-filter is fine"

Reranker runs on the full retrieval set; results are then filtered. Wasted rerank work.

**Corrective.** Filter then rerank per section 4.2.

### 10.5 "Cross-query score comparison"

Team interprets rerank scores as absolute relevance; compares across queries; misled.

**Corrective.** Scores are per-query; don't compare across queries per section 5.3.

### 10.6 "Strict threshold without empty-rate measurement"

Hard threshold rejects most reranker output; empty-retrieval rate spikes; quality regresses.

**Corrective.** Threshold calibration per section 5.2 with empty-rate as a measured constraint.

### 10.7 "Reranker as fallback for poor retrieval"

Team uses reranker to compensate for bad retrieval; the reranker can't fix what retrieval missed.

**Corrective.** Reranker helps when retrieval has good recall but poor precision; doesn't help when retrieval recall is low. Fix retrieval first.

### 10.8 "Conditional rerank without eval"

Team adds conditional skip; never validates that the skip pattern doesn't drop quality.

**Corrective.** Eval-validate the conditional pattern per section 8.3.

---

## 11. Findings (sprint-assignable)

### RERANK-001 — Severity: High
**Finding.** Reranker in production without measured lift.
**Recommendation.** Measure per section 6.1; remove if no measured lift; adopt if strong lift.
**Owner.** ai-platform-eng, sprint N+1.

### RERANK-002 — Severity: High
**Finding.** Reranker version unpinned.
**Recommendation.** Pin per section 5.5 / [model-registry.md](../model-lifecycle/model-registry.md).
**Owner.** ai-platform-eng, sprint N+1.

### RERANK-003 — Severity: High
**Finding.** Candidate count not tuned; latency or cost suboptimal.
**Recommendation.** Tune per section 4.3.
**Owner.** ai-platform-eng, sprint N+2.

### RERANK-004 — Severity: High
**Finding.** Reranker runs pre-filter when post-filter would be more efficient.
**Recommendation.** Filter then rerank per section 4.2.
**Owner.** ai-platform-eng, sprint N+2.

### RERANK-005 — Severity: High
**Finding.** Reranker observability missing; per-query rerank attribution invisible.
**Recommendation.** Trace attributes per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) section 5.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### RERANK-006 — Severity: High
**Finding.** Reranker cost not tracked separately.
**Recommendation.** Per-query cost attribution per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — reranker as separate cost line.
**Owner.** ai-platform-eng + finops, sprint N+3.

### RERANK-007 — Severity: Medium
**Finding.** Reranker version-change without eval validation.
**Recommendation.** Pre-migration eval per section 5.5; align with [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
**Owner.** ai-platform-eng, sprint N+3.

### RERANK-008 — Severity: Medium
**Finding.** Conditional rerank considered but rejected without eval.
**Recommendation.** Eval-validate per section 8.3 before deciding.
**Owner.** ai-platform-eng, sprint N+3.

### RERANK-009 — Severity: Medium
**Finding.** Threshold-based cutoff causes high empty-retrieval rate.
**Recommendation.** Recalibrate threshold per section 5.2; or remove threshold.
**Owner.** ai-platform-eng, sprint N+3.

### RERANK-010 — Severity: Medium
**Finding.** Reranker selection done by intuition; alternatives not evaluated.
**Recommendation.** Compare alternatives per section 3.6.
**Owner.** ai-platform-eng, sprint N+4.

### RERANK-011 — Severity: Medium
**Finding.** Reranker as fallback for poor retrieval; can't fix what retrieval missed.
**Recommendation.** Fix retrieval first per section 2.3; reranker is precision-not-recall.
**Owner.** ai-platform-eng, sprint N+3.

### RERANK-012 — Severity: Medium
**Finding.** Reranker latency exceeds retrieval budget; causes TTFT regression.
**Recommendation.** Latency reduction per section 7.6; reduce candidate count or change reranker.
**Owner.** ai-platform-eng, sprint N+4.

### RERANK-013 — Severity: Medium
**Finding.** Reranker re-evaluation not scheduled; choice may have drifted.
**Recommendation.** Annual re-evaluation per section 6.5.
**Owner.** ai-platform-eng team lead, sprint N+4.

### RERANK-014 — Severity: Medium
**Finding.** Per-case-class lift not measured; conditional opportunity missed.
**Recommendation.** Per-class measurement per section 6.2.
**Owner.** ai-platform-eng, sprint N+4.

### RERANK-015 — Severity: Low
**Finding.** Reranker failure path not designed; reranker outage cascades to retrieval failure.
**Recommendation.** Fallback per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — return retrieval top-K if rerank fails.
**Owner.** ai-platform-eng, sprint N+4.

### RERANK-016 — Severity: Low
**Finding.** Reranker results not cached for repeated queries.
**Recommendation.** Cache rerank results in the LRU per section 7.5 / retrieval caching.
**Owner.** ai-platform-eng, sprint N+5.

### RERANK-017 — Severity: Low
**Finding.** Cross-encoder option not evaluated; potentially higher-performance reranker missed.
**Recommendation.** Evaluate cross-encoder option per section 3.5 if other options exhaust.
**Owner.** ai-platform-eng, sprint N+5.

### RERANK-018 — Severity: Low
**Finding.** Reranker documentation thin.
**Recommendation.** Document choice rationale, lift measurements, configuration.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team considering reranking:

- [ ] **Sprint 0 — measure baseline.** Run the eval suite without rerank; record precision@K.
- [ ] **Sprint 1 — pilot reranker.** Add a candidate reranker (Cohere Rerank as default); eval-measure lift.
- [ ] **Sprint 1 — adoption decision.** Adopt if lift justifies cost/latency; reject if not.
- [ ] **Sprint 2 — pin and observability.** Pin reranker version; per-query trace observability.
- [ ] **Sprint 2 — candidate-count tuning.** Shadow-test different N values; pick optimal.
- [ ] **Sprint 3 — failure handling.** Reranker fallback if rerank fails.
- [ ] **Sprint 3 — cost attribution.** Per-feature cost tracking includes reranker.
- [ ] **Sprint 4 — conditional consideration.** Per-class lift measurement; evaluate conditional rerank.
- [ ] **Sprint 5 — alternatives.** A/B against alternative rerankers if eval suggests gains.
- [ ] **Ongoing — discipline.** Annual re-evaluation; quarterly cost / latency review.

A team that completes this sequence has reranking that earns its cost. A team that ships rerank by default may be paying for no quality benefit.

---

## 13. References

- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — the retrieval the reranker consumes.
- This repo: [rag-engineering/chunking-engineering.md](./chunking-engineering.md) — chunks the reranker scores.
- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — eval discipline for measuring lift.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — reranker span attributes.
- This repo: [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — reranker registration.
- This repo: [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — reranker pinning.
- This repo: [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — reranker failure fallback.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost circuits include rerank.
- Cohere Rerank documentation.
- BGE Reranker documentation (BAAI).
- Voyage AI Rerank documentation.
