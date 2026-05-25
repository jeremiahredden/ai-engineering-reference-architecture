# Retrieval Engineering

> **Audience.** Engineers building or refactoring the retrieval execution stage. Tech leads asking "we have indexing; now how do we make queries fast and accurate." **Scope.** The *engineering* practice of retrieval execution — hybrid retrieval implementation, merging strategies, multi-stage retrieval, metadata filtering, latency/quality tradeoffs. Pair with [chunking-engineering.md](./chunking-engineering.md), [embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md), [reranking-engineering.md](./reranking-engineering.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Retrieval is where the corpus meets the question. Good retrieval makes the rest of the system possible; bad retrieval cascades into wrong answers regardless of how good the model is. The engineering for retrieval — hybrid query execution, score merging, metadata filtering, latency tuning — is what determines whether a RAG system meets its retrieval recall and latency SLOs.

Most teams start with a single-retriever pattern (vector-only) and discover its limitations after deployment. The team adds hybrid (BM25+vector), then reranking, then query rewriting, each in response to observed quality issues. The trajectory is predictable; the discipline this document codifies is doing it deliberately rather than reactively.

The architecture sibling's `rag-architecture-decision-guide.md` chose the RAG pattern; this document is *how to execute* the chosen pattern at production quality and scale.

This document is opinionated about three things:

1. **Hybrid (BM25 + vector) is the production default.** Single-retriever patterns are starting points; hybrid is what production workloads consistently benefit from.
2. **Score merging is itself engineering.** RRF, weighted, score-normalized — each has trade-offs; the choice is eval-validated, not arbitrary.
3. **Metadata filtering is first-class.** Production retrieval almost always needs metadata constraints (tenant scope, content-type, time window); the implementation handles them efficiently.

Structure: (2) the retrieval pipeline; (3) hybrid retrieval implementation; (4) score merging strategies; (5) metadata filtering; (6) multi-stage retrieval; (7) latency optimization; (8) integration with the wrapper; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The retrieval pipeline

The execution-time pipeline that runs per user query.

### 2.1 The stages

```
User query
    │
    ▼
[Query embedding]      — embed the query (per embedding-pipeline-engineering.md hot-path)
    │
    ▼
[Query rewriting]      — if applicable; conversational context; multi-step decomposition
    │
    ▼
[Parallel retrieval]   — BM25 + vector + (optionally graph) in parallel
    │
    ▼
[Merge]                — combine scores from parallel retrievers
    │
    ▼
[Filter]               — apply metadata filters
    │
    ▼
[Rerank]               — if applicable (per reranking-engineering.md)
    │
    ▼
[Return]               — top-K to the caller
```

Each stage is independently testable. The whole pipeline runs per query within the latency budget.

### 2.2 The latency budget

The pipeline's latency budget per query:

- Query embedding: 50-200ms (single hot-path embed call).
- Query rewriting: 200-1000ms (LLM call; only when applied).
- Parallel retrieval: 100-300ms (BM25 + vector run concurrently).
- Merge: ~10ms (in-memory operation).
- Filter: ~5ms (in the underlying store).
- Rerank: 200-500ms (model call).

Total: ~600-2000ms per query, depending on which stages run.

For real-time chat: target < 1s total per [sync-vs-async-vs-streaming.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/sync-vs-async-vs-streaming.md) section 8.2. The retrieval pipeline must fit within the broader latency budget.

### 2.3 The throughput budget

Per-instance query throughput depends on:
- Underlying store query latency.
- Concurrency configuration.
- API rate limits (embedding API, rerank API).

For Meridian Care Coordinator at production scale: ~30 QPS sustained per pipeline instance; multi-instance for load balancing.

### 2.4 The pipeline as a wrapper component

The retrieval pipeline is invoked through the retrieval-client wrapper per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md). The wrapper handles scope; the pipeline handles execution.

Wrapper + pipeline together = the production retrieval system.

---

## 3. Hybrid retrieval implementation

Hybrid (BM25 + vector) is the production default.

### 3.1 Why hybrid wins

