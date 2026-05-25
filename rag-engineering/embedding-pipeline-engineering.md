# Embedding Pipeline Engineering

> **Audience.** Engineers building or refactoring the embedding stage. Tech leads who have been surprised by "the embedding model changed underneath us" or "we need to re-embed everything by Friday." **Scope.** The *engineering* practice of embedding generation at scale — batching, throughput, model-version pinning, deprecation tracking, rebuild-vs-incremental, drift detection, embedding-model migration. Pair with [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md), [chunking-engineering.md](./chunking-engineering.md), [retrieval-engineering.md](./retrieval-engineering.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Embeddings are dependencies. Every embedding in the corpus is produced by a specific embedding model version with a specific configuration; the embeddings are useful only as long as the production retrieval uses the same model+version+configuration. When the embedding model changes (new version, deprecation, alternative model considered), the entire corpus needs to be re-embedded. For a 50K-chunk corpus, this is a few hours; for a 50M-chunk corpus, weeks.

Most teams underestimate the embedding-model-as-dependency story. They use the embedding model's default identifier (often an alias resolving to "latest"); they don't pin the version; they discover the corpus has drifted when retrieval quality regresses. The remediation — re-embed against a pinned version, establish discipline going forward — is itself a multi-week project for non-trivial corpora.

The discipline this document codifies: embedding is a *pipeline engineering problem* with model-as-dependency, version-as-contract, throughput-as-SLO. The pipeline is tested at scale; failures are recoverable; migrations are structured projects.

This document is opinionated about three things:

1. **The embedding model version is pinned.** No aliases. Pinned per [model-registry.md](../model-lifecycle/model-registry.md) and [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md). Treated as a release-time dependency.
2. **Embedding migration is a structured project.** Embedding-model changes trigger a full re-embed; the project has a shadow phase, an eval-validation phase, a cutover, a decommission. Not an inline swap.
3. **Per-call cost telemetry applies to embeddings.** Embedding API calls produce trace spans per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md); cost is attributed; circuit breakers apply.

Structure: (2) the embedding pipeline stages; (3) batching and throughput; (4) embedding model version pinning; (5) rebuild vs incremental; (6) drift detection; (7) embedding-model migration; (8) failure handling; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The embedding pipeline stages

Embedding is a sub-pipeline within the broader ingestion pipeline.

### 2.1 The stages

```
Chunks (from chunking-engineering)
    │
    ▼
[Batch formation]      — group chunks into batches for API efficiency
    │
    ▼
[Embed]                — invoke the embedding model API per batch
    │
    ▼
[Validate]             — confirm embeddings produced (dimension check, NaN check)
    │
    ▼
[Index write]          — write embeddings to vector store
    │
    ▼
[Lineage update]       — record per-chunk embedding metadata
```

Each stage is independently failable; the pipeline orchestrator handles per-stage failures.

### 2.2 The batch formation

Embedding APIs accept multiple inputs per call (typically 8-256 inputs per call, provider-specific). Batching:

- Reduces per-call overhead.
- Improves throughput.
- Stays within provider rate limits.

Batch size is provider-specific and rate-limit-aware. For OpenAI text-embedding: ~100 inputs per call typically. For Cohere: ~96. The pipeline tunes batch size per provider.

### 2.3 The embed call

Per batch:

```python
response = embedding_client.embed(
    provider="openai",
    model="text-embedding-3-large",
    model_version="2024-01-25",   # pinned
    inputs=batch_of_chunk_texts,
    dimensions=1536,
    context=call_context,          # tenant, feature, trace
)

for chunk, embedding in zip(batch, response.embeddings):
    write_embedding(chunk, embedding, model_version=response.model_version)
```

The call goes through the LLM-call wrapper per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md); trace span; cost attribution.

### 2.4 The validation

Per embedding:

- **Dimension check.** Embedding has the expected dimension (e.g., 1536). Wrong dimension = pipeline configuration mismatch.
- **NaN / Inf check.** Embedding values are finite. NaN indicates provider error or pipeline bug.
- **Magnitude check (optional).** Embedding magnitude is in expected range. Outliers may indicate empty or garbage input.

Failed embeddings are flagged; the corresponding chunk is quarantined or re-attempted.

### 2.5 The index write

The embedding + chunk metadata is written to the vector store. Per [vector-store-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/vector-store-architecture.md):

- pgvector: INSERT into the per-tenant partition.
- Pinecone / Weaviate: upsert with metadata.
- The write is atomic per chunk; per-batch transactions where supported.

