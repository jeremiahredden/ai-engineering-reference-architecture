# RAG Eval Integration

> **Audience.** RAG engineers and tech leads connecting the eval discipline to the production RAG pipeline. The "how do we run eval against this pipeline" owner — distinct from the "how do we design eval cases" owner. **Scope.** The *engineering* practice of pipeline-side eval support — replay patterns, shadow eval, eval-driven tuning, the operational side of running RAG eval. Complements (not duplicates) [eval-of-rag.md](../eval-engineering/eval-of-rag.md) which covers the eval methodology and case design. **Worked client.** Meridian Health.

---

## 1. Why this document exists

[eval-of-rag.md](../eval-engineering/eval-of-rag.md) covers how to design RAG eval cases (case structure, scoring rubric, citation accuracy, faithfulness). This document covers how the RAG engineer *supports* eval from the pipeline side — instrumenting the pipeline so eval can replay it, running eval against pipeline configurations, using eval results to tune the pipeline.

The two perspectives are complementary: the eval engineer designs the eval; the RAG engineer makes the eval possible at the pipeline level. Without the pipeline-side support, eval is theoretical; with it, eval drives concrete pipeline improvements.

This document is opinionated about three things:

1. **The pipeline supports replay.** Captured production retrievals can be re-executed in the eval environment; alternative pipeline configurations can be tested against production traffic.
2. **Eval-driven tuning is the operational mode.** Pipeline parameter changes (HNSW, candidate count, rerank threshold) are eval-validated before deployment; the workflow is structured.
3. **The pipeline tracks eval-suite version on every retrieval.** Production retrievals know which eval suite validated their configuration; mismatch detected and surfaced.

Structure: (2) the pipeline-eval integration patterns; (3) the replay capability; (4) shadow eval for tuning; (5) eval-driven configuration tuning; (6) production sampling for online eval; (7) the eval-suite versioning per-retrieval; (8) integration with the broader eval practice; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The pipeline-eval integration patterns

How the production pipeline and the eval suite interact.

### 2.1 The three integration points

- **Offline eval against the pipeline.** The eval suite runs against the production pipeline (or a test deployment of it); pass/fail per case is recorded.
- **Shadow eval against alternative configurations.** A candidate pipeline change is evaluated in shadow before deployment.
- **Online eval against production traffic.** Sampled production interactions are scored by the judge.

Each point requires different pipeline support.

### 2.2 The offline integration

For offline eval:

- Eval cases are sent to the pipeline (often as test queries).
- Pipeline produces retrievals and answers as if production.
- Eval judge scores the results.
- Results aggregated per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md).

The pipeline support: a test endpoint that accepts eval queries with a flag indicating "this is eval traffic" (so it's separately attributed in cost and observability).

### 2.3 The shadow integration

For shadow eval:

- An alternative pipeline configuration is deployed in shadow (parallel to production).
- Sample production traffic (or eval cases) is replayed against the shadow.
- Shadow results are compared to production.
- Decision to promote the configuration is based on the comparison.

The pipeline support: configuration-version pinning that allows multiple versions to coexist; replay capability per section 3.

### 2.4 The online integration

For online eval:

- Sampled production interactions are scored by the judge (reference-free mode).
- Pipeline emits the data the judge needs (retrieval doc IDs, scores, the generated answer).
- Judge runs asynchronously per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md).

The pipeline support: rich trace observability per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md); the judge consumes the trace.

### 2.5 The integration architecture

The eval system and the pipeline are loosely coupled:

- Eval consumes traces (the production observability).
- Eval invokes the pipeline (for replay and shadow).
- Pipeline emits the data eval needs; doesn't depend on eval running.

Loose coupling means pipeline operation is independent of eval operation; either can be evolved without breaking the other.

---

## 3. The replay capability

Replaying production retrievals is the foundation for shadow eval and post-incident investigation.

### 3.1 The replay shape

```
Capture: production interaction with full context
    │
    ▼
Replay: re-execute in test environment with the captured context
    │
    ▼
Compare: original trace vs replay trace
```

The captured context includes:
- The query.
- The conversation history (for context-aware retrieval).
- The tenant ID and other scope context.
- The corpus version (for reproducibility).