- **BM25** excels on lexical matches, named entities, exact keywords, codes (drug names, ICD-10 codes).
- **Vector** excels on semantic similarity, paraphrasing, conceptual matches.
- **Hybrid** captures both; production workloads consistently see 5-15 point recall lifts over either alone.

The architecture decision for hybrid is in the architecture sibling's `rag-architecture-decision-guide.md`; this document is the engineering of the chosen hybrid pattern.

### 3.2 The parallel execution

The two retrievers run concurrently:

```python
async def retrieve_hybrid(query, embedding, top_k, filters):
    bm25_task = retrieve_bm25(query, top_k * 5, filters)
    vector_task = retrieve_vector(embedding, top_k * 5, filters)
    bm25_results, vector_results = await asyncio.gather(bm25_task, vector_task)
    return merge(bm25_results, vector_results, top_k)
```

Each retriever is asynchronous; total latency is the max of the two, not the sum.

### 3.3 The candidate count

Each retriever returns more candidates than the final top-K:

- top_k = 10 (final return).
- Each retriever returns top-50 candidates.
- Merge produces top-K from the combined 100 candidates (with deduplication).

The over-fetch supports:
- Reranking (rerank picks the best from a larger candidate set).
- Merging (the union of two retrievers' top-N has redundancy; over-fetch ensures the final top-K is well-supported).

### 3.4 The BM25 implementation

BM25 implementations:

- **PostgreSQL FTS.** Built-in; uses `ts_vector` and `ts_query`. Adequate for most workloads.
- **OpenSearch / Elasticsearch.** More advanced; configurable scoring; large-scale.
- **In-memory indexes (Lucene-based libraries).** For specific use cases; not typical for production-scale.

For Meridian: PostgreSQL FTS in the same Aurora instance as pgvector. One database, two retrieval indexes, one query path.

### 3.5 The vector implementation

Vector implementations:

- **pgvector** with HNSW or IVF index. Same DB as BM25 if using PostgreSQL; co-located.
- **Pinecone / Weaviate / Qdrant.** Purpose-built; separate from the document store; more advanced.
- **OpenSearch with vector plugin.** Hybrid-in-one-store.

For Meridian: pgvector with HNSW. Same connection pool, same transaction context as the BM25 query (when needed for consistency).

### 3.6 The hybrid query API

The engineering wraps both retrievers in a unified API:

```python
results = retrieval_client.hybrid_retrieve(
    query="What is the post-discharge follow-up protocol for CHF?",
    top_k=10,
    filters={
        "tenant_id": ["mercy-cleveland", "global-guidelines"],
        "corpus": "clinical-guidelines-and-protocols",
    },
    candidate_count_per_retriever=50,
    merge_strategy="rrf",
)
```

The unified API hides the retriever-specific details; downstream code (the LLM call site) doesn't know about BM25 vs vector.

---

## 4. Score merging strategies

Two retrievers produce two ranked lists; the merge produces one.

### 4.1 The strategies

**Reciprocal Rank Fusion (RRF).** Default for most workloads.

```
For each candidate doc:
    score = sum over retrievers of (1 / (k + rank_in_retriever))
where k is typically 60.

Final ranking: sort by score (descending).
```

RRF requires only ranks (not scores); robust to different scoring scales.

**Weighted score merge.**

```
For each candidate doc:
    score = w_bm25 * normalized_bm25_score + w_vector * normalized_vector_score
where w_bm25 + w_vector = 1.

Final ranking: sort by score.
```

Weighted requires score normalization; produces a calibrated combined score.

**Score-normalized merge.**

Both retrievers' scores normalized to [0, 1]; arithmetic combination (max, average, weighted sum).

### 4.2 The strategy choice

| Workload property | Recommended strategy |
|---|---|
| New corpus; no prior data | RRF (robust default) |
| Eval shows clear retriever-specific dominance per case class | Weighted (tune weights to favor the dominant retriever per class) |
| Scoring scales are stable across queries | Score-normalized weighted |
| Workload requires explainable scores | Score-normalized (more interpretable) |

The Meridian default is RRF; it works without per-corpus calibration.

### 4.3 The eval validation

The merge strategy choice is eval-validated:

- For each candidate strategy: run the eval; measure recall and precision.
- Pick the strategy that maximizes recall at acceptable precision.
- Document the choice.

The choice may differ per content type; per-content-type merge strategies are supported.

### 4.4 The deduplication

The merge encounters duplicates (a doc returned by both retrievers). Deduplication:

- Match by doc_id (most common).
- Retain the highest-scored instance.
- The deduplication is part of the merge logic.

### 4.5 The merge implementation

```python
def reciprocal_rank_fusion(retriever_results: list[list[Result]], k: int = 60) -> list[Result]:
    """RRF merge of multiple ranked result lists."""
    scores = defaultdict(float)
    docs = {}
    for results in retriever_results:
        for rank, result in enumerate(results, start=1):
            scores[result.doc_id] += 1 / (k + rank)
            docs[result.doc_id] = result  # latest occurrence wins for the metadata
    sorted_doc_ids = sorted(scores.keys(), key=lambda d: scores[d], reverse=True)
    return [docs[doc_id] for doc_id in sorted_doc_ids]
```

The implementation is simple; the discipline is choosing it deliberately and validating against eval.

---

## 5. Metadata filtering

Production retrieval almost always needs metadata constraints.

### 5.1 The common filters

- **Tenant scope.** Per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — every query filters by tenant.
- **Corpus / content-type.** Which corpus or content type to retrieve from.
- **Time window.** Only documents updated within the last N days.
- **Sensitivity level.** Caller's sensitivity-clearance.
- **Specific fields.** Workload-specific (e.g., "only documents tagged 'cardiology'").

### 5.2 The filter implementation

For each retriever:

- **BM25 (PostgreSQL FTS):** SQL `WHERE` clauses combined with the FTS query.
- **Vector (pgvector):** SQL `WHERE` clauses combined with the vector query.
- **Pinecone:** metadata filter passed to the query API.
- **OpenSearch:** filter clause in the search query.

Each retriever's filter syntax is provider-specific; the wrapper translates a uniform filter object to per-retriever syntax.

### 5.3 The pre-filter vs post-filter

- **Pre-filter (efficient).** Filter applied during retrieval; the underlying index handles the constraint.
- **Post-filter (inefficient).** Filter applied after retrieval; results filtered after the index already returned them.

Pre-filtering is more efficient; the index doesn't return excluded results in the first place. Post-filtering is sometimes necessary (when the filter is too complex for the index) but loses efficiency.

The implementation prefers pre-filter; post-filter only when forced.

### 5.4 The filter selectivity

A highly selective filter (returns 1% of the corpus) needs different handling than a low-selectivity filter (returns 90%):

- Highly selective: filter early; the remaining set is small; retrieval is fast.
- Low selectivity: filter does little; the retriever still processes most of the corpus.

Some retrievers handle selective filters poorly (the index can't use them efficiently). Workload-specific tuning is sometimes needed.

### 5.5 The mandatory filters

Some filters are mandatory and enforced architecturally per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md):

- Tenant filter: always applied; cannot be skipped.
- Sensitivity filter: applied based on caller's clearance.

The wrapper injects these; calling code cannot omit.

### 5.6 The optional filters

Other filters are caller-controlled:

- Time window: caller may specify "last 90 days" or omit (no time constraint).
- Specific tags: caller may include or omit.

Optional filters are part of the retrieval API; the wrapper passes them through to the underlying retrievers.

---

## 6. Multi-stage retrieval

Some workloads benefit from multi-stage retrieval pipelines.

### 6.1 The stages

```
User query
    │
    ▼
[Stage 1: Recall]      — retrieve a large candidate set (top-100)
    │
    ▼
[Stage 2: Filter]      — apply additional filtering (often complex)
    │
    ▼
[Stage 3: Rerank]      — model-based reranking (top-10 from filtered set)
    │
    ▼
[Stage 4: Context fit] — select chunks to fit within the LLM's context budget
    │
    ▼
[Return]
```

### 6.2 When multi-stage helps

- Workloads where retrieval recall is the bottleneck (need many candidates) but precision must be high in the final set (reranker tightens).
- Workloads with complex filtering that doesn't fit the underlying index (multi-criteria, joins).
- Workloads with context-window budgets that require selecting the most-valuable chunks (context-fit stage).

### 6.3 The recall-then-precision pattern

The most common multi-stage:

- Stage 1: high-recall, low-precision (top-100 from hybrid retrieval).
- Stage 2: rerank or filter to high-precision (top-10).

Per [rag-architecture-decision-guide.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/rag-architecture-decision-guide.md) section 3.3, this is the hybrid-plus-reranker pattern.

### 6.4 The context-fit stage

When the model's context budget is tight, the context-fit stage selects which chunks to include:

- Calculate token cost per chunk.
- Sum until the budget is reached; drop the lowest-scored chunks.
- Or: include diverse chunks (one per section) rather than the top-K by score.

The context-fit decision affects both quality (more chunks may help) and cost (more tokens = more cost). Workload-specific tuning.

### 6.5 The integration with reranking

Multi-stage retrieval naturally accommodates reranking per [reranking-engineering.md](./reranking-engineering.md) as one stage.

---

## 7. Latency optimization

Retrieval latency is often the dominant component of total RAG latency.

### 7.1 The latency components

- **Query embedding:** 50-200ms per call.
- **BM25 query:** 10-100ms depending on index size and query complexity.
- **Vector query:** 10-100ms depending on index size, dimensions, HNSW parameters.
- **Merge:** ~10ms.
- **Rerank:** 200-500ms.

The total depends on which components run; parallel execution helps.

### 7.2 The optimization patterns

- **Parallel BM25 + vector.** Total = max(bm25, vector), not sum.
- **Reduce candidate counts.** Top-30 instead of top-100; less to process; less to rerank.
- **Tune HNSW parameters.** Larger M, larger efConstruction = better recall, slower queries. Tune for the latency budget.
- **Cache hot queries.** If the same query is asked frequently, cache the retrieval result.
- **Embedding cache.** Cache the query embedding when the same query repeats.

### 7.3 The candidate-count tuning

Each retriever returns N candidates; the merge produces top-K from up to 2N (if no overlap) or fewer (if overlap).

Too few candidates (N=20): top-K may miss good results in the long tail.
Too many candidates (N=200): retrieval is slower; merge is slower; reranking is slower.

Sweet spot: N=50-100 for top-K=10. Eval-validated per workload.

### 7.4 The HNSW tuning

For pgvector HNSW:

- **M** (max edges per node): higher = better recall, more memory, slower build.
- **efConstruction** (build-time): higher = better recall at query time, slower build.
- **efSearch** (query-time): higher = better recall, slower query.

Production tuning: M=16, efConstruction=64, efSearch=40 for Meridian's corpus size. Tuned per workload via shadow testing.

### 7.5 The caching pattern

Hot-query caching:

- LRU cache of (query_hash, filter_hash) → retrieval results.
- Cache invalidation: TTL-based (5-15 minutes) or on-corpus-update.
- Hit rate is workload-dependent; chat workloads often see 20-40% cache hits on common questions.

Embedding caching:

- Per-query-text cache of embeddings.
- LRU with longer TTL (hours).
- Saves embedding API calls on repeat queries.

### 7.6 The streaming consideration

For streaming response patterns, retrieval should complete before generation starts. The retrieval latency is part of the user-visible time-to-first-token (TTFT) for chat-shaped workloads.

If retrieval takes 800ms, TTFT cannot be under 800ms regardless of model speed. Optimize retrieval to meet the TTFT target.

---

## 8. Integration with the retrieval-client wrapper

The retrieval pipeline lives inside the retrieval-client wrapper.

### 8.1 The wrapper as chokepoint

Per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md), the wrapper is the only path to retrieval. The retrieval pipeline is internal to the wrapper; calling code never invokes BM25 or vector queries directly.