### 2.6 The pipeline observability

Per-batch trace:

- Batch size.
- Per-batch latency.
- Per-batch cost.
- Per-batch success / failure.
- Embedding model version.

Per-pipeline-run aggregate:

- Total chunks embedded.
- Total cost.
- Total runtime.
- Failure rate.

The observability supports: throughput planning, cost monitoring, failure diagnosis.

---

## 3. Batching and throughput

The pipeline's throughput determines how long an embedding job takes.

### 3.1 The throughput equation

```
Throughput = (batch_size × batches_per_minute)
Batches_per_minute = bounded by rate limits and concurrency
```

Provider rate limits (requests per minute, tokens per minute) determine the ceiling.

### 3.2 The rate-limit-aware batching

The pipeline knows provider rate limits and stays within them:

- Per-minute token quota: track tokens consumed; back off as the quota approaches.
- Per-minute request quota: track requests; throttle.

Hitting rate limits produces errors; the pipeline backs off; throughput stalls. The discipline: monitor rate-limit headroom; provision quotas before they're exhausted.

### 3.3 The concurrency

The pipeline can run multiple batches concurrently:

- Increases throughput.
- Subject to provider rate limits (more concurrent calls = faster rate-limit consumption).
- Subject to local resource limits (memory, network).

Typical concurrency: 5-20 concurrent batches per pipeline instance. Multiple pipeline instances may run if quota allows.

### 3.4 The throughput targets

For different scenarios:

- **Initial corpus build.** 50K chunks; can take hours; throughput planned for the available quota.
- **Incremental update.** Daily delta of ~500 chunks; takes minutes.
- **Full re-embed migration.** All chunks re-embedded; planned as a multi-day project.
- **Hot path** (real-time embedding of user queries for retrieval). Latency-critical; 1-2 queries per second per instance; per-query latency 100-500ms.

The hot path is different from batch path: hot path uses single-input or small-batch calls; batch path uses large batches.

### 3.5 The throughput optimization

- **Batch size tuning.** Test larger batches; verify quality and latency hold.
- **Concurrency tuning.** Test more concurrent batches; verify rate limits hold.
- **Provider region selection.** Some providers offer faster endpoints in some regions.
- **Multiple provider accounts.** Distribute load across accounts to expand effective quota.

Each optimization tested in shadow; verified against quality metrics.

---

## 4. Embedding model version pinning

The most-important discipline.

### 4.1 The pin

The embedding model is pinned to a specific provider+model+version:

```yaml
embedding_model:
  provider: openai
  model: text-embedding-3-large
  version_pin: "2024-01-25"
  dimensions: 1536
```

Pinned via the model registry per [model-registry.md](../model-lifecycle/model-registry.md); referenced in the pipeline configuration; recorded on every embedding produced.

### 4.2 Why pinning matters

Without pinning:

- Provider releases a new version behind the alias; subsequent embeddings differ from prior embeddings.
- The corpus has a mix of embeddings from two model versions.
- Retrieval against a mixed corpus produces inconsistent results (vector spaces don't align across versions).
- The team discovers the issue weeks or months later; remediation requires a full re-embed.

The pin prevents this. Provider version changes are caught at the gateway (per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) section 4); the embedding pipeline doesn't dispatch calls to non-pinned versions.

### 4.3 The per-chunk recording

Each embedding records its source model version:

```yaml
chunk: chunk-id
embedding: [...]
embedding_model_version: "openai/text-embedding-3-large@2024-01-25"
created_at: 2026-05-25T08:42:00Z
```

Per-chunk attribution lets the team:

- Identify chunks from old model versions (potential re-embed targets).
- Diagnose retrieval drift across versions.
- Audit corpus consistency.

### 4.4 The release manifest integration

The embedding model version is part of the release manifest per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md):

```yaml
models:
  embedding_corpus: openai/text-embedding-3-large@2024-01-25
```

Releases that change the embedding model version are deliberate; they require eval validation and migration planning.

### 4.5 The deprecation tracking

Embedding models deprecate. The model registry tracks deprecation per [model-registry.md](../model-lifecycle/model-registry.md):

- Subscribe to provider deprecation announcements.
- When a deprecation is announced: identify the corpus that uses the model; plan migration to a replacement.
- The migration is a structured project (per section 7).

Without deprecation tracking, the team learns about deprecation when the provider stops serving the model; production breaks.

---

## 5. Rebuild vs incremental

The two pipeline modes.

### 5.1 Incremental embedding

For incremental updates:

- New chunks from new documents are embedded.
- Updated chunks (from updated documents) are re-embedded.
- Existing unchanged chunks are not re-embedded.

The change-detection logic per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 5 drives this; embedding is the consumer of "what changed."

### 5.2 Full re-embed

For migration scenarios:

- All chunks in the corpus are re-embedded.
- New embeddings written to a shadow index.
- Eval validation against the shadow.
- Cutover: retrieval routes to the shadow index.
- Old embeddings decommissioned.

Triggers for full re-embed:

- **Embedding model version change.** Required (the new model produces different embeddings).
- **Dimension change.** Required (different dimension = incompatible index).
- **Pipeline-bug remediation.** When a chunking or pipeline bug produced corrupted embeddings, full re-embed fixes.

### 5.3 The full re-embed project shape

A full re-embed is a multi-step project:

1. **Provision quota.** Embedding-API quota for the duration; estimate hours/days.
2. **Stand up shadow index.** New vector store collection / namespace.
3. **Re-embed batch.** Process the entire corpus through the embedding pipeline against the shadow.
4. **Eval validation.** Run the eval suite against the shadow; compare to production.
5. **Cutover.** Atomic switch (retrieval routes to shadow); production index decommissioned.
6. **Cleanup.** Old embeddings deleted after a grace period.

For a 50K-chunk corpus: ~3-6 hours of compute time. For a 5M-chunk corpus: days.

### 5.4 The reconciliation re-embed

Periodic reconciliation runs a partial re-embed for drift detection:

- Sample 1% of the corpus; re-embed.
- Compare new embeddings to stored embeddings.
- Variance > epsilon indicates drift.

Drift detected → investigate (was the embedding model alias updated silently? Did the corpus content change without the pipeline detecting?).

### 5.5 The cost projection

Full re-embed cost = chunks × tokens_per_chunk × cost_per_million_tokens.

For Meridian's 50K chunks × ~400 tokens × $0.13/M tokens = ~$2.60. Negligible.

For a 50M-chunk corpus: $2,600. Meaningful but bounded.

For some workloads, the cost is meaningful enough that batch-API pricing (50% discount, longer latency) is worth using.

---

## 6. Drift detection

Embedding drift can be silent and damaging.

### 6.1 The drift sources

- **Model version change.** Provider released new version behind alias; corpus has mixed versions.
- **API parameter change.** Default dimension or normalization changed; embeddings differ.
- **Pipeline configuration change.** A change to the embedding pipeline (e.g., normalization on/off) shifts embeddings.
- **Content drift.** Source documents updated; chunks updated; embeddings legitimately differ. (Not really drift; the change is intentional.)

The first three are silent failures; the fourth is normal operation.

### 6.2 The detection patterns

- **Per-chunk model-version-stored.** Compare against current configured version; mismatch = drift.
- **Sampled re-embed.** Sample existing chunks; re-embed; compare to stored. Distance > epsilon = drift.
- **Per-corpus version distribution.** Visualize model-version distribution across the corpus; multi-version = drift.

Drift detection is part of routine pipeline operations.

### 6.3 The remediation

When drift is detected:

- Investigate cause.
- If pipeline change: revert or plan corpus-wide migration.
- If silent provider change: plan full re-embed; update pinning to prevent recurrence.
- If content drift: confirm legitimate; no remediation.

### 6.4 The audit

Quarterly audit:

- Sample chunks across the corpus; verify model-version-stored matches the configured pin.
- Sample a small percentage; re-embed; verify distance is near-zero.

Audit failures trigger remediation projects.

---

## 7. Embedding-model migration

The structured project for changing embedding models.

### 7.1 The migration triggers

- **Provider deprecation.** Forced migration; deadline-driven.
- **Better-model availability.** Team-initiated; quality / cost / latency improvement.
- **Multi-modal need.** Workload now requires capabilities the current model lacks (image embeddings, multi-lingual embeddings).
- **Cost optimization.** New model is cheaper at the same quality level.

### 7.2 The migration project phases

1. **Decision.** Confirm the migration target; document rationale; get sign-off.
2. **Pre-migration eval.** Eval-validate the new model on a sample; quality must match or exceed current.
3. **Shadow build.** Stand up the shadow index; full re-embed.
4. **Shadow validation.** Eval suite against the shadow; quality SLI computed.
5. **Cutover.** Atomic switch; retrieval routes to shadow; new model is production.
6. **Decommission.** Old embeddings and old index removed after grace period.
7. **Documentation.** Migration report; lessons learned.