### 3.2 The replay infrastructure

The pipeline supports replay via:

- **Replay endpoint.** A test endpoint that accepts captured context as input; runs the pipeline; returns results.
- **Trace-driven replay.** Tooling that reads a production trace and reconstructs the replay invocation.
- **Configuration overrides.** The replay can run with different pipeline configuration (different HNSW parameters, different rerank threshold) for shadow eval.

### 3.3 The replay scope

Replay can target:
- **Single interaction.** Reproduce one production failure for investigation.
- **Batch.** Replay N interactions; aggregate statistics.
- **Full production sample.** Replay a representative sample of production traffic against the candidate configuration.

Each scope serves different purposes (debugging vs eval vs validation).

### 3.4 The replay accuracy

The replay should produce the same trace as production (within model nondeterminism):

- Same retrieval doc IDs (assuming corpus is unchanged).
- Same scores (assuming retriever parameters unchanged).
- Same context formation (assuming chunking unchanged).
- Generation may vary slightly (model nondeterminism); for eval purposes, this is acceptable.

Significant divergence between replay and original signals a reproducibility issue.

### 3.5 The replay cost

Replay incurs the same per-call cost as production:
- Embedding API calls.
- Reranker API calls.
- Generation API calls (if the replay extends to generation).

For investigation (a few replays): cost is negligible. For full-traffic replay (e.g., 1% of last week's traffic against a new configuration): budget per the eval cost line.

### 3.6 The replay observability

Replay execution emits its own traces (with a flag marking them as replay, not production). The replay trace can be compared to the original.

---

## 4. Shadow eval for tuning

The structured pattern for evaluating pipeline configuration changes.

### 4.1 The shadow eval workflow

When the team considers a pipeline configuration change:

1. **Define the change.** Specific parameters changed (e.g., HNSW efSearch 40 → 32).
2. **Define the eval target.** Which eval suite, which sample of production traffic.
3. **Deploy shadow.** The candidate configuration in a parallel deployment.
4. **Replay.** The eval suite (or production sample) replayed against the shadow.
5. **Compare.** Shadow results vs current production results.
6. **Decision.** Promote, refine, or reject.

### 4.2 The metrics for shadow eval

- **Retrieval recall** (against eval cases).
- **Retrieval precision** (against eval cases).
- **Online judge SLI** (against production sample).
- **Latency.**
- **Cost.**

The decision balances all five.

### 4.3 The traffic-sample for shadow

For production-sample replay:

- Stratified sample (representative of class distribution).
- Sample size: 100-1000 typically (enough for statistical signal).
- Sample period: recent (last 7 days typically).
- Per-tenant sampling: optional (premium tenants may sample more).

### 4.4 The decision criteria

The shadow result drives the decision:

| Result | Decision |
|---|---|
| Quality matches or improves; latency / cost matches or improves | Promote |
| Quality improves; latency / cost degrades acceptably | Promote with cost-line update |
| Quality matches; latency / cost improves | Promote |
| Quality degrades; latency / cost improves | Reject or refine |
| Quality degrades; latency / cost matches | Reject |

Trade-off acceptance is explicit and documented.

### 4.5 The cutover

Once a shadow configuration is promoted:

- Per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md), the configuration becomes a release artifact.
- Canary rollout: small percentage of production traffic first.
- Full rollout after canary validation.
- Shadow is decommissioned.

### 4.6 The shadow as default for tuning

Every pipeline tuning change uses shadow eval. The discipline:

- "We're going to change X" → "what does the shadow eval show?"
- Without shadow data, the change is not approved for production.

The discipline catches regressions before they reach users.

---

## 5. Eval-driven configuration tuning

The workflow that uses eval to inform tuning decisions.

### 5.1 The tuning candidates

Pipeline parameters that benefit from eval-driven tuning:

- HNSW parameters (M, efConstruction, efSearch) per [retrieval-engineering.md](./retrieval-engineering.md) section 7.4.
- Hybrid retrieval weights (BM25 vs vector contribution).
- Merge strategy (RRF vs weighted vs normalized) per [retrieval-engineering.md](./retrieval-engineering.md) section 4.
- Candidate count (top-N from retrievers; top-K to reranker).
- Rerank threshold per [reranking-engineering.md](./reranking-engineering.md) section 5.
- Cache TTL and size.
- Query-rewrite skip thresholds per [query-rewriting.md](./query-rewriting.md) section 3.
- Conditional-rerank thresholds.