### 8.2 The wrapper's responsibilities

- Authorize the query (per scope rules).
- Inject mandatory filters.
- Invoke the retrieval pipeline.
- Verify scope on results.
- Apply lineage tokens.
- Emit observability per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md).
- Return results.

The pipeline does the retrieval; the wrapper handles cross-cutting concerns.

### 8.3 The error path

If the pipeline fails (one retriever down, embedding API timeout, etc.):

- Configured fallback per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md): retrieval-fallback to BM25-only when vector fails; or vice versa.
- Worst case: graceful failure; the wrapper returns empty results with a marker for the caller.

The caller's prompt is built to handle empty-retrieval cases.

### 8.4 The observability integration

The pipeline emits spans per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md):

- Parent `retrieval` span.
- Per-retriever sub-span (bm25, vector).
- Rerank sub-span (if applicable).
- Filter span attributes (which filters were applied, scope verification result).

The trace supports diagnostic walkthroughs.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The retrieval pipeline

Meridian's pipeline:

```
Query (from clinical-knowledge worker)
    │
    ▼
[Query embedding]      — pgvector hot-path embed (~100ms)
    │
    ▼
[Query rewriting]      — only if turn 2+ in conversation (~300ms)
    │
    ▼
[Parallel retrieval]   — BM25 (PostgreSQL FTS) + vector (pgvector); top-50 each
    │
    ▼
[RRF merge]            — top-50 from combined
    │
    ▼
[Filter]               — tenant scope + content-type
    │
    ▼
[Cohere Rerank-3.5]    — top-10 from top-50
    │
    ▼
[Return]               — top-10 to the clinical-knowledge worker
```