Typical timeline: 2-6 weeks depending on corpus size and complexity.

### 7.3 The pre-migration eval

Critical: validate the new model on a sample of the workload before committing to the migration:

- Embed a representative subset (e.g., 10% of corpus).
- Run the eval suite against the partial corpus with the new model.
- Compare retrieval recall to current.

If the new model performs worse: stop the migration; investigate why (different optimal chunking? different prompt assembly?).

### 7.4 The shadow validation

Full corpus on the new model; full eval suite:

- Retrieval recall on the eval set.
- Citation accuracy.
- Faithfulness.
- Empty-retrieval rate.

The new model must match or exceed current on each dimension. Acceptable trade-offs (cost reduction with quality on par) are documented.

### 7.5 The cutover

The atomic switch:

- Retrieval configuration updated to point at the shadow index.
- Cache (if any) invalidated.
- Within the cache-refresh window (typically minutes), all retrieval uses the new model.

Rollback path: revert the configuration; retrieval reverts to the old index (still active during grace period).

### 7.6 The post-cutover monitoring

For the first 24-48 hours after cutover:

- Online judge SLI watched closely.
- Per-class retrieval recall metrics compared to pre-cutover.
- User feedback monitored.

Issues detected → rollback (cheap because the old index is still active).

### 7.7 The decommission

After ~1 week of stable post-cutover operation:

- Old embeddings deleted.
- Old index decommissioned.
- Storage freed.

The decommission is irreversible; verify thoroughly before proceeding.

---

## 8. Failure handling

The pipeline anticipates failures.

### 8.1 The failure classes

| Class | Examples | Disposition |
|---|---|---|
| Transient API error | Provider 5xx | Retry with backoff |
| Rate limit exceeded | Provider 429 | Back off; retry later |
| Auth failure | Provider 401 | Halt; alert; investigate credentials |
| Quota exceeded | Provider quota response | Halt; alert; expand quota |
| Bad input | Provider 400 (malformed chunk) | Quarantine the chunk; alert |
| Dimension mismatch | Validation failure | Pipeline configuration bug; halt; investigate |
| NaN / Inf embedding | Validation failure | Quarantine; investigate |
| Network failure | Connection error | Retry with backoff |

Each class has a defined disposition.

### 8.2 The retry strategy

Bounded retry: 3 attempts with exponential backoff (1s, 2s, 4s). After 3 failures, the batch is quarantined.

Per-batch retry; not per-chunk (the batch is the atomic unit at the API level).

### 8.3 The quarantine

Failed batches go to a quarantine queue:

- Batch contents preserved.
- Failure metadata attached.
- The pipeline continues with other batches.
- Quarantined batches can be re-processed after fixing the cause.

### 8.4 The halt pattern

For severe failures (mass-quarantine, infrastructure failures, auth-expired):

- The pipeline halts.
- On-call alerted.
- Manual intervention required.

Halt prevents the pipeline from churning through a broken state.

### 8.5 The cost-runaway prevention

Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md), per-feature daily budget applies to embedding. A runaway pipeline (e.g., a bug that re-processes the entire corpus repeatedly) trips the circuit before cost explodes.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The embedding pipeline

Meridian's embedding pipeline:

- Embedding model: OpenAI text-embedding-3-large, version 2024-01-25, 1536 dimensions.
- Pinned via the model registry.
- Pipeline implemented as part of the ingestion Step Functions workflow.
- ~50K chunks across all corpora.
- Daily incremental: ~500 chunks new/updated; ~5 minutes processing.
- Quarterly full re-embed: ~3 hours processing; ~$10 cost.

### 9.2 The batching configuration

- Batch size: 96 chunks per API call (OpenAI tuned).
- Concurrency: 4 batches per pipeline instance.
- Single pipeline instance suffices for the workload.

Throughput: ~400 chunks per minute; sufficient for both incremental and full re-embed.

### 9.3 The pinning discipline

The OpenAI embedding model is pinned to "2024-01-25":

- The pin is in the release manifest per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
- The model registry records the pin as `active`.
- The pipeline rejects calls to non-pinned versions.

In 2026-Q1, OpenAI released a new embedding model version; the team's pin held; the new version was evaluated independently; the team chose not to migrate (the new model offered marginal improvement at the same cost).

### 9.4 The migration history

- **2025-Q1**: Initial corpus build with text-embedding-3-large@2024-01-25.
- **2025-Q3**: Considered migration to text-embedding-3-small (lower cost). Pre-migration eval showed 4% recall reduction. Rejected.
- **2026-Q1**: Provider released a new version. Evaluated. Marginal improvement; staying on 2024-01-25.
- No actual migration to date. The pinning has kept the corpus consistent.

