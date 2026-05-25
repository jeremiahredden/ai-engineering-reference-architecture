# Retrieval Instrumentation

> **Audience.** Engineers building or refactoring the retrieval observability for a RAG system. Tech leads who have been asked "why didn't the model use the right source?" and could not answer from the trace. **Scope.** The *engineering* pattern for instrumenting retrieval calls — span structure, attribute capture, integration with the trace framework and the retrieval-client wrapper. Pair with [trace-and-span-design.md](./trace-and-span-design.md), [llm-call-instrumentation.md](./llm-call-instrumentation.md), and [agent-step-instrumentation.md](./agent-step-instrumentation.md). Not the architectural retrieval-pattern decision (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `reference-patterns/rag-architecture-decision-guide.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Retrieval is where most production RAG quality regressions originate, and where most quality regressions are hardest to diagnose. The model produced a wrong answer; was that because retrieval returned the wrong chunks, or because retrieval returned the right chunks and the model ignored them, or because the right chunks were in the corpus but never made it into top-K? Without retrieval-specific instrumentation, the team cannot tell.

The pattern this document describes: every retrieval call emits a span with structured attributes — what was queried, what was returned, what scores, what corpus version, what embedding model, what tenant filter. The trace becomes a debuggable record of the retrieval; quality investigations read the trace and arrive at a diagnosis without re-running the system.

This document is opinionated about three things:

1. **The retrieval span is structurally separable from the LLM-call span.** When an LLM call's prompt was built from retrieved content, the retrieval is its own span (likely a sibling or parent of the LLM span). Conflating them loses the ability to investigate retrieval-specific issues.
2. **Per-call attributes capture the full retrieval context** — query (or query hash for sensitive workloads), query rewrite if any, retrieved doc IDs with scores broken out per-retriever and per-reranker, corpus version, embedding model version, applied filters. The trace tells the full story.
3. **The trace, not new experiments, is the debugging surface.** When a quality regression is investigated, the captured trace is the primary evidence. Re-running the system produces variable output; the trace is the actual record of what happened.

Structure: (2) the retrieval-span structure; (3) attribute taxonomy; (4) hybrid retrieval instrumentation; (5) reranker instrumentation; (6) query-rewrite instrumentation; (7) integration with the retrieval-client wrapper; (8) sensitive-data handling; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The retrieval-span structure

A retrieval call produces a span. Sub-retrievers (e.g., BM25 + vector in a hybrid retrieval) produce sub-spans. Reranking is a sub-span. Query rewriting (when used) is a sub-span.

### 2.1 The span hierarchy

```
span: retrieval               (the overall retrieval call as seen by the caller)
├── span: query_rewrite       (if query rewriting was applied)
├── span: retrieval:bm25      (sub-retriever)
├── span: retrieval:vector    (sub-retriever)
└── span: rerank              (if reranking was applied)
```

The `retrieval` span is the parent. Each technical step is a sub-span. The trace reads top-down as the retrieval pipeline.

### 2.2 The placement in the broader trace

Retrieval spans typically sit:

- **Inside a workflow stage span** for workflow-shaped features.
- **Inside an agent turn span** for agent-shaped features (the retrieval is one tool call's worth of work within the turn).
- **Inside an agent tool-call sub-span** when retrieval is invoked via a tool registration.

Wherever they sit, the parent context is part of the trace hierarchy; the retrieval is contextualized by what invoked it.

### 2.3 The single-shot vs multi-call distinction

A single retrieval call (one user query → one retrieval execution) produces one `retrieval` span. A workflow with multiple retrievals (e.g., agentic RAG with several retrieval iterations) produces multiple `retrieval` spans within the same parent. The order in the trace is the order of execution; each span is independently inspectable.

---

## 3. Attribute taxonomy

The attributes the wrapper populates on every retrieval span.

### 3.1 Universal retrieval attributes (every retrieval span)

| Attribute | Meaning | Example |
|---|---|---|
| `ai.trace.interaction_id` | The user-visible interaction | `interaction-2026-05-25-a7b8` |
| `ai.tenant.id` | Tenant context | `mercy-cleveland` |
| `ai.feature.id` | Which feature initiated the call | `care-coordinator` |
| `ai.user.id` | The acting user | `u-mercy-cleveland-rn-001234` |
| `ai.retrieval.id` | Unique ID for this retrieval | `ret-2026-05-25-a7b8-1` |

These flow from the standard universal attributes per [trace-and-span-design.md](./trace-and-span-design.md) section 4.1.

### 3.2 Query attributes

| Attribute | Meaning |
|---|---|
| `ai.retrieval.query` | The query string (or hash for sensitive workloads — see §8) |
| `ai.retrieval.query_hash` | SHA-256 hash of the query string |
| `ai.retrieval.query_rewrite_applied` | Boolean — was the query rewritten? |
| `ai.retrieval.query_rewrite_original` | If rewritten, the original query |
| `ai.retrieval.query_rewriter_version` | Version of the query rewriter prompt / model |

### 3.3 Corpus and scope attributes

| Attribute | Meaning |
|---|---|
| `ai.retrieval.corpus_id` | Which corpus was queried |
| `ai.retrieval.corpus_version` | Corpus version at time of query |
| `ai.retrieval.tenant_filter` | The tenant filter that was applied (defense-in-depth visibility) |
| `ai.retrieval.scope.per_record` | If per-record scope was applied, the rule |
| `ai.retrieval.scope.content_type` | Content-type scope |
| `ai.retrieval.scope.time_window` | Time-window scope, if applied |
| `ai.retrieval.scope.sensitivity_max` | Sensitivity-level ceiling |
| `ai.retrieval.scope_verification` | `all_passed` / `<failure>` |

### 3.4 Retriever-config attributes

| Attribute | Meaning |
|---|---|
| `ai.retrieval.retriever_type` | `bm25` / `vector` / `hybrid` / `graph` / `multi-stage` |
| `ai.retrieval.embedding_model` | When vector retrieval is used |
| `ai.retrieval.embedding_model_version` | Pinned version |
| `ai.retrieval.top_k_requested` | The top-K requested |
| `ai.retrieval.candidate_count` | How many candidates the underlying store returned (pre-rerank) |
| `ai.retrieval.returned_count` | Final returned count after reranking / filtering |

### 3.5 Result attributes

| Attribute | Meaning |
|---|---|
| `ai.retrieval.doc_ids` | Array of returned doc IDs |
| `ai.retrieval.scores` | Array of final scores (per-doc, post-rerank if applicable) |
| `ai.retrieval.scores_bm25` | Per-doc BM25 scores (for hybrid) |
| `ai.retrieval.scores_vector` | Per-doc vector scores (for hybrid) |
| `ai.retrieval.scores_rerank` | Per-doc rerank scores (if reranked) |
| `ai.retrieval.empty_result` | Boolean — did retrieval return zero results? |
| `ai.retrieval.empty_reason` | If empty, the reason class (no_matches / filter_too_restrictive / corpus_empty) |

### 3.6 Latency attributes

| Attribute | Meaning |
|---|---|
| `ai.retrieval.latency.total_ms` | Total wall-clock latency |
| `ai.retrieval.latency.embed_ms` | Time to embed the query (if vector retrieval) |
| `ai.retrieval.latency.search_ms` | Time for the underlying search |
| `ai.retrieval.latency.rerank_ms` | Time for reranking |
| `ai.retrieval.latency.query_rewrite_ms` | Time for query rewriting (if applied) |

### 3.7 Cost attributes

| Attribute | Meaning |
|---|---|
| `ai.cost.usd` | Total retrieval cost (rerank API + embedding API + compute) |
| `ai.cost.usd.embed` | Embedding-call cost (if vector retrieval) |
| `ai.cost.usd.rerank` | Reranker cost |

### 3.8 The attribute discipline

- Names follow `ai.retrieval.*` namespace.
- Cardinality-aware: full query and full doc-content do not go in attributes (see §8); doc IDs and hashes do.
- The wrapper enforces; consumers do not invent new attribute names ad-hoc.

---

## 4. Hybrid retrieval instrumentation

Hybrid retrieval combines multiple retrievers (typically BM25 + vector); each produces its own sub-span; the merging step produces the final result.

### 4.1 The sub-span structure

```
span: retrieval (parent)
├── span: retrieval:bm25
│   attributes:
│     ai.retrieval.retriever_type: bm25
│     ai.retrieval.scores: [doc_ids and bm25 scores]
│     ai.retrieval.latency.search_ms: 45
├── span: retrieval:vector
│   attributes:
│     ai.retrieval.retriever_type: vector
│     ai.retrieval.embedding_model: text-embedding-3-large
│     ai.retrieval.embedding_model_version: 2024-01-25
│     ai.retrieval.scores: [doc_ids and vector scores]
│     ai.retrieval.latency.embed_ms: 12
│     ai.retrieval.latency.search_ms: 78
└── span: rerank
    attributes:
      ai.retrieval.rerank_model: cohere-rerank-3.5
      ai.retrieval.rerank_model_version: 2025-09-01
      ai.retrieval.scores: [final scores after rerank]
      ai.retrieval.latency.rerank_ms: 320
```

Each sub-span is independently inspectable. Investigations of "why did this hybrid retrieval do X" inspect the relevant sub-span.

### 4.2 The merging instrumentation

The merging step (RRF, weighted, or score-normalized) produces its own attributes on the parent `retrieval` span:

| Attribute | Meaning |
|---|---|
| `ai.retrieval.merge_strategy` | `rrf` / `weighted` / `score_normalized` |
| `ai.retrieval.merge_parameters` | Strategy parameters (e.g., RRF k value, weights for weighted merge) |
| `ai.retrieval.bm25_only_count` | Of the final results, how many came only from BM25 |
| `ai.retrieval.vector_only_count` | How many came only from vector |
| `ai.retrieval.both_count` | How many were ranked by both |

These attributes support analysis like "what fraction of our retrievals are getting value from each retriever" — useful for deciding whether the hybrid setup is earning its operational cost.

### 4.3 Parallel execution

Hybrid retrievers typically run in parallel. The sub-spans show the parallelism (overlapping start times); the parent span's latency is the max of the parallel sub-spans plus the merging step.

---

## 5. Reranker instrumentation

When a reranker is in the pipeline, it produces its own sub-span with reranker-specific attributes.

### 5.1 The rerank sub-span

| Attribute | Meaning |
|---|---|
| `ai.retrieval.rerank_model` | Reranker model identifier |
| `ai.retrieval.rerank_model_version` | Pinned version |
| `ai.retrieval.rerank_input_count` | How many candidates were sent to the reranker |
| `ai.retrieval.rerank_output_count` | How many were returned |
| `ai.retrieval.scores_rerank` | Per-doc rerank scores |
| `ai.retrieval.rerank_threshold` | If threshold-based cutoff was applied |
| `ai.retrieval.latency.rerank_ms` | Latency |
| `ai.cost.usd.rerank` | Cost |

### 5.2 The rerank-lift signal

For each retrieval, the team can compute the *rerank lift* — how much the reranker shifted the doc order from the pre-rerank ranking. The lift signal:

- High lift → the reranker is doing meaningful work; its cost is earned.
- Low lift → the reranker is mostly agreeing with the pre-rerank ranking; its cost may not be justified.

The rerank-lift dashboard surfaces this per feature per workload class.

### 5.3 The conditional-rerank pattern

Some pipelines skip reranking when the pre-rerank top-K is already high-confidence (top scores well-separated from lower scores). When this pattern is used, the trace records whether reranking was applied:

| Attribute | Meaning |
|---|---|
| `ai.retrieval.rerank_applied` | Boolean |
| `ai.retrieval.rerank_skipped_reason` | If skipped: `high_confidence` / `cost_budget` / `disabled` |

The skip-rate dashboard helps tune the high-confidence threshold.

---

## 6. Query-rewrite instrumentation

When query rewriting is applied (typically for conversational follow-ups), the rewriting step produces a sub-span.

### 6.1 The query-rewrite sub-span

| Attribute | Meaning |
|---|---|
| `ai.retrieval.query_rewrite_original` | The original query |
| `ai.retrieval.query_rewrite_result` | The rewritten query |
| `ai.retrieval.query_rewrite_strategy` | `decomposition` / `expansion` / `context-aware` / `LLM-rewrite` |
| `ai.retrieval.query_rewrite_model` | Model used for rewriting (if LLM-rewrite) |
| `ai.retrieval.query_rewrite_prompt_version` | Prompt version (if LLM-rewrite) |
| `ai.retrieval.latency.query_rewrite_ms` | Latency |
| `ai.cost.usd.query_rewrite` | Cost |

### 6.2 The rewrite-lift signal

Similar to rerank-lift: did the rewrite improve retrieval recall over the original query? Tracked by:

- Same query class scored against rewrite-applied retrievals vs rewrite-skipped retrievals.
- A/B comparison surfaces whether the rewrite is earning its cost.

### 6.3 Failure handling

Query rewriting can fail (model returns malformed output, model times out). The wrapper captures:

| Attribute | Meaning |
|---|---|
| `ai.retrieval.query_rewrite_success` | Boolean |
| `ai.retrieval.query_rewrite_failure_class` | If failed: `parse_error` / `timeout` / `validation_failure` |

On failure, the wrapper falls back to the original query and proceeds with retrieval. The trace records both the attempted rewrite and the fallback.

---

## 7. Integration with the retrieval-client wrapper

The retrieval-client wrapper (per the architecture sibling's [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md)) is the chokepoint for retrieval. Instrumentation lives in the wrapper.

### 7.1 The wrapper's instrumentation responsibilities

When the wrapper's `retrieve` method is called:

1. Open the `retrieval` span; populate universal attributes from CallContext.
2. Resolve scope dimensions; populate scope attributes.
3. If query rewriting is configured for this call: open `query_rewrite` sub-span; execute; populate attributes.
4. For each sub-retriever (BM25, vector, graph): open sub-span; execute; populate attributes.
5. Merge results; populate merge attributes on the parent span.
6. If reranking: open `rerank` sub-span; execute; populate attributes.
7. Post-retrieval verification (scope verification per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md)); populate verification attribute.
8. Populate result attributes (doc IDs, final scores, returned count, empty result if applicable).
9. Populate latency and cost attributes.
10. Close the span.

The wrapper is the single source of instrumentation. Tool implementations that call the wrapper do not need their own retrieval instrumentation; the wrapper has done it.

### 7.2 The trace context propagation

The wrapper receives the parent trace context as part of CallContext. It opens its span as a child of the current span (typically a tool-call span or a workflow stage span). The hierarchy is naturally maintained by OpenTelemetry context propagation.

### 7.3 Failure spans

When retrieval fails (provider error, timeout, store unavailable):

- The span is closed with `error` status.
- An `ai.retrieval.error.kind` attribute classifies the failure.
- An `ai.retrieval.error.recovery` attribute records what the wrapper did (returned empty, returned fallback, raised exception).

The trace shows the failure clearly; the caller's failure handling is traced as well.

---

## 8. Sensitive-data handling

Retrieval involves sensitive content by default — the query may include sensitive terms; retrieved chunks contain sensitive material. The wrapper handles this per the sensitivity classification from [trace-and-span-design.md](./trace-and-span-design.md) section 7.

### 8.1 Query handling

- **Routine queries:** stored verbatim in the trace.
- **Sensitive queries (PHI / PII):** stored as hash in the trace; full text in the redirected sink.
- **Regulated queries:** never stored in the trace; only the hash and a placeholder go in.

The classification is per workload. The Meridian Care Coordinator's clinical retrieval treats queries as sensitive (PHI-flag implied).

### 8.2 Result handling

- **Doc IDs:** always in the trace (low cardinality, low sensitivity — IDs are not the content).
- **Doc content:** never in the trace (high cardinality, often high sensitivity); accessible via lineage links per [lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md).
- **Scores:** always in the trace.
- **Source metadata** (publication date, source name): in the trace; sensitive metadata redacted.

### 8.3 The redirected sink

For sensitive content that does need to be queryable later (queries from regulated workloads), the redirected sink stores the full content with stricter access controls. The trace contains the linkage; the content is fetched on authorized access.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The retrieval trace shape

For a typical clinical-knowledge retrieval inside the Care Coordinator (per [meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) section 5.1):

```
span: retrieval (parent)
  attributes:
    ai.trace.interaction_id: interaction-2026-05-25-a7b8
    ai.tenant.id: mercy-cleveland
    ai.feature.id: care-coordinator
    ai.user.id: u-mercy-cleveland-rn-001234
    ai.retrieval.id: ret-2026-05-25-a7b8-1
    
    ai.retrieval.query_hash: 9c2a1f8b...
    ai.retrieval.query_rewrite_applied: false
    
    ai.retrieval.corpus_id: clinical-guidelines-and-protocols
    ai.retrieval.corpus_version: 2026-Q2-1
    ai.retrieval.tenant_filter: "tenant_id IN ('mercy-cleveland', 'global-guidelines')"
    ai.retrieval.scope_verification: all_passed
    
    ai.retrieval.retriever_type: hybrid
    ai.retrieval.top_k_requested: 10
    ai.retrieval.candidate_count: 50
    ai.retrieval.returned_count: 10
    
    ai.retrieval.merge_strategy: rrf
    ai.retrieval.bm25_only_count: 2
    ai.retrieval.vector_only_count: 3
    ai.retrieval.both_count: 5
    
    ai.retrieval.doc_ids: [...]
    ai.retrieval.scores: [...]
    ai.retrieval.empty_result: false
    
    ai.retrieval.latency.total_ms: 487
    ai.cost.usd: 0.0027
  
  ├── span: retrieval:bm25
  │     attributes:
  │       ai.retrieval.retriever_type: bm25
  │       ai.retrieval.candidate_count: 50
  │       ai.retrieval.scores_bm25: [per-doc BM25 scores]
  │       ai.retrieval.latency.search_ms: 45
  │
  ├── span: retrieval:vector
  │     attributes:
  │       ai.retrieval.retriever_type: vector
  │       ai.retrieval.embedding_model: text-embedding-3-large
  │       ai.retrieval.embedding_model_version: 2024-01-25
  │       ai.retrieval.candidate_count: 50
  │       ai.retrieval.scores_vector: [per-doc vector scores]
  │       ai.retrieval.latency.embed_ms: 12
  │       ai.retrieval.latency.search_ms: 78
  │       ai.cost.usd.embed: 0.0001
  │
  └── span: rerank
        attributes:
          ai.retrieval.rerank_model: cohere-rerank-3.5
          ai.retrieval.rerank_model_version: 2025-09-01
          ai.retrieval.rerank_input_count: 50
          ai.retrieval.rerank_output_count: 10
          ai.retrieval.scores_rerank: [per-doc final scores]
          ai.retrieval.latency.rerank_ms: 320
          ai.cost.usd.rerank: 0.0025
```

The trace tells the full story: hybrid retrieval merging BM25 + vector with RRF; rerank pulled top-10 from top-50; tenant scope verified; no empty result; ~$0.003 cost; ~490ms latency.

### 9.2 A diagnostic walkthrough

Suppose a clinician reports a wrong answer. The investigation:

1. Pull the interaction trace by interaction ID.
2. Find the retrieval span(s) within the trace.
3. Inspect the returned doc IDs.
4. Compare against expected doc IDs from the eval golden set (if the case matches one).
5. If the expected docs are not in the returned list: retrieval failure. Inspect the bm25 and vector sub-spans to see if either retriever had the expected doc in their candidate set.
6. If the expected docs are in the returned list but rank low: rerank issue. Inspect the rerank scores.
7. If retrieval was correct: generation issue. Move to the LLM-call span analysis.

Each step uses the trace's attributes. No re-running.

### 9.3 The dashboards

The team's retrieval dashboards aggregate from these span attributes:

- **Retrieval latency** (p50 / p95 / p99) per feature per retriever type.
- **Retrieval recall** (when eval-sampled) per feature per question class.
- **Empty-retrieval rate** per feature per corpus.
- **Rerank-lift** per feature.
- **Hybrid merge composition** (bm25-only vs vector-only vs both) per corpus.
- **Cost per retrieval** per corpus per retriever type.
- **Scope-verification failure rate** (should be near zero; spikes page on-call).

Each dashboard surfaces a different operational concern.

### 9.4 The 2026-Q1 retrieval debugging story

In 2026-Q1, the team identified a quality regression in cardiology questions. The diagnosis used retrieval traces:

1. Sampled traces for cardiology-class questions showed the expected cardiology guideline docs were retrieved.
2. Rerank sub-spans showed the cardiology docs were re-ranked to lower positions after rerank.
3. Investigation of the rerank: the rerank prompt had been changed to emphasize procedural detail; cardiology guideline docs were lower in procedural detail than other clinical content.
4. Fix: revert the rerank prompt change; eval-validate; deploy.

Without the per-sub-span attribute capture, the diagnosis would have been "rerank is doing something weird" — useful but not actionable. With them: actionable.

### 9.5 The platform discipline

- Every retrieval call goes through the wrapper; the wrapper emits the standardized span.
- Span attribute schema is versioned; changes go through review.
- Dashboards consume from the standard attributes.
- Quarterly review of the dashboards; alert thresholds calibrated.

---

## 10. Anti-patterns

### 10.1 "Retrieval is logged, not traced"

Retrieval activity goes to application logs as text entries. Spans do not exist. Investigating "why did the model use these chunks" requires correlating log entries by timestamp.

**Corrective.** Spans with structured attributes. Investigators read the trace.

### 10.2 "Single span for hybrid retrieval"

The trace shows one `retrieval` span with no sub-structure. The BM25 vs vector contribution is not visible.

**Corrective.** Sub-spans per retriever. Each independently inspectable.

### 10.3 "Doc content in trace attributes"

Retrieved chunk content is captured as span attributes. Cardinality is high; storage cost is high; sensitive content is in the trace backend.

**Corrective.** Doc IDs in trace; content via lineage links if needed.

### 10.4 "No rerank attributes"

The reranker runs but the rerank step is not separately instrumented. Rerank-lift cannot be measured; rerank issues cannot be diagnosed.

**Corrective.** Rerank sub-span with input count, output count, scores, latency, cost.

### 10.5 "Query is not captured"

The query is not in the trace (or is in an inconsistent field). Diagnosing "the model answered the wrong question" requires reconstructing the query from upstream context.

**Corrective.** Query attribute (or query hash for sensitive queries) on every retrieval span.

### 10.6 "Embedding model version unrecorded"

The retrieval span shows scores but not which embedding model produced them. When the embedding model changes, the team cannot correlate.

**Corrective.** `ai.retrieval.embedding_model` + `ai.retrieval.embedding_model_version` per section 3.4.

### 10.7 "Empty-retrieval rate ignored"

Empty retrievals (zero results returned) are treated as normal. The dashboard does not surface the rate. A drift in empty-retrieval rate signals a real issue (filter too restrictive, corpus stale, query rewriter broken) that goes unnoticed.

**Corrective.** Empty-retrieval rate as an SLI. Alert on baseline-deviation.

### 10.8 "Scope verification not in the trace"

The wrapper verifies scope but does not record the verification result on the span. Cross-tenant incident investigations require correlation with separate scope-violation logs.

**Corrective.** `ai.retrieval.scope_verification` on every retrieval span.

---

## 11. Findings (sprint-assignable)

### OBS-RET-001 — Severity: Critical
**Finding.** Retrieval activity is logged as application log entries; spans with structured attributes do not exist.
**Recommendation.** Implement retrieval spans per section 2; populate attribute taxonomy per section 3; integrate with trace framework.
**Owner.** ai-platform-eng, sprint N+1.

### OBS-RET-002 — Critical
**Finding.** Hybrid retrieval produces a single retrieval span with no per-retriever sub-structure.
**Recommendation.** Per-retriever sub-spans per section 4; each with its own scores and latency.
**Owner.** ai-platform-eng, sprint N+1.

### OBS-RET-003 — High
**Finding.** Retrieved doc content is captured in trace attributes; cardinality and storage cost are high.
**Recommendation.** Doc IDs in trace; content via lineage links per [lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md).
**Owner.** ai-platform-eng, sprint N+2.

### OBS-RET-004 — High
**Finding.** Reranker step is not separately instrumented; rerank-lift cannot be measured.
**Recommendation.** Rerank sub-span per section 5; rerank-lift dashboard.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### OBS-RET-005 — High
**Finding.** Query is not captured in the retrieval span; diagnosing "model answered the wrong question" requires correlation.
**Recommendation.** `ai.retrieval.query` or `ai.retrieval.query_hash` per section 3.2.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-RET-006 — High
**Finding.** Embedding model version is not recorded on the retrieval span; embedding-model changes are not correlatable.
**Recommendation.** `ai.retrieval.embedding_model` + version per section 3.4.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-RET-007 — High
**Finding.** Empty-retrieval rate is not surfaced as a dashboard or SLI; quality regressions tied to empty retrievals go unnoticed.
**Recommendation.** Empty-retrieval rate SLI; alert on baseline-deviation.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### OBS-RET-008 — High
**Finding.** Scope verification result is not on the retrieval span; cross-tenant investigations require correlation with separate logs.
**Recommendation.** `ai.retrieval.scope_verification` attribute per section 3.3.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### OBS-RET-009 — High
**Finding.** Query rewriting steps are not instrumented; rewrite-lift cannot be measured; rewrite failures are not diagnosable.
**Recommendation.** Query-rewrite sub-span per section 6.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-RET-010 — Medium
**Finding.** Hybrid merge composition (BM25-only vs vector-only vs both) is not tracked; team cannot assess hybrid effectiveness.
**Recommendation.** Merge-composition attributes per section 4.2; dashboard.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### OBS-RET-011 — Medium
**Finding.** Corpus version is not recorded on the retrieval span; correlations with corpus changes are manual.
**Recommendation.** `ai.retrieval.corpus_version` per section 3.3.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-RET-012 — Medium
**Finding.** Per-doc scores are aggregated; per-retriever scores in hybrid retrieval are lost in the aggregate.
**Recommendation.** Per-retriever scores per section 3.5; rerank scores separately.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-RET-013 — Medium
**Finding.** Retrieval cost is not separately tracked per retrieval call; cost-per-feature analysis requires correlation.
**Recommendation.** `ai.cost.usd.*` attributes per section 3.7.
**Owner.** ai-platform-eng + finops, sprint N+3.

### OBS-RET-014 — Medium
**Finding.** Retrieval failures are recorded as generic errors; failure-class diagnosis is manual.
**Recommendation.** `ai.retrieval.error.kind` + `ai.retrieval.error.recovery` per section 7.3.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-RET-015 — Medium
**Finding.** Sensitive-data handling for queries is inconsistent across features; some traces contain PHI.
**Recommendation.** Sensitivity classification per section 8; PHI queries hashed in trace, full content in redirected sink.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### OBS-RET-016 — Medium
**Finding.** Conditional-rerank decisions are not recorded; team cannot tune the high-confidence threshold.
**Recommendation.** `ai.retrieval.rerank_applied` + `rerank_skipped_reason` per section 5.3.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-RET-017 — Low
**Finding.** Trace-content schema versioning is informal; consumers cannot tell which schema a trace was emitted with.
**Recommendation.** `ai.trace.schema_version` per [trace-and-span-design.md](./trace-and-span-design.md).
**Owner.** ai-platform-eng, sprint N+4.

### OBS-RET-018 — Low
**Finding.** Retrieval span documentation is thin; new engineers do not understand the attribute taxonomy.
**Recommendation.** Generate attribute documentation; include in onboarding.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team without retrieval instrumentation:

- [ ] **Sprint 0 — design.** Confirm the trace framework (per [trace-and-span-design.md](./trace-and-span-design.md)) is in place. Decide the attribute schema for retrieval.
- [ ] **Sprint 1 — primary retrieval span.** Emit the parent retrieval span with universal attributes and basic result attributes (doc IDs, scores, latency).
- [ ] **Sprint 1 — wrapper integration.** All retrieval flows through the retrieval-client wrapper; the wrapper is the span source.
- [ ] **Sprint 2 — sub-spans.** Per-retriever sub-spans (BM25, vector); rerank sub-span; query-rewrite sub-span (if used).
- [ ] **Sprint 2 — scope verification on trace.** Surface scope-verification result; integrate with security alerting.
- [ ] **Sprint 3 — cost on retrieval span.** Per-retrieval cost; integrate with cost-attribution pipeline.
- [ ] **Sprint 3 — dashboards.** Latency, empty-rate, rerank-lift, hybrid-composition dashboards.
- [ ] **Sprint 3 — alerting.** Empty-retrieval rate, scope-verification failures, latency regressions.
- [ ] **Sprint 4 — sensitive-data handling.** Query hashing for sensitive workloads; redirected sink.
- [ ] **Sprint 5 — quarterly review.** Dashboard relevance; threshold calibration; documentation refresh.

A team that completes this sequence has the retrieval observability that turns retrieval-quality investigations from "let's re-run and guess" into "let's read the trace."

---

## 13. References

- OpenTelemetry semantic conventions (with `gen_ai.*` and emerging `ai.retrieval.*` extensions).
- LangSmith, Braintrust, Phoenix retrieval-trace conventions.
- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — the framework this builds on.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — the parallel pattern for LLM calls.
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](./agent-step-instrumentation.md) — the parallel pattern for agent loops.
- This repo: [rag-engineering/retrieval-observability.md](../rag-engineering/) (coming) — the engineering practice for retrieval observability at the pipeline level.
- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — the eval discipline that consumes retrieval observability.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — the wrapper this instrumentation lives in.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — the lineage system that retrieval spans feed.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