Total typical latency: ~450ms.

### 9.2 The hybrid implementation

BM25 (PostgreSQL FTS):
- `ts_vector` column on chunks.
- GIN index for performance.
- Query: `ts_query` from user query; `WHERE tenant_id IN (...) AND content_type = '...'`.

Vector (pgvector):
- 1536-dim embeddings.
- HNSW index per partition.
- Query: `embedding <=> query_embedding` ordering with `LIMIT 50`.

Both queries run concurrently against the same Aurora cluster; total parallel latency ~80-120ms.

### 9.3 The merge

RRF with k=60 (standard default). No per-content-type weight tuning; uniform RRF across the corpus.

The merge is in-Python (after the SQL queries return). Sub-millisecond.

### 9.4 The filter

Mandatory filters:
- Tenant scope: `tenant_id IN (<caller_tenant>, 'global-guidelines')`.
- Content-type: per the calling feature (`clinical-guidelines-and-protocols`).

Optional filters:
- Time window: not used by default; available for cases that need recency.
- Sensitivity: tied to caller role.

All filters applied at the SQL level (pre-filter); efficient.

### 9.5 The reranker

Cohere Rerank-3.5:
- Input: top-50 from RRF merge.
- Output: top-10 reranked.
- Latency: ~200-300ms.
- Cost: ~$0.0025 per call.