### 9.5 The drift audit

Quarterly audit:
- Sample 200 chunks; re-embed; compare to stored.
- Maximum cosine distance: 0.0001 (consistent with floating-point variance).
- Per-corpus version distribution: 100% on 2024-01-25.

No drift detected; the discipline is working.

### 9.6 The failure handling

Production failure modes observed in 2026-Q2:

- Provider transient 5xx: ~3 batches per week; auto-retried; resolved.
- Rate-limit-exceeded during full re-embed: planned for; backoff handles.
- One auth-expired event (credential rotation lagged); pipeline halted; manual intervention restored within 30 minutes.
- Zero NaN / dimension-mismatch events (the pipeline is stable).

### 9.7 The cost profile

- Daily incremental: ~$0.05.
- Quarterly full re-embed: ~$10.
- Monthly aggregate: ~$15.

Cost is negligible against the broader AI spend.

### 9.8 The platform discipline

- Embedding model pinned per the registry.
- Per-chunk model-version recorded.
- Quarterly drift audit.
- Migration projects planned (no active migration).
- Cost monitored as part of FinOps.

---

## 10. Anti-patterns

### 10.1 "Embedding model alias in production"

The pipeline uses `text-embedding-3-large` without version suffix. Provider updates resolve silently; corpus drifts.

**Corrective.** Pin to full version per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).

### 10.2 "Per-chunk model-version not recorded"

Embeddings are stored without their source model version. Drift detection is impossible; migration is risky.

**Corrective.** Record per chunk per section 4.3.

### 10.3 "Embedding-model change without re-embed"

Team switches embedding models; only new chunks use the new model; corpus has mixed versions; retrieval is inconsistent.

**Corrective.** Embedding-model change requires full re-embed per section 5.2.

### 10.4 "No drift detection"

Embeddings are produced; nobody verifies they match the pin; silent drift accumulates.

**Corrective.** Quarterly audit per section 6.4.

### 10.5 "Migration without shadow"

Embedding model migration done in-place: new model replaces old; if quality regresses, rollback requires another full re-embed.

**Corrective.** Shadow + cutover per section 7.

### 10.6 "Migration without pre-migration eval"

Team commits to migration; runs full re-embed; discovers the new model performs worse. Cost and time wasted.

**Corrective.** Pre-migration eval on sample per section 7.3 before full re-embed.

### 10.7 "Rate limits not monitored"

Pipeline runs; rate limits exceeded; batch failures accumulate; some chunks never embedded.

**Corrective.** Rate-limit-aware batching per section 3.2; quota monitoring; expansion planning.

### 10.8 "Validation absent"

Embeddings written without validation; NaN or wrong-dimension embeddings in the index; retrieval returns garbage.

**Corrective.** Per-embedding validation per section 2.4.

---

## 11. Findings (sprint-assignable)

### EMBED-001 — Severity: Critical
**Finding.** Embedding model referenced by alias; silent version drift possible.
**Recommendation.** Pin per section 4; align with [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
**Owner.** ai-platform-eng, sprint N+1.

### EMBED-002 — Severity: Critical
**Finding.** Per-chunk embedding model version not recorded.
**Recommendation.** Record per chunk per section 4.3.
**Owner.** ai-platform-eng, sprint N+1.

### EMBED-003 — Severity: High
**Finding.** Embedding model change shipped without full re-embed; corpus has mixed versions.
**Recommendation.** Full re-embed migration project per section 7.
**Owner.** ai-platform-eng, sprint N+2.

### EMBED-004 — Severity: High
**Finding.** Drift audit not scheduled.
**Recommendation.** Quarterly drift audit per section 6.4.
**Owner.** ai-platform-eng team lead, sprint N+2.

### EMBED-005 — Severity: High
**Finding.** Per-embedding validation absent; NaN or wrong-dimension embeddings can be indexed.
**Recommendation.** Validation per section 2.4.
**Owner.** ai-platform-eng, sprint N+2.

### EMBED-006 — Severity: High
**Finding.** Rate-limit-aware batching not implemented; batch failures from rate limits.
**Recommendation.** Throughput planning per section 3.
**Owner.** ai-platform-eng, sprint N+2.

### EMBED-007 — Severity: High
**Finding.** No quarantine for failed batches; embedding failures cause document gaps.
**Recommendation.** Quarantine per section 8.3.
**Owner.** ai-platform-eng, sprint N+3.

### EMBED-008 — Severity: High
**Finding.** Embedding-cost telemetry not integrated with FinOps; cost regressions invisible.
**Recommendation.** Per-call cost attribution per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md).
**Owner.** ai-platform-eng + finops, sprint N+3.