Each parameter is tuned via shadow eval.

### 5.2 The grid-search pattern

For multi-parameter tuning:

- Define a grid of candidate parameter combinations.
- For each combination: shadow eval; capture metrics.
- Choose the combination on the cost-quality-latency frontier.

The grid search is bounded (typically 10-30 combinations); each combination is shadow-evaluated.

### 5.3 The single-parameter sweep

For single-parameter tuning:

- Define a range (e.g., HNSW efSearch from 20 to 100 in steps of 10).
- Shadow eval at each value.
- Plot the curve; identify the knee point.

The sweep produces a curve that guides the tuning intuition.

### 5.4 The eval cost of tuning

Each tuning evaluation costs:

- One full eval suite run per configuration: ~$10-50 (per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) cost estimates).
- For a 20-configuration grid: ~$200-1000.

The cost is bounded; tuning is not a runaway expense.

### 5.5 The tuning frequency

Active tuning is more common early:
- First 6 months: weekly tuning cycles.
- Steady state: quarterly tuning review; trigger-based for specific signals.

The frequency declines as the system stabilizes.

### 5.6 The tuning audit

Every tuning change documented:
- Pre-tuning baseline metrics.
- Tuning experiments performed (with results).
- Decision rationale.
- Post-tuning metrics (verifying the change produced the intended effect).

The audit supports reversibility (if the change turned out wrong, the prior state is documented).

---

## 6. Production sampling for online eval

Per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md), production traffic is sampled for online judging.

### 6.1 The pipeline support

The pipeline emits everything the online judge needs:

- The query and conversation context.
- The retrieval results (doc IDs, scores).
- The retrieved chunk content.
- The generated answer.
- The citations.

All via traces; the judge consumes traces; no separate API call required.

### 6.2 The sample selection

Per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 3:

- Random sample (typically 5-15%).
- Stratified across classes.
- Augmented with tail-based criteria.

The pipeline doesn't control sampling; the eval system does. The pipeline produces the traces; the eval system picks which to judge.

### 6.3 The judge invocation

For each selected interaction:

- Eval system invokes the judge with the trace contents.
- Judge produces a verdict.
- Verdict is recorded; aggregate SLI updated.

The pipeline is unaffected by the judge's operation.

### 6.4 The drift detection

If the online judge surfaces a sustained quality drop:

- The pipeline team is alerted per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).
- Investigation per [rag-failure-modes-and-debugging.md](./rag-failure-modes-and-debugging.md).
- Fix; canary; deploy.

The online judge is a continuous quality signal feeding the broader operational discipline.

---

## 7. The eval-suite versioning per-retrieval

Each production retrieval knows which eval suite version validated its configuration.

### 7.1 The tracking

Per [trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md):

- The release manifest includes the eval suite version.
- The pipeline records the active eval suite version on each retrieval trace.
- Mismatch (the deployed pipeline references an old suite) is detectable.

### 7.2 The mismatch alert

If the deployed pipeline's eval suite version differs from the current eval suite:

- Warning issued.
- Investigation: was the pipeline updated but eval suite not? Or vice versa?

Mismatch should be rare; the release process synchronizes both.

### 7.3 The retrospective query

For any past retrieval, the team can ask:

- "Which eval suite validated this retrieval's configuration?"
- "Did that suite include test cases for this question class?"

The retrospective informs incident reviews ("the failed case wasn't covered by the eval suite at the time").

### 7.4 The integration with regression cases

Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md):

- A failed retrieval becomes a regression case.
- The case is added to the suite.
- The suite version bumps.
- The release manifest updates to the new suite version on next deploy.

The cycle ensures the suite grows with the system.

---

## 8. Integration with the broader eval practice

The pipeline-side support enables the broader eval practice.

### 8.1 The eval engineer's view

Per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md), the eval engineer designs cases, calibrates the judge, runs the suite. The pipeline-side support makes this practical.

### 8.2 The RAG engineer's view