Per [reranking-engineering.md](./reranking-engineering.md), the reranker's lift is measured and earns its cost.

### 9.6 The latency profile

Typical latency breakdown:
- Query embedding: ~90ms.
- BM25 + vector (parallel): ~110ms.
- Merge: ~5ms.
- Filter: (in-SQL; counted in retrieval).
- Rerank: ~260ms.
- Total: ~465ms.

Within the Care Coordinator's overall latency budget; retrieval is not the bottleneck.

### 9.7 The throughput

Production peak: ~25 QPS sustained. Single Aurora reader instance handles the load comfortably; vector query latency stable under load.

Multi-instance for horizontal scaling if needed (not currently needed at current volume).

### 9.8 The 2026-Q1 latency incident

In 2026-Q1, vector query latency spiked to ~400ms p95. Investigation:

- HNSW efSearch was bumped to 100 during a quality experiment.
- The experiment improved recall by 1 point but degraded latency by 200ms.
- Decision: revert efSearch to 40; the recall gain wasn't worth the latency cost.

The incident reinforced: HNSW tuning is per-workload trade-off; eval the quality-latency joint, not just quality.

### 9.9 The platform discipline

- Hybrid (BM25 + vector + RRF + rerank) is the standard pipeline.
- Parallel execution; concurrent queries.
- Filter pre-applied at the index level.
- Per-query trace observability.
- Quarterly tuning review (HNSW parameters, candidate counts).