### EMBED-009 — Severity: Medium
**Finding.** Embedding model deprecation not tracked; provider deprecation may cause production break.
**Recommendation.** Subscribe to provider announcements per section 4.5; track in model registry.
**Owner.** ai-platform-eng, sprint N+3.

### EMBED-010 — Severity: Medium
**Finding.** Full re-embed pattern not documented; team is uncertain how to migrate.
**Recommendation.** Document per section 7; rehearse migration playbook.
**Owner.** ai-platform-eng + sre, sprint N+3.

### EMBED-011 — Severity: Medium
**Finding.** Concurrency tuning not done; pipeline throughput is suboptimal.
**Recommendation.** Concurrency tuning per section 3.5.
**Owner.** ai-platform-eng, sprint N+4.

### EMBED-012 — Severity: Medium
**Finding.** Migration shadow + cutover pattern not used; risky in-place migrations.
**Recommendation.** Shadow + cutover per section 7.
**Owner.** ai-platform-eng, sprint N+3.

### EMBED-013 — Severity: Medium
**Finding.** Pre-migration eval not done; new model adopted without sample validation.
**Recommendation.** Pre-migration eval per section 7.3.
**Owner.** ai-platform-eng, sprint N+4.

### EMBED-014 — Severity: Medium
**Finding.** Per-batch trace observability absent; embedding failures hard to diagnose.
**Recommendation.** Per-batch trace per section 2.6.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### EMBED-015 — Severity: Medium
**Finding.** Batch-API pricing (50% discount) not used for full re-embeds where latency allows.
**Recommendation.** Use provider batch APIs for non-time-sensitive embedding jobs.
**Owner.** ai-platform-eng + finops, sprint N+5.

### EMBED-016 — Severity: Low
**Finding.** Per-corpus version-distribution dashboard absent; team cannot quickly verify pin compliance.
**Recommendation.** Dashboard per section 6.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### EMBED-017 — Severity: Low
**Finding.** Hot-path embedding (query embedding for retrieval) not optimized; latency higher than necessary.
**Recommendation.** Hot-path tuning per section 3.4.
**Owner.** ai-platform-eng, sprint N+5.

### EMBED-018 — Severity: Low
**Finding.** Embedding-pipeline documentation thin; new engineers cannot understand the discipline.
**Recommendation.** Documentation alongside the pipeline.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building or refactoring an embedding pipeline:

- [ ] **Sprint 0 — design.** Choose embedding model; pin to a version; document.
- [ ] **Sprint 1 — pipeline.** Build the staged pipeline per section 2; per-chunk version recording.
- [ ] **Sprint 1 — validation.** Per-embedding validation per section 2.4.
- [ ] **Sprint 2 — batching.** Tune batch size; rate-limit awareness; concurrency.
- [ ] **Sprint 2 — observability.** Per-batch trace; per-pipeline metrics; cost telemetry.
- [ ] **Sprint 3 — failure handling.** Retry, quarantine, halt per section 8.
- [ ] **Sprint 3 — drift detection.** Quarterly audit pattern; per-corpus version distribution.
- [ ] **Sprint 4 — migration playbook.** Document the migration pattern per section 7; rehearse.
- [ ] **Sprint 4 — deprecation tracking.** Subscribe to provider announcements; tracking in registry.
- [ ] **Sprint 5 — optimization.** Batch-API for non-real-time; throughput optimization.
- [ ] **Ongoing — discipline.** Pin per release; audit per quarter; migrations as structured projects.

A team that completes this sequence has an embedding pipeline that scales and stays consistent. A team that skips pinning and migration discipline pays in silent corpus drift.

---

## 13. References

- This repo: [rag-engineering/ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) — upstream pipeline.
- This repo: [rag-engineering/chunking-engineering.md](./chunking-engineering.md) — produces the chunks this stage embeds.
- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — consumes the embeddings.
- This repo: [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — where embedding model is registered.
- This repo: [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — release-time pinning.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — wrapper for embedding API calls.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost circuits include embedding.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/vector-store-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/vector-store-architecture.md) — where embeddings are stored.
- OpenAI text-embedding documentation.
- Cohere embed documentation.
- Voyage AI, NVIDIA NeMo, BGE embedding documentation.