This document. The RAG engineer ensures the pipeline produces the data eval needs; supports replay; tunes based on eval results.

### 8.3 The shared workflow

For an eval-driven pipeline improvement:

1. Online judge flags a quality regression.
2. Eval engineer identifies the affected case class; adds golden-set cases.
3. RAG engineer investigates the pipeline; identifies a tuning candidate.
4. RAG engineer shadow-evals the tuning change.
5. Eval engineer validates the shadow result.
6. RAG engineer deploys via canary.
7. Online judge confirms improvement.

Both engineers play roles; the workflow is structured.

### 8.4 The eval-gate integration

Per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md), CI runs eval on PRs. For RAG-related PRs:

- Fast subset includes RAG cases.
- Full suite (nightly) includes the comprehensive RAG suite.
- Release-candidate gate is strict.

The pipeline supports CI eval by accepting eval queries efficiently.

### 8.5 The post-incident eval addition

Per [regression-eval-suites.md](../eval-engineering/regression-eval-suites.md):

- Every fixed RAG bug produces a regression case.
- The case enters the regression suite.
- Subsequent CI runs prevent recurrence.

The integration is bidirectional: pipeline issues feed eval; eval validates pipeline.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The integration shape

Meridian's RAG-eval integration:

- **Offline eval.** `meridian-eval` runs against the production pipeline via a test endpoint; nightly full suite + per-PR fast subset.
- **Shadow eval.** Pipeline configuration changes go through shadow eval before promotion.
- **Online eval.** 10% sampled production interactions judged; SLI feeds dashboards and alerts.
- **Replay.** `meridian-replay` tool reads production traces and replays.

### 9.2 The replay capability

`meridian-replay` is a CLI tool:

```bash
meridian-replay --trace-id t-2026-05-25-3a4b --config-override hnsw_efSearch=32
```

The tool:
- Pulls the trace from the trace backend.
- Reconstructs the retrieval invocation.
- Executes against the test pipeline with the configuration override.
- Compares results to the original trace.

Used for:
- Post-incident investigation (reproducing the failure with the proposed fix).
- Shadow eval (running candidate configurations against production cases).
- Quality regression analysis.

### 9.3 The shadow eval workflow

Recent shadow eval (2026-Q1 HNSW tuning experiment):

1. **Define:** HNSW efSearch change from 40 to 100.
2. **Shadow deploy:** Parallel pgvector configuration.
3. **Replay:** 500 production traces from the prior 7 days.
4. **Compare:**
   - Recall: 91% (vs 90% production); +1 point.
   - Latency p95: 285ms (vs 80ms production); +205ms.
5. **Decision:** Reject. Latency degradation not worth 1-point recall gain.
6. **Document:** Tuning audit log updated.

### 9.4 The eval-driven tuning history

Recent tuning changes (2026):

- 2026-Q1: candidate count tuning. Shadow-evaluated 30, 50, 75, 100. Selected 50 (best balance).
- 2026-Q1: query-rewrite threshold tuning. Shadow-evaluated turn-position thresholds. Selected: rewrite on turn 2+.
- 2026-Q2: rerank threshold tuning. Considered conditional rerank; rejected per [reranking-engineering.md](./reranking-engineering.md) section 9.6.

Each documented; each deployable artifact pins the chosen configuration.

### 9.5 The online judge integration

Production interactions:
- Pipeline emits traces with full retrieval and answer attributes.
- Eval system samples 10% (stratified per class).
- Judge invoked per sampled interaction.
- Verdicts feed the production quality SLI.

For Meridian's volume (~3K interactions/day):
- ~300 judged per day.
- ~$5-15 daily judge cost.
- Quality SLI updated hourly; alerts integrated with on-call.

### 9.6 The eval-suite versioning

Meridian's release manifest:
```yaml
release:
  version: 2026.05.25-r3
  ...
  eval_suite:
    version: 2026-05-15
    pass_rates:
      clinical_golden_set: 95.2
      drug_interaction_subset: 98.1
      conversational_subset: 91.5
```

Each retrieval trace records `ai.eval.suite.version: 2026-05-15`. Mismatch detection ensures pipeline and eval suite stay aligned.

### 9.7 The bidirectional flow