---

## 10. Anti-patterns

### 10.1 "Vector-only retrieval"

Production uses vector only; BM25 not used. Lexical-precise queries (drug names, codes) underperform.

**Corrective.** Hybrid (BM25 + vector) per section 3.

### 10.2 "Sequential BM25 + vector"

The two retrievers run sequentially; total latency is sum, not max. Latency unnecessarily high.

**Corrective.** Parallel execution per section 3.2.

### 10.3 "Score merging by intuition"

Team picked a merge strategy without evaluation; doesn't validate against alternatives.

**Corrective.** Eval validation per section 4.3; RRF as default.

### 10.4 "Post-filter when pre-filter is possible"

Retrieval returns all results; filtering happens in calling code; the index returned data that should have been excluded. Latency and cost suffer.

**Corrective.** Pre-filter per section 5.3.

### 10.5 "Mandatory filters skippable"

Calling code can bypass tenant filtering; isolation breaks.

**Corrective.** Wrapper enforces per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md).

### 10.6 "HNSW parameters not tuned"

Default HNSW parameters; latency or recall is suboptimal.

**Corrective.** Tune per workload per section 7.4; shadow-test changes.

### 10.7 "Caching absent for hot queries"

Common questions hit the database every time; throughput is bounded by database query rate.

**Corrective.** LRU cache for hot queries per section 7.5.

### 10.8 "Candidate count too high"

Top-100 candidates per retriever when top-30 would suffice; merge and rerank latency suffers.

**Corrective.** Tune candidate count per section 7.3; eval validates lower counts suffice.

---

## 11. Findings (sprint-assignable)

### RETR-001 — Severity: High
**Finding.** Vector-only retrieval; lexical queries underperform.
**Recommendation.** Hybrid per section 3.
**Owner.** ai-platform-eng, sprint N+1.

### RETR-002 — Severity: High
**Finding.** BM25 and vector run sequentially; latency unnecessarily high.
**Recommendation.** Parallel per section 3.2.
**Owner.** ai-platform-eng, sprint N+2.

### RETR-003 — Severity: High
**Finding.** Merge strategy not eval-validated; suboptimal choice.
**Recommendation.** Eval per section 4.3.
**Owner.** ai-platform-eng, sprint N+2.

### RETR-004 — Severity: High
**Finding.** Post-filter pattern; index returns excluded results; latency suffers.
**Recommendation.** Pre-filter per section 5.3.
**Owner.** ai-platform-eng, sprint N+2.

### RETR-005 — Severity: High
**Finding.** Tenant filter not mandatory in retrieval; isolation depends on calling code.
**Recommendation.** Wrapper enforcement per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
**Owner.** ai-platform-eng + security-eng, sprint N+1.

### RETR-006 — Severity: High
**Finding.** HNSW parameters at defaults; not tuned for workload.
**Recommendation.** Tune per section 7.4; eval the quality-latency joint.
**Owner.** ai-platform-eng, sprint N+3.

### RETR-007 — Severity: High
**Finding.** Hot-query caching absent; redundant database load.
**Recommendation.** LRU cache per section 7.5.
**Owner.** ai-platform-eng, sprint N+3.

### RETR-008 — Severity: High
**Finding.** Candidate count too high; merge and rerank latency suffers.
**Recommendation.** Tune per section 7.3.
**Owner.** ai-platform-eng, sprint N+2.

### RETR-009 — Severity: Medium
**Finding.** Retrieval latency not measured; SLO compliance unknown.
**Recommendation.** Per-stage latency metrics per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md).
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### RETR-010 — Severity: Medium
**Finding.** Retrieval throughput not capacity-planned; spikes overwhelm.
**Recommendation.** Throughput planning per section 2.3.
**Owner.** ai-platform-eng + sre, sprint N+3.

