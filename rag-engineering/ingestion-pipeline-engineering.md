# Ingestion Pipeline Engineering

> **Audience.** Engineers building or refactoring the document ingestion pipeline for a RAG system. Tech leads whose RAG corpus "just appeared" through ad-hoc scripts and is now causing silent quality issues. **Scope.** The *engineering* practice of the ingestion stage — source connectors, format conversion, chunking dispatch, metadata enrichment, deduplication, idempotency, failure handling. Pair with [chunking-engineering.md](./chunking-engineering.md), [embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md), [retrieval-engineering.md](./retrieval-engineering.md). Not the architectural data-contract decision (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `data-architecture-for-ai/data-contracts-for-retrieval.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The ingestion pipeline is the bottom of the RAG stack and the source of most silent quality issues. Most production RAG quality regressions originate in ingestion — an upstream content source changed format, a chunker's defaults broke on a new document type, deduplication dropped content that was supposed to be kept, an idempotency bug double-ingested. The retrieval system then returns either too little, too much, or wrong content; the model produces a wrong answer; the team investigates and eventually traces back to ingestion.

The discipline this document codifies: ingestion is a *pipeline engineering problem*, not "we wrote a script to load documents." Each stage of the pipeline (fetch, validate, parse, chunk, enrich, index) is its own engineering surface with failure modes and instrumentation. The pipeline as a whole has idempotency guarantees, incremental-update semantics, contract enforcement, and observability.

The architecture sibling's `data-contracts-for-retrieval.md` establishes *what* the corpus must look like. This document is *how* the pipeline reads upstream content and produces a corpus that satisfies the contract.

This document is opinionated about three things:

1. **The contract gate is the pipeline's first stage.** Per [data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md), every document is validated against the contract before any further processing. Non-conforming documents are quarantined, not silently ingested.
2. **Ingestion is idempotent.** Re-running the pipeline (whole or part) produces the same corpus state. Without idempotency, partial-run failures leave the corpus in inconsistent states.
3. **Every stage is observable.** Per-document trace: which source, what format, how many chunks, what enrichment, what failures. Investigations are read-the-trace, not re-run-and-guess.

Structure: (2) the pipeline stages; (3) source connectors; (4) format conversion; (5) deduplication and idempotency; (6) metadata enrichment; (7) failure handling and quarantine; (8) incremental vs full rebuild; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The pipeline stages

Ingestion is a multi-stage pipeline. Each stage has a defined input, output, and failure mode.

### 2.1 The canonical stages

```
Upstream source
    │
    ▼
[Fetch]               — pull from upstream (API, S3, webhook, file drop)
    │
    ▼
[Contract gate]       — validate against the data contract; quarantine on violation
    │
    ▼
[Format conversion]   — normalize from upstream format (HTML/PDF/DOCX/etc.) to canonical
    │
    ▼
[Deduplication]       — detect and skip already-ingested documents
    │
    ▼
[Chunk]               — split into retrievable units (see chunking-engineering.md)
    │
    ▼
[Enrich]              — add metadata (source attribution, classification, lineage tokens)
    │
    ▼
[Embed]               — produce embeddings (see embedding-pipeline-engineering.md)
    │
    ▼
[Index]               — write to vector store + metadata store
    │
    ▼
[Lineage record]      — write lineage entries linking to source
```

Each stage is independently testable and instrumentable. The pipeline as a whole is the composition.

### 2.2 The stage interfaces

Each stage has a typed interface: input documents in, output documents out, side effects to the durable store. The interfaces are stable; stages can be swapped without affecting others.

For example, the format-conversion stage:

```python
class FormatConverter:
    def convert(
        self,
        raw_document: RawDocument,    # bytes + content-type metadata
        contract: DataContract,
    ) -> NormalizedDocument:
        ...
```

The stage knows its input shape, its output shape, and the contract it must satisfy. The stage is unit-testable against synthetic inputs.

### 2.3 The pipeline orchestration

The pipeline orchestrator runs stages in sequence. The orchestrator handles:

- Stage dispatch.
- Per-stage failure handling (skip, retry, quarantine).
- Per-document trace emission.
- Checkpointing (for resumability).
- Throughput control.

The orchestrator is durable-workflow-shaped (per [agent-loop-design.md](../agent-engineering/agent-loop-design.md) section 6 patterns, applied to ingestion): each document's ingestion can survive infrastructure failures and resume from the last checkpoint.

### 2.4 The pipeline deployment

The orchestrator runs as a long-running service or as scheduled batch jobs:

- **Service (continuous).** For sources that produce streaming updates (webhooks, message queues). The service consumes events; processes per-document.
- **Batch (scheduled).** For sources that produce periodic snapshots (daily FDA SPL feed, quarterly clinical-guideline publication). The batch runs on schedule; processes the snapshot.
- **Hybrid.** Most platforms have both — streaming for some sources, batch for others.

The deployment shape is per-source; the pipeline code is shared.

---

## 3. Source connectors

Per source, a connector that knows how to fetch from upstream.

### 3.1 The connector pattern

Each connector implements a common interface:

```python
class SourceConnector:
    def list_available(self, since: datetime) -> Iterator[DocumentMetadata]:
        """List documents available from upstream since the given timestamp."""

    def fetch(self, doc_metadata: DocumentMetadata) -> RawDocument:
        """Fetch a specific document."""

    def authenticate(self) -> Credentials:
        """Authenticate with the upstream system."""
```

Different upstream systems require different implementations:

- HTTP API connectors (pull JSON / XML / etc.).
- S3 connectors (list bucket; read objects).
- Webhook receivers (push from upstream).
- Database replica readers (CDC streams or periodic snapshots).
- File-drop monitors (watch a directory; process new files).

The interface is uniform; implementations are per-source.

### 3.2 The authentication discipline

Each connector authenticates per the upstream's auth model. The credentials are managed per [the broader cloud-security practice](https://github.com/jeremiahredden/cloud-security-reference-architecture):

- API tokens / OAuth refresh tokens stored in secret manager.
- Per-connector rotation schedule.
- Credential expiration alerts.

The connector should fail gracefully on auth failure; failures route to alerts.

### 3.3 The throughput control

Connectors respect upstream rate limits:

- Per-source throughput cap (configured).
- Exponential backoff on rate-limit responses.
- Quotation of remaining quota in metrics.

Without throughput control, ingestion can DoS upstream or get rate-limited and lose data.

### 3.4 The change-detection pattern

For sources that produce periodic snapshots, the connector needs to detect what changed:

- **Last-modified header.** HTTP-friendly; check the last-modified of each document.
- **Content hash.** Compute hash of each fetched document; compare to previously-stored hash.
- **Version field.** If the upstream exposes a version, compare.
- **Full re-fetch with diff.** Sometimes unavoidable for simpler sources.

The change-detection minimizes redundant processing.

### 3.5 The failure-recovery pattern

When a connector fails partway through a fetch:

- Documents already fetched are not re-fetched on retry.
- The next run resumes from where the previous failed (per the change-detection state).
- Unrecoverable failures (auth expired, upstream gone) escalate to alerts.

---

## 4. Format conversion

Upstream content arrives in many formats; the pipeline normalizes to a canonical internal format.

### 4.1 The canonical format

The pipeline's internal canonical format is typically:

- Text content (extracted from whatever original format).
- Structural metadata (sections, headings, lists, tables).
- Source metadata (origin URL or identifier, version, timestamps).
- Content-type tags (for downstream chunking decisions).

The canonical format is stable across the pipeline; downstream stages do not care about the upstream format.

### 4.2 The per-format converters

| Format | Converter approach |
|---|---|
| HTML | DOM parser; extract text per element; preserve heading hierarchy and table structure |
| Markdown | Markdown parser; structure preserved natively |
| PDF | Text extraction (pdfplumber, pypdf, or OCR for scanned PDFs); structure heuristics |
| DOCX | python-docx; element traversal; structure preserved |
| XML | XML parser; XPath-driven extraction per upstream schema |
| Plain text | Pass-through |
| Image (with text) | OCR (Tesseract, AWS Textract, or specialized clinical-OCR) |
| Mixed (e.g., PDFs with embedded images) | Decompose; convert each component; reassemble |

The converters are per-format; the canonical output shape is uniform.

### 4.3 The structural preservation

For RAG quality, structural preservation matters:

- Headings determine section boundaries (used by structure-aware chunking).
- Tables should not be split mid-row.
- Lists should preserve item boundaries.
- Code blocks should not be split mid-block.

The converters extract and preserve structure as metadata. Downstream chunking (per [chunking-engineering.md](./chunking-engineering.md)) uses the structure.

### 4.4 The conversion failure modes

- **Extraction failure.** The converter cannot parse the upstream format (corrupt file, unsupported variant).
- **Partial extraction.** Some content extracted, some lost.
- **Garbled extraction.** Text extracted but quality is poor (e.g., bad OCR on a low-res scan).
- **Structure loss.** Text extracted but structural metadata lost.

The converter reports the failure class; the pipeline decides how to handle (quarantine, partial-ingest with flag, retry with different converter).

### 4.5 The converter as separate artifact

Per-format converters are themselves versioned artifacts. Improvements to the converter (better PDF extraction, better OCR) are deployable; re-running ingestion with the new converter version updates the corpus.

The converter version is recorded in lineage per [lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md); the team can trace which converter version produced which corpus content.

### 4.6 The conversion observability

Per-document conversion produces metrics:

- Conversion time.
- Original format and size.
- Output size (text characters, structural elements detected).
- Failure class (if applicable).

Aggregate dashboards surface conversion-issue patterns (e.g., "PDFs from this upstream source consistently fail" → investigate the upstream or the PDF converter).

---

## 5. Deduplication and idempotency

The pipeline must produce the same corpus state regardless of re-runs.

### 5.1 The deduplication levels

Multiple deduplication concerns:

- **Document-level.** A document already in the corpus (same source ID, same version) should not be re-ingested as a new document.
- **Chunk-level.** A chunk identical to one already in the corpus (e.g., the same boilerplate paragraph appearing in multiple documents) might be deduplicated (or might be kept for context).
- **Embedding-level.** Identical chunks produce identical embeddings; deduplication at the chunk level handles this.

For most workloads, document-level deduplication is the focus; chunk-level deduplication is workload-specific.

### 5.2 The document-level deduplication

Each document has an identity:

- For document-id-bearing sources (every document has a unique upstream ID): match on (source, source_doc_id, source_version).
- For hash-based sources (no stable IDs): match on (source, content_hash).

Re-ingesting a matched document:

- If the new content_hash matches the existing one: no-op (document unchanged).
- If the content_hash differs: this is an update; reprocess (re-chunk, re-embed, re-index) and mark the prior version as superseded.

### 5.3 The idempotency contract

Re-running ingestion (on a full source or a single document) produces the same corpus state. Idempotency depends on:

- Stable IDs (the same source produces the same document IDs across runs).
- Stable hashes (the same content produces the same content hash).
- Atomic writes (a re-ingest does not leave half-written state).

Without idempotency, partial-run failures leave the corpus inconsistent.

### 5.4 The atomic-update pattern

When a document is updated:

1. New chunks are written to the vector store.
2. The document's prior chunks are marked as superseded (not yet deleted).
3. Atomic switch: the document's "current" chunk set updates to the new chunks.
4. After a grace period, the superseded chunks are deleted.

The atomic switch ensures that retrieval never sees a partial update.

For Postgres-pgvector deployments, the atomic switch can be a transaction. For distributed vector stores, the pattern requires more care (often a per-document version field that retrieval filters on).

### 5.5 The full vs incremental run

- **Full run.** All documents from upstream are fetched, processed, indexed. Used for: initial corpus build; embedding model migration; full corpus rebuild.
- **Incremental run.** Only changed documents (since last successful run) are processed. Used for: routine updates.

The full run is more expensive but produces a definitively-correct corpus. The incremental run is cheaper but accumulates state drift if any incremental run fails silently.

Common pattern: incremental for routine updates; full run quarterly to ensure consistency.

### 5.6 The idempotency failure modes

- **Non-stable IDs.** Upstream generates new IDs each time; deduplication fails; corpus grows with duplicates.
- **Non-stable hashes.** Whitespace differences in the same content produce different hashes; deduplication fails.
- **Non-atomic updates.** A re-ingest partially completes; the corpus has mixed-version chunks for one document.

The pipeline detects and reports each (with corrective recommendations).

---

## 6. Metadata enrichment

Per-document metadata that downstream consumers depend on.

### 6.1 The required metadata

Every chunk in the corpus carries:

- **Source attribution.** Which upstream document; which version.
- **Tenant attribution.** Which tenant this content belongs to (or "global" for shared content).
- **Sensitivity classification.** Per the data contract.
- **Content-type tag.** What kind of content (clinical-guideline, patient-education, etc.).
- **Last-update timestamp.** When the source was last modified.
- **Lineage token.** Per [lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — the ID that links retrieval back to the source.

These are non-optional; the contract requires them.

### 6.2 The enrichment sources

- **From upstream.** Most metadata comes from upstream document headers or fields.
- **From the connector.** Source attribution (the connector knows where it fetched from).
- **From the contract.** Tenant attribution (per the contract's tenant rule for this source).
- **From classifiers.** Content-type or sensitivity classifications may use ML classifiers (often a small LLM call).

Each enrichment source is itself documented; failures route to alerts.

### 6.3 The classifier-based enrichment

When metadata requires classification:

- A small LLM call (Haiku-tier per [tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md)) classifies the chunk.
- The classification result is added as metadata.
- Per-classifier accuracy is tracked; quarterly recalibration.

For Meridian: clinical guidelines are classified by topic (cardiology / respiratory / etc.) using a Haiku-based classifier; classification accuracy ~92%; results are added as metadata for filtered retrieval.

### 6.4 The enrichment ordering

Enrichment stages have dependencies:

- Source attribution → before deduplication (so duplicates can be matched by source).
- Tenant attribution → before indexing (so tenant scoping works).
- Content-type tag → before chunking (so structure-aware chunking can pick the right strategy).
- Lineage token → after chunking (per-chunk token).

The pipeline orchestrator handles ordering.

### 6.5 The enrichment failure handling

When enrichment fails:

- Missing optional metadata: document proceeds with the metadata it has.
- Missing required metadata: document is quarantined.

The contract distinguishes required vs optional metadata; the enrichment respects the contract.

---

## 7. Failure handling and quarantine

The pipeline anticipates failures at every stage.

### 7.1 The failure classification

| Class | Examples | Disposition |
|---|---|---|
| Transient | Network timeout fetching from upstream | Retry with backoff |
| Format-error | Document cannot be parsed | Quarantine; alert |
| Contract-violation | Document fails contract validation | Quarantine; alert per data-contracts |
| Enrichment-error | Required classifier failed | Retry; on persistent failure, quarantine |
| Resource-error | Out of memory; disk full | Halt the pipeline; page on-call |
| Unknown | Unexpected exception | Quarantine; log full context; alert |

Each class has a defined disposition.

### 7.2 The quarantine destination

Quarantined documents go to a separate "quarantine" store:

- The document is kept (not silently lost).
- The failure reason is attached.
- The original document is browsable for investigation.
- An alert routes to the contract-owner team (per [data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) section 4.2).

The team can re-process quarantined documents after fixing the cause (upstream change, converter fix, contract amendment).

### 7.3 The fail-soft pattern

For non-critical enrichment failures, the pipeline can fail-soft:

- Document is ingested without the failed enrichment.
- Metadata records the missing enrichment.
- Downstream retrieval handles the missing metadata (may degrade quality).
- A backfill job re-processes the document when the enrichment is fixed.

Fail-soft is for non-critical paths; for required metadata or contract-violating documents, quarantine is the only acceptable disposition.

### 7.4 The pipeline halt pattern

For severe failures (resource exhaustion, mass-quarantine event), the pipeline halts:

- Subsequent documents are not processed.
- The halt event is logged.
- On-call is paged.
- Manual intervention required to resume.

Halt is a safety brake; it prevents the pipeline from churning through a broken state.

### 7.5 The post-incident recovery

After a pipeline failure is resolved:

- The pipeline resumes from the last successful checkpoint.
- Quarantined documents from the failure period are reprocessed.
- A reconciliation run verifies the corpus state matches upstream.

The recovery pattern depends on the pipeline being idempotent (per section 5.3).

---

## 8. Incremental vs full rebuild

The two pipeline modes, each with its place.

### 8.1 Incremental update

The default mode for routine operation:

- Only changed documents (since last successful run) are processed.
- Faster, cheaper, lower-risk for routine operation.
- Depends on the change-detection logic per section 3.4.

### 8.2 Full rebuild

The reset mode for specific scenarios:

- **Embedding model migration.** The pipeline re-embeds the entire corpus with the new model (per [embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md)).
- **Chunking strategy change.** The pipeline re-chunks the entire corpus with the new strategy.
- **Contract change.** The pipeline re-validates the entire corpus against the new contract.
- **Reconciliation.** Periodic verification that the corpus matches upstream state.

Full rebuilds are expensive (hours to days depending on corpus size) but produce a definitively-correct state.

### 8.3 The migration pattern

For changes that require rebuilds (embedding model, chunking, contract):

- The new state is built in shadow (a parallel index).
- Once the shadow is complete, retrieval cuts over (atomic or canary).
- The old state is decommissioned.

The shadow approach means production retrieval is not affected during the migration.

### 8.4 The reconciliation cadence

Periodic full rebuilds (or full-rebuild-equivalent reconciliation):

- Monthly: verify the corpus state matches upstream document inventory.
- Quarterly: full re-ingest with the current pipeline (catches accumulated drift).
- Per-incident: after any pipeline-failure event, reconcile.

The reconciliation catches incremental-update bugs that accumulate silently.

### 8.5 The incremental vs full cost

For Meridian:

- Incremental run (daily): ~5K documents processed in ~10 minutes; cost negligible.
- Full rebuild (quarterly): ~50K documents; ~3 hours; cost ~$50 (mostly embedding-API).

The cost ratios are highly workload-dependent.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The pipeline shape

Meridian's ingestion pipeline:

```
Sources:
  - clinical-guidelines-CMS (HTML)
  - tenant-protocols (per-tenant, varied formats)
  - FDA-SPL-feed (XML)
  - patient-education (Markdown)

         │  (per source)
         ▼
[Source connector]
         │
         ▼
[Contract gate] — per data-contracts-for-retrieval
         │       (quarantine on failure)
         ▼
[Format converter] — per source's format
         │       (HTML/XML/Markdown/PDF-with-OCR)
         ▼
[Deduplication] — by (source, source_doc_id, content_hash)
         │
         ▼
[Chunker] — document-structure-aware per chunking-engineering
         │
         ▼
[Metadata enrichment]
         │  - source attribution
         │  - tenant attribution
         │  - content-type classification (Haiku)
         │  - lineage token generation
         ▼
[Embedder] — per embedding-pipeline-engineering
         │
         ▼
[Indexer] — atomic write to pgvector + metadata tables
         │
         ▼
[Lineage record] — write lineage entries
```

Each stage is its own component; the orchestrator (AWS Step Functions) drives them per document.

### 9.2 The source connectors

| Source | Connector | Cadence |
|---|---|---|
| clinical-guidelines-CMS | HTTP API pull | Weekly (CMS publishes weekly) |
| tenant-protocols | per-tenant CMS API | Per-tenant cadence (configurable) |
| FDA-SPL-feed | XML feed download | Monthly |
| patient-education | Internal Markdown repo | On-commit via webhook |

Each connector handles its source's authentication, throughput, change-detection.

### 9.3 The contract gate in practice

The contract gate validates per the data contract:

- Required fields present?
- Schema match?
- Content-type allowed?
- Size within bounds?

A typical week (2026-Q2):
- ~5K documents processed.
- ~12 quarantined (mostly clinical-guideline updates with new HTML structure; AHA released a new figure-embedding format).
- Quarantined documents routed to the medical-content-licensing team for resolution.

### 9.4 The format converter incidents

In 2026-Q1, a PDF converter incident:
- A clinical-guideline PDF used embedded TIFF images for figures; the PDF converter's image extraction silently produced empty placeholder text.
- The document was ingested with degraded content.
- Quality regression detected via online judge sampling on cardiology-figure-related queries.
- Investigation traced to the PDF converter's TIFF handling.
- Fix: upgraded the converter; re-ingested affected documents.

The incident motivated:
- Per-format conversion-quality metrics (text density per document; outliers flagged).
- A converter-quality-eval suite (sample PDFs from each upstream; converter output evaluated against ground truth).

### 9.5 The idempotency pattern

Meridian's deduplication:

- Each document has `source_doc_id` from upstream.
- The pipeline computes a content_hash on the canonical text.
- On ingest: lookup by (source, source_doc_id). If found and content_hash matches: no-op. If found and content_hash differs: process as update (atomic chunk replacement). If not found: process as new document.

The pattern is durable; re-running ingestion (whole or partial) produces the same corpus state.

### 9.6 The monitoring

Per-pipeline-run dashboards:
- Documents processed.
- Documents quarantined (with classification).
- Documents updated.
- Documents added.
- Per-stage latency.
- Per-stage failure rate.

Quarantine-rate spike alerts route to the contract-owner team.

### 9.7 The full-rebuild cadence

Meridian runs quarterly full rebuilds:

- Driven by `meridian-ingest-rebuild` Step Functions workflow.
- Builds the new corpus in shadow.
- Eval validates the new corpus retrieval quality matches or exceeds the production corpus.
- Cutover: retrieval routes to the new corpus.
- Old corpus decommissioned.

Recent rebuilds:
- 2026-Q1: full rebuild on embedding model pin update. ~50K documents; ~3 hours; ~$48 cost.
- 2026-Q2: scheduled quarterly rebuild; passed eval validation; cutover; no incident.

### 9.8 The platform discipline

- Per-source connector with documented contract.
- Contract gate on every ingest.
- Atomic updates for idempotency.
- Per-stage observability.
- Quarterly full rebuilds.
- Quarantine-rate alerting routed to upstream owners.

---

## 10. Anti-patterns

### 10.1 "Ingestion is a one-time script"

The corpus was built by a script that ran once; updates are manual. The corpus drifts from upstream; nobody knows what's stale.

**Corrective.** Continuous ingestion pipeline with change-detection and idempotency per section 5.

### 10.2 "No contract gate"

Documents are ingested as-is; format-changes and schema-violations slip through; quality regressions trace back to silent ingestion errors.

**Corrective.** Contract gate per section 2.1; quarantine on violation per the data-contracts pattern.

### 10.3 "Non-atomic updates"

Document updates partially complete; the corpus has mixed-version chunks for one document; retrieval returns inconsistent results.

**Corrective.** Atomic update pattern per section 5.4.

### 10.4 "No deduplication"

Documents are re-ingested as duplicates; the corpus has multiple versions of the same content; retrieval returns duplicate chunks.

**Corrective.** Deduplication by (source, source_doc_id, content_hash) per section 5.2.

### 10.5 "Format-conversion silently lossy"

The PDF converter produces empty text on certain PDFs; the documents are ingested as empty; the corpus has phantom content.

**Corrective.** Per-conversion quality metrics; outlier detection; converter-quality eval suite.

### 10.6 "Quarantine destination doesn't exist"

Ingestion failures are logged but documents are lost; there's no way to recover or investigate.

**Corrective.** Quarantine destination with kept documents and failure metadata per section 7.2.

### 10.7 "Pipeline is not idempotent"

Re-running the pipeline produces different corpus states; partial-run failures leave the corpus inconsistent.

**Corrective.** Idempotency discipline per section 5.3; deterministic IDs and hashes; atomic writes.

### 10.8 "No reconciliation"

Incremental updates accumulate state drift over months; the corpus diverges from upstream; nobody notices.

**Corrective.** Periodic full rebuilds or reconciliation per section 8.4.

---

## 11. Findings (sprint-assignable)

### INGEST-001 — Severity: Critical
**Finding.** Ingestion is ad-hoc scripts; no pipeline structure.
**Recommendation.** Build the staged pipeline per section 2; orchestrate per section 2.3.
**Owner.** ai-platform-eng, sprint N+1.

### INGEST-002 — Severity: Critical
**Finding.** No contract gate; non-conforming documents silently ingested.
**Recommendation.** Contract gate per section 2.1; quarantine per section 7.2.
**Owner.** ai-platform-eng + corpus-owners, sprint N+1.

### INGEST-003 — Severity: High
**Finding.** Pipeline is not idempotent; re-runs produce inconsistent corpus state.
**Recommendation.** Idempotency per section 5.3; deterministic IDs; atomic writes.
**Owner.** ai-platform-eng, sprint N+2.

### INGEST-004 — Severity: High
**Finding.** No deduplication; corpus has multiple versions of the same content.
**Recommendation.** Document-level deduplication per section 5.2.
**Owner.** ai-platform-eng, sprint N+2.

### INGEST-005 — Severity: High
**Finding.** Format converters fail silently on certain document types.
**Recommendation.** Per-conversion quality metrics per section 4.6; converter-quality eval suite.
**Owner.** ai-platform-eng, sprint N+2.

### INGEST-006 — Severity: High
**Finding.** Quarantine destination does not exist; failures cause document loss.
**Recommendation.** Quarantine per section 7.2; documents kept; alerts routed.
**Owner.** ai-platform-eng + sre, sprint N+2.

### INGEST-007 — Severity: High
**Finding.** No periodic full rebuild or reconciliation; incremental-update drift accumulates silently.
**Recommendation.** Quarterly full rebuild or reconciliation per section 8.4.
**Owner.** ai-platform-eng, sprint N+3.

### INGEST-008 — Severity: High
**Finding.** Per-source connectors lack rate-limit handling; upstream rate-limit incidents cause pipeline failures.
**Recommendation.** Throughput control per section 3.3.
**Owner.** ai-platform-eng, sprint N+3.

### INGEST-009 — Severity: Medium
**Finding.** Source change-detection is full re-fetch; redundant processing every run.
**Recommendation.** Change-detection per section 3.4 (last-modified, content hash, or version field).
**Owner.** ai-platform-eng, sprint N+3.

### INGEST-010 — Severity: Medium
**Finding.** Format converters lack structural preservation; chunks split mid-table or mid-list.
**Recommendation.** Structural preservation per section 4.3; metadata for downstream chunking.
**Owner.** ai-platform-eng, sprint N+3.

### INGEST-011 — Severity: Medium
**Finding.** Pipeline halt patterns are absent; severe failures churn forever instead of stopping.
**Recommendation.** Pipeline halt per section 7.4; on-call alerts on halt.
**Owner.** ai-platform-eng + sre, sprint N+3.

### INGEST-012 — Severity: Medium
**Finding.** Per-document trace observability is absent; ingestion failures cannot be debugged from the trace.
**Recommendation.** Per-document trace emission per section 2.3; per-stage metrics.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### INGEST-013 — Severity: Medium
**Finding.** Connector credentials are not rotated on schedule; auth-expired events disrupt ingestion.
**Recommendation.** Per-connector rotation schedule per section 3.2.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### INGEST-014 — Severity: Medium
**Finding.** Metadata enrichment is monolithic; enrichment-stage failures fail the entire pipeline.
**Recommendation.** Fail-soft for non-critical enrichment per section 7.3; backfill on resolution.
**Owner.** ai-platform-eng, sprint N+4.

### INGEST-015 — Severity: Medium
**Finding.** Classifier-based enrichment lacks recalibration; classifier accuracy drifts.
**Recommendation.** Quarterly classifier recalibration per section 6.3.
**Owner.** ai-platform-eng, sprint N+4.

### INGEST-016 — Severity: Medium
**Finding.** Quarantine-rate is not alerted; quarantine accumulation goes unnoticed.
**Recommendation.** Quarantine-rate as SLI; alerts on threshold breach.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### INGEST-017 — Severity: Low
**Finding.** Per-source ingestion-throughput dashboards do not exist; capacity planning is reactive.
**Recommendation.** Per-source throughput metrics; capacity planning.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### INGEST-018 — Severity: Low
**Finding.** Migration pattern (shadow + cutover) is undocumented; embedding-model migrations are risky.
**Recommendation.** Migration pattern per section 8.3 documented and rehearsed.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team without a structured ingestion pipeline:

- [ ] **Sprint 0 — design.** Inventory sources; design the staged pipeline; choose orchestrator (Step Functions / Temporal / Airflow / custom).
- [ ] **Sprint 1 — first source.** Build the connector + contract gate + format converter + chunker dispatch for one source.
- [ ] **Sprint 1 — idempotency.** Deterministic IDs; content hashes; atomic writes.
- [ ] **Sprint 2 — observability.** Per-document trace; per-stage metrics; quarantine destination.
- [ ] **Sprint 2 — failure handling.** Retry / quarantine / halt patterns per section 7.
- [ ] **Sprint 3 — additional sources.** Onboard remaining sources; per-source connectors.
- [ ] **Sprint 3 — change detection.** Incremental update pattern per section 3.4.
- [ ] **Sprint 4 — full rebuild pattern.** Documented; rehearsed; quarterly cadence.
- [ ] **Sprint 4 — converter quality.** Per-format conversion-quality metrics; eval suite.
- [ ] **Sprint 5 — enrichment lifecycle.** Classifier recalibration cadence; fail-soft for optional enrichment.
- [ ] **Ongoing — discipline.** Quarantine-rate monitoring; reconciliation cadence; capacity planning.

A team that completes this sequence has an ingestion pipeline that catches upstream changes and produces a corpus that matches the contract. A team that ships ad-hoc-scripts pays in silent quality regressions.

---

## 13. References

- This repo: [rag-engineering/chunking-engineering.md](./chunking-engineering.md) — the next stage.
- This repo: [rag-engineering/embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md) — the embedding stage.
- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — the consumer of the ingested corpus.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](./) (coming) — the diagnostic side.
- This repo: [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/) (coming) — corpus versioning.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — retrieval span attributes that consume ingestion lineage.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) — the contract this pipeline enforces.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — the lineage this pipeline produces.
- AWS Step Functions, Temporal, Airflow documentation for orchestration patterns.
- LlamaIndex, LangChain, Haystack — ingestion framework references.