In 2026-Q2:
- Online judge flagged pediatric-CHF cases (per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 9.6).
- Eval engineer added 5 pediatric-CHF golden-set cases.
- RAG engineer investigated: pipeline correctly retrieved general CHF protocols; needed prompt update for pediatric handling.
- Supervisor prompt updated.
- Shadow eval validated.
- Deployed.
- Online judge confirmed improvement.

The cycle: production issue → eval cases → pipeline change → eval validation → deployment → confirmation.

### 9.8 The platform discipline

- Replay capability is a first-class tool.
- All tuning changes shadow-evaluated.
- Online judge runs continuously.
- Eval suite version on every release manifest.
- Bidirectional eval-pipeline workflow established.

---

## 10. Anti-patterns

### 10.1 "Pipeline doesn't support replay"

Investigations require running fresh queries against the pipeline; cannot reproduce specific failures.

**Corrective.** Replay capability per section 3.

### 10.2 "Tuning without shadow eval"

Pipeline parameters changed in production; regressions discovered after-the-fact.

**Corrective.** Shadow eval workflow per section 4.

### 10.3 "Eval suite version not tracked per retrieval"

Released pipeline uses an old eval suite; nobody notices the mismatch.

**Corrective.** Eval-suite version tracking per section 7.

### 10.4 "Online judge data not available to pipeline team"

The online judge runs; results go to a dashboard; the RAG engineer doesn't see them.

**Corrective.** Pipeline team has access to online judge dashboards; integrates into operational reviews.

### 10.5 "Replay reproducibility unverified"

Replay produces different results than production; replay-based investigations are unreliable.

**Corrective.** Replay accuracy validation per section 3.4; investigate divergences.

### 10.6 "Shadow without representative sampling"

Shadow eval uses test cases only, not production sample; the shadow result may not reflect production behavior.

**Corrective.** Production-sample replay per section 4.3.

### 10.7 "Tuning experiments not documented"

Pipeline parameters changed; nobody remembers what was tried before; the team repeats failed experiments.

**Corrective.** Tuning audit log per section 5.6.

### 10.8 "Eval-pipeline integration broken on release"

Pipeline changes break the eval interface; eval pipeline fails until fixed.

**Corrective.** Loose coupling per section 2.5; eval interface as a contract.

---

## 11. Findings (sprint-assignable)

### RAGEVAL-001 — Severity: Critical
**Finding.** Pipeline lacks replay capability; investigations cannot reproduce failures.
**Recommendation.** Build replay per section 3.
**Owner.** ai-platform-eng, sprint N+1.

### RAGEVAL-002 — Severity: Critical
**Finding.** Pipeline tuning ships without shadow eval; regressions in production.
**Recommendation.** Shadow eval workflow per section 4; required for all tuning changes.
**Owner.** ai-platform-eng + sre, sprint N+1.

### RAGEVAL-003 — Severity: High
**Finding.** Eval suite version not tracked per retrieval; mismatches undetected.
**Recommendation.** Per-retrieval suite version per section 7.
**Owner.** ai-platform-eng, sprint N+2.

### RAGEVAL-004 — Severity: High
**Finding.** Online judge data inaccessible to RAG engineering team.
**Recommendation.** Dashboard access; integrate into weekly review per [retrieval-observability.md](./retrieval-observability.md).
**Owner.** ai-platform-eng team lead, sprint N+2.

### RAGEVAL-005 — Severity: High
**Finding.** Replay reproducibility not validated; replay-based investigations unreliable.
**Recommendation.** Reproducibility check per section 3.4.
**Owner.** ai-platform-eng, sprint N+2.

### RAGEVAL-006 — Severity: High
**Finding.** Tuning audit log absent; experiments are repeated.
**Recommendation.** Documented tuning history per section 5.6.
**Owner.** ai-platform-eng, sprint N+2.

### RAGEVAL-007 — Severity: High
**Finding.** Shadow eval uses test cases only; production-sample replay missing.
**Recommendation.** Production-sample replay per section 4.3.
**Owner.** ai-platform-eng, sprint N+3.

### RAGEVAL-008 — Severity: Medium
**Finding.** Eval-pipeline interface not stable; pipeline changes break eval.
**Recommendation.** Loose coupling per section 2.5; interface as a versioned contract.
**Owner.** ai-platform-eng, sprint N+3.