### RETR-011 — Severity: Medium
**Finding.** Embedding cache absent; repeat query embeds.
**Recommendation.** Embedding cache per section 7.5.
**Owner.** ai-platform-eng, sprint N+3.

### RETR-012 — Severity: Medium
**Finding.** Filter selectivity not analyzed; some filters force inefficient query plans.
**Recommendation.** Per-filter selectivity analysis per section 5.4.
**Owner.** ai-platform-eng, sprint N+4.

### RETR-013 — Severity: Medium
**Finding.** Multi-stage retrieval not used despite workload benefit.
**Recommendation.** Evaluate multi-stage per section 6.
**Owner.** ai-platform-eng, sprint N+4.

### RETR-014 — Severity: Medium
**Finding.** Context-fit stage absent; chunks selected by score alone, exceeding context budget.
**Recommendation.** Context-fit per section 6.4.
**Owner.** ai-platform-eng, sprint N+4.

### RETR-015 — Severity: Medium
**Finding.** Per-content-type merge strategy not differentiated.
**Recommendation.** Per-content-type weight tuning if eval shows benefit per section 4.3.
**Owner.** ai-platform-eng, sprint N+4.

### RETR-016 — Severity: Low
**Finding.** Streaming TTFT impacted by retrieval latency; not jointly optimized.
**Recommendation.** Joint optimization of retrieval and streaming per section 7.6.
**Owner.** ai-platform-eng, sprint N+5.

### RETR-017 — Severity: Low
**Finding.** Retrieval-config A/B tests not framework-supported; manual experiments only.
**Recommendation.** Shadow-config A/B framework.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### RETR-018 — Severity: Low
**Finding.** Retrieval-execution documentation thin; new engineers cannot reason about the pipeline.
**Recommendation.** Documentation alongside the implementation.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building production retrieval:

- [ ] **Sprint 0 — design.** Choose retrieval pattern (per architecture sibling's RAG decision guide); decide merge strategy.
- [ ] **Sprint 1 — hybrid implementation.** BM25 + vector in parallel; RRF default merge.
- [ ] **Sprint 1 — filter integration.** Mandatory filters (tenant); optional filters (content-type, time).
- [ ] **Sprint 2 — wrapper integration.** Pipeline lives inside the retrieval-client wrapper.
- [ ] **Sprint 2 — observability.** Per-stage spans; latency metrics.
- [ ] **Sprint 3 — reranking.** Reranker integration per [reranking-engineering.md](./reranking-engineering.md).
- [ ] **Sprint 3 — HNSW tuning.** Shadow-test parameter changes; eval validates quality-latency joint.
- [ ] **Sprint 4 — caching.** Hot-query cache; embedding cache.
- [ ] **Sprint 4 — multi-stage.** If workload benefits, multi-stage pipeline.
- [ ] **Sprint 5 — context-fit.** Context budget aware chunk selection.
- [ ] **Ongoing — discipline.** Quarterly tuning review; per-query latency monitored; throughput planned.

A team that completes this sequence has retrieval that meets quality and latency SLOs. A team that ships a vector-only or sequential pipeline leaves both on the table.

---

## 13. References

- This repo: [rag-engineering/chunking-engineering.md](./chunking-engineering.md) — what the retrieval consumes.
- This repo: [rag-engineering/embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md) — the embeddings the retrieval queries.
- This repo: [rag-engineering/reranking-engineering.md](./reranking-engineering.md) — the rerank stage.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — span emission.
- This repo: [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — fallback when retrievers fail.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/rag-architecture-decision-guide.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/rag-architecture-decision-guide.md) — the pattern decision this implements.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md) — the wrapper architecture.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/vector-store-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/vector-store-architecture.md) — the storage layer.
- pgvector, Pinecone, Weaviate, OpenSearch hybrid-search documentation.
- Cormack et al., *Reciprocal Rank Fusion outperforms Condorcet and Individual Rank Learning Methods* (2009).