### RAGEVAL-009 — Severity: Medium
**Finding.** Offline eval integration is via direct DB access; eval system bypasses the production pipeline.
**Recommendation.** Eval via test endpoint per section 2.2; same code path as production.
**Owner.** ai-platform-eng, sprint N+3.

### RAGEVAL-010 — Severity: Medium
**Finding.** Grid-search tuning not used; multi-parameter optimization manual.
**Recommendation.** Grid-search workflow per section 5.2.
**Owner.** ai-platform-eng, sprint N+3.

### RAGEVAL-011 — Severity: Medium
**Finding.** Eval-driven post-incident workflow not formalized.
**Recommendation.** Per section 8.3; structured workflow.
**Owner.** ai-platform-eng + sre, sprint N+3.

### RAGEVAL-012 — Severity: Medium
**Finding.** Replay cost not tracked separately; eval-cost line in FinOps muddled.
**Recommendation.** Per-replay cost attribution per section 3.5.
**Owner.** ai-platform-eng + finops, sprint N+4.

### RAGEVAL-013 — Severity: Medium
**Finding.** Sample selection for online eval is uniform; stratification absent.
**Recommendation.** Stratified sampling per section 6.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### RAGEVAL-014 — Severity: Medium
**Finding.** Drift detection on online judge SLI absent; quality regressions detected late.
**Recommendation.** Drift detection per section 6.4; alert integration.
**Owner.** ai-platform-eng + sre, sprint N+4.

### RAGEVAL-015 — Severity: Medium
**Finding.** Bidirectional eval-pipeline workflow ad-hoc; quality issues stall.
**Recommendation.** Structured workflow per section 8.3; weekly review tracks state.
**Owner.** ai-platform-eng team lead + sre, sprint N+4.

### RAGEVAL-016 — Severity: Low
**Finding.** Single-parameter sweep tooling absent; manual sweep is tedious.
**Recommendation.** Automation per section 5.3.
**Owner.** ai-platform-eng, sprint N+5.

### RAGEVAL-017 — Severity: Low
**Finding.** Per-class shadow eval results not visualized; multi-class trade-offs invisible.
**Recommendation.** Per-class shadow result dashboard.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### RAGEVAL-018 — Severity: Low
**Finding.** Integration documentation thin; new engineers don't understand the eval-pipeline relationship.
**Recommendation.** Documentation alongside the integration code.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team integrating eval with the RAG pipeline:

- [ ] **Sprint 0 — inventory.** Existing eval discipline; existing pipeline observability.
- [ ] **Sprint 1 — replay capability.** Build per section 3; validate reproducibility.
- [ ] **Sprint 1 — eval-suite versioning.** Per-retrieval suite version per section 7.
- [ ] **Sprint 2 — shadow eval framework.** Shadow deployment + replay + comparison.
- [ ] **Sprint 2 — tuning discipline.** All tuning changes shadow-evaluated.
- [ ] **Sprint 3 — online judge data access.** Pipeline team has dashboard access.
- [ ] **Sprint 3 — bidirectional workflow.** Structured per section 8.3.
- [ ] **Sprint 4 — production-sample replay.** Stratified production sampling for shadow eval.
- [ ] **Sprint 4 — drift detection.** Online judge SLI drift; alerts.
- [ ] **Sprint 5 — refinement.** Grid-search tooling; per-class visualization.
- [ ] **Ongoing — discipline.** Tuning history; weekly review; eval-pipeline integration health.

A team that completes this sequence has a closed-loop eval-driven pipeline operation. A team that ships pipeline changes without shadow eval pays in production regressions.

---

## 13. References

- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — eval methodology this complements.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — broader eval practice.
- This repo: [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — online judge details.
- This repo: [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — regression-case workflow.
- This repo: [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — CI integration.
- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — what's tuned.
- This repo: [rag-engineering/retrieval-observability.md](./retrieval-observability.md) — observability that supports eval.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](./rag-failure-modes-and-debugging.md) — diagnostic patterns.
- This repo: [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — release manifest pinning.
- This repo: [observability-and-telemetry/trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) — trace structure.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — architecture context.
