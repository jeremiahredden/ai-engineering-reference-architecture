# Retrieval Corpus Engineering

> **Audience.** Engineers maintaining a retrieval corpus. Tech leads whose retrieval quality is drifting and they suspect corpus issues. Anyone whose "corpus" was loaded once and never updated, and is now stale. **Scope.** The *engineering* practice of building and maintaining the retrieval corpus: source connectors; freshness SLOs per source; deduplication; content-type normalization; corpus-as-product discipline; integration with the architecture sibling's data-contracts-for-retrieval. Not the embedding strategy (see [rag-engineering/embedding-pipeline-engineering.md](../rag-engineering/embedding-pipeline-engineering.md)). Not the retrieval engineering itself (see [rag-engineering/retrieval-engineering.md](../rag-engineering/retrieval-engineering.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The retrieval corpus is the data underneath every RAG workload. Without engineering discipline:

- Sources go stale; users get outdated answers.
- Duplicates accumulate; retrieval returns redundant content.
- New sources added without ownership; nobody knows when to update.
- The "we scraped 10k documents that one time" pattern.

With engineering discipline:

- Each source has an owner, refresh cadence, freshness SLO.
- Deduplication maintains the corpus.
- Content-type normalization makes retrieval consistent.
- The corpus is a product, with a roadmap.

This document covers the engineering.

This document is opinionated about four things:

1. **The corpus is a product, not data.** Owner; roadmap; quality discipline.
2. **Per-source freshness SLO is the load-bearing metric.** Without it, freshness drifts.
3. **Content-type normalization is essential.** Heterogeneous content degrades retrieval.
4. **Deduplication and quality are continuous, not one-time.** Cross-link to [data-quality-for-ai.md](./data-quality-for-ai.md).

Structure: (2) the corpus as product; (3) source connectors; (4) freshness SLOs; (5) deduplication; (6) content-type normalization; (7) corpus lifecycle; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The corpus as product

Treating the corpus as a managed artifact.

### 2.1 The corpus role

The corpus is:

- Input to retrieval.
- Source of truth for downstream AI consumers.
- A maintained artifact with quality requirements.

Treating it as data-blob "we have somewhere" causes quality issues.

### 2.2 The corpus owner

Each corpus has an owner team:

- Responsibility for quality.
- Maintenance.
- Roadmap.

Often: data engineering team for the platform-wide corpus; AI platform team for feature-specific corpora.

### 2.3 The corpus catalog

Per organization:

```
clinical-corpus (v23.1.0):
  Owner: data engineering team
  Sources: EHR clinical notes, formulary, guidelines
  Total documents: 40,000
  Last update: 2026-05-27
  Refresh cadence: nightly

reference-corpus (v5.2.0):
  Owner: clinical informatics team
  Sources: regulatory references, internal procedures
  Total documents: 1,200
  Last update: 2026-05-15
  Refresh cadence: weekly
```

Inventory.

### 2.4 The corpus changelog

Per corpus version:

```markdown
# clinical-corpus Changelog

## v23.1.0 - 2026-05-27
- Added 3,200 new clinical notes from May.
- Removed 12 deprecated documents.
- Re-deduplicated; removed 45 near-duplicates.

## v23.0.0 - 2026-05-01 [BREAKING]
- Restructured chunking from 500 to 800 token chunks.
- Re-embedded entire corpus.
```

Per changelog discipline.

### 2.5 The corpus roadmap

Per quarter:

- Sources to add.
- Sources to deprecate.
- Quality improvements.
- Scaling considerations.

Forward-looking.

### 2.6 The "we never planned the corpus" reality

For teams without corpus product discipline:

- "We added this source because someone asked."
- "We added another because it seemed useful."
- "Nobody owns it."

Resolve by establishing ownership.

### 2.7 The corpus per workload

For multi-workload platforms:

- Shared corpus for general clinical knowledge.
- Per-feature corpora for specific workloads.
- Per-tenant corpora for tenant-specific data.

Per workload structure.

### 2.8 The corpus governance

Decisions about:

- Adding a new source.
- Deprecating a source.
- Quality threshold changes.

Through governance process; documented.

### 2.9 The "the corpus has grown too large to maintain" reality

When corpus grows:

- Storage cost.
- Embedding cost (re-embedding on updates).
- Query latency.

Periodic curation:

- Remove stale content.
- Archive old versions.

Per-corpus growth strategy.

---

## 3. Source connectors

How data flows into the corpus.

### 3.1 The connector architecture

Per source:

```
Source system (EHR, file store, etc.)
    ↓
Connector (extract; transform; load)
    ↓
Corpus storage
```

Standard ETL pattern.

### 3.2 The connector responsibilities

- Extract from source (API, file, etc.).
- Transform (parse, normalize).
- Load (into corpus storage).
- Track lineage (where did this document come from).

### 3.3 The connector types

- **Pull-based.** Connector polls source on schedule.
- **Push-based.** Source pushes events; connector ingests.
- **Hybrid.** Mix.

Per source.

### 3.4 The connector resilience

Connectors fail:

- Source unavailable.
- Network issues.
- Schema changes upstream.

Mitigations:

- Retry with backoff.
- Dead-letter queue.
- Alerting on persistent failure.

### 3.5 The schema-change handling

When upstream schema changes:

- Connector breaks.
- Alert.
- Updated connector deployed.

Cross-link to [data-contracts-for-ai.md](./data-contracts-for-ai.md).

### 3.6 The connector cataloging

Per connector:

```yaml
connector: ehr-clinical-notes
source: EHR system
schedule: nightly @ 02:00
owner: data-eng-team
last_successful_run: 2026-05-27T02:15:00Z
average_records: 800/day
```

Tracked.

### 3.7 The new-connector decision

When considering a new source:

- Is the data valuable enough?
- Is the connector maintainable?
- Does owner exist?

Approval workflow.

### 3.8 The connector retirement

For deprecated sources:

- Connector stopped.
- Historical data retained per retention policy.
- Catalog entry marked deprecated.

### 3.9 The "we have 50 connectors; some are broken" mature-but-not-ideal state

Common for older platforms:

- Old connectors quietly broken.
- Half-stale data.

Diagnose; address.

---

## 4. Freshness SLOs

How fresh the corpus is per source.

### 4.1 The freshness SLO

Per source:

```yaml
clinical_notes_freshness:
  source: EHR system
  target: < 24h staleness P99
  alert_when: > 36h staleness

formulary_freshness:
  source: pharmacy db
  target: < 1 week
  alert_when: > 2 weeks
```

Per source; tracked.

### 4.2 The freshness measurement

For each item:

- Source-side updated_at.
- Corpus-side ingested_at.
- Staleness = corpus_now - source_updated_at.

P99 staleness per source.

### 4.3 The freshness vs cost trade-off

More frequent updates:

- Higher infrastructure cost.
- Better freshness.

Per source:

- Critical (clinical alerts): real-time or near-real-time.
- Normal (most clinical content): daily.
- Slow (policy docs): weekly.

Match SLO to need.

### 4.4 The "we never updated; it's all stale" recovery

For corpora that haven't been refreshed:

- Sample to check staleness.
- Identify worst sources.
- Re-ingest priority sources.

Catch up; then maintain.

### 4.5 The freshness-incident response

When freshness SLO violated:

- Connector failed? Restart.
- Source down? Coordinate.
- Schema change? Update connector.

Per-cause response.

### 4.6 The "data is stale but quality is fine" assessment

Sometimes stale data is acceptable:

- If users don't notice.
- If staleness is bounded.

But: monitor; review.

### 4.7 The cross-source freshness dependencies

For corpora with multiple sources:

- Overall freshness = worst source.
- Or: per-source SLOs aggregated.

Per-corpus design.

### 4.8 The "we promised freshness but can't deliver" deficit

If consistently missing SLO:

- Reduce promise (longer staleness allowed).
- Increase infrastructure (more frequent connector runs).

Honest.

---

## 5. Deduplication

For retrieval corpora.

### 5.1 The dedup motivation

Without dedup:

- Same content from multiple sources.
- Retrieval returns multiple copies.
- Token cost on retrieval (multiple chunks of same content).
- User sees redundancy.

### 5.2 The exact-match dedup

Hash-based:

- Each document hashed.
- Same hash = duplicate.

Catches obvious.

### 5.3 The near-duplicate dedup

Semantic-similarity:

- Embed each document.
- Cluster by similarity.
- Document pairs with similarity > 0.95 are near-duplicates.

Cross-link to [data-quality-for-ai.md §5](./data-quality-for-ai.md).

### 5.4 The dedup-decision policy

For near-duplicates, keep:

- Most-recent.
- Best-quality (rated).
- Canonical version.
- Source with most authority.

Per-workload.

### 5.5 The cross-source dedup

For documents from multiple sources:

- Same document may be indexed in multiple places.
- Dedup across.

### 5.6 The dedup-frequency

Per corpus:

- On every update (per-batch dedup).
- Periodically (weekly full-corpus dedup).

Per-corpus design.

### 5.7 The dedup-vs-retrieval-quality

Aggressive dedup can hurt:

- Remove a document that's specifically referenced in some queries.
- Lose context.

Conservative threshold; review dedup decisions.

### 5.8 The "we have 2 versions of a policy; both are needed" exception

For some workloads:

- Both old and new policy needed (e.g., for historical reasoning).
- Don't dedup these.

Per-workload exceptions.

### 5.9 The dedup-pipeline placement

In the pipeline:

```
Source → Connector → Dedup → Embedding → Storage
```

Standard.

### 5.10 The "we ran dedup once; never again" mistake

Without ongoing dedup:

- New duplicates accumulate.
- Corpus quality degrades.

Cross-link to [data-quality-for-ai.md §5.10](./data-quality-for-ai.md).

---

## 6. Content-type normalization

Making heterogeneous content homogeneous.

### 6.1 The heterogeneity problem

Sources have different shapes:

- HTML docs.
- PDFs.
- Text files.
- Database records.
- Markdown.

Retrieval works best on consistent format.

### 6.2 The normalization steps

For each source:

- Extract text (handle PDFs, HTML, etc.).
- Clean (remove boilerplate, fix encoding).
- Standardize (consistent format).
- Add metadata (source, date, etc.).

### 6.3 The text-extraction

Per source type:

- PDF: pdfminer / PyMuPDF.
- HTML: BeautifulSoup; readability.
- Markdown: parse to text.
- Word: docx2txt.

Tools per type.

### 6.4 The boilerplate removal

For HTML / web content:

- Remove navigation menus.
- Remove ads.
- Remove sidebars.
- Keep main content.

Tools: readability libraries.

### 6.5 The encoding normalization

- UTF-8 default.
- Fix mojibake (mis-encoded characters).
- Normalize whitespace.

Standard text-cleaning.

### 6.6 The structural normalization

- Headers consistent.
- Tables → structured representation (or markdown).
- Lists handled.

Per-content-type.

### 6.7 The chunking strategy

For long documents:

- Chunked into smaller units.
- Cross-link to [rag-engineering/chunking-engineering.md](../rag-engineering/chunking-engineering.md).

The corpus stores chunks (often) or full documents (sometimes).

### 6.8 The metadata enrichment

Per document:

- Source.
- Date created / updated.
- Author / authority.
- Tenant (for multi-tenant).
- Document type.

Stored alongside content.

### 6.9 The "we have 12 different doc types; each needs custom handling" reality

Maturity:

- Document-type catalog.
- Per-type extractor.
- Standardized output format.

Engineering investment.

### 6.10 The validation

Post-normalization:

- Spot-check normalized output.
- Detect failures (parsing errors).
- Flag for manual review.

---

## 7. Corpus lifecycle

Document and corpus evolution.

### 7.1 The document lifecycle

Per document:

- Ingestion (first added).
- Updates (re-ingested with new content).
- Deprecation (marked obsolete; kept for history).
- Removal (deleted).

States.

### 7.2 The document-update handling

When a document is updated:

- New version ingested.
- Old version archived (or deleted).
- Embedding re-computed.
- Caches invalidated (cross-link to [ai-architecture-reference-architecture / cost-and-performance-architecture / caching-tiers.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/cost-and-performance-architecture/caching-tiers.md)).

Cascade.

### 7.3 The document-deletion

When a document is deleted from source:

- Marked deleted in corpus.
- Removed from retrieval index (or marked deleted).
- Caches invalidated.

### 7.4 The "we keep all versions" history

Some workloads need history:

- Audit (what was the policy at time X).
- Compliance.

Versioned storage:

- Each version retained.
- Time-based query support.

### 7.5 The corpus-as-snapshot

Periodically:

- Snapshot full corpus.
- Versioned (cross-link to [dataset-versioning.md §3.2](./dataset-versioning.md)).

Reproducible.

### 7.6 The corpus rebuild

Occasionally:

- Full rebuild from source.
- Re-extract; re-normalize; re-embed.

Reasons:
- Embedding model changed.
- Chunking strategy changed.
- Cleanup of accumulated cruft.

### 7.7 The "we rebuild the corpus every quarter" cadence

Some teams:

- Quarterly full rebuild.
- Forces cleanup.
- Catches drift.

For others:

- Continuous; never rebuild.

Per-team.

### 7.8 The "the corpus has 50% deprecated content" sign

If many documents are deprecated:

- Periodic cleanup needed.
- Or accept the noise.

Cleanup discipline.

---

## 8. Worked Meridian example

Meridian's corpus engineering.

### 8.1 The corpora

```
clinical-corpus (v23.1.0):
  Sources: 4 (EHR clinical notes, formulary, guidelines, policies)
  Documents: 40,000
  Updates: nightly (clinical notes); weekly (formulary, guidelines); monthly (policies)
  Owner: Data Engineering team
  Freshness SLO: clinical notes <24h; formulary <1 week
  Dedup: 1.5% near-duplicates removed
  Storage: per-tenant namespace in Pinecone

reference-corpus (v5.2.0):
  Sources: 2 (regulatory references, internal procedures)
  Documents: 1,200
  Updates: weekly
  Owner: Clinical Informatics team
  Freshness SLO: <1 week
  Dedup: minimal (few duplicates expected)

internal-policy-corpus (v3.1.0):
  Sources: 1 (Confluence / internal wiki)
  Documents: 400
  Updates: weekly
  Owner: IT team
  Freshness SLO: <2 weeks
```

Three corpora.

### 8.2 The connectors per source

```
EHR clinical notes connector:
  Type: API-based pull
  Schedule: Nightly @ 02:00
  Records/run: ~800
  Resilience: retry 3x; DLQ
  Owner: Data Eng

Formulary connector:
  Type: Database-based pull
  Schedule: Weekly @ Sun 01:00
  Records/run: ~150
  Owner: Data Eng

Guidelines connector:
  Type: File-based pull (S3)
  Schedule: Weekly
  Records/run: ~20 documents
  Owner: Clinical Inf

Policies connector:
  Type: API-based pull (Confluence)
  Schedule: Monthly @ 1st 01:00
  Records/run: ~30 documents
  Owner: IT
```

Per source.

### 8.3 The freshness tracking

Per source:

- Freshness P50 / P99 dashboard.
- Per-source SLO compliance.
- Alerts on SLO violation.

Reviewed monthly.

### 8.4 The Q1 2026 freshness incident

Clinical notes connector failed for 48 hours:

- Source EHR API change broke connector.
- Detection: freshness alert at 36h.
- Mitigation: connector updated; backfill.
- Recovery: 12 hours to catch up.

Lessons:

- Connector resilience improved.
- Schema-change detection added.

### 8.5 The dedup pipeline

Per nightly update:

- New documents ingested.
- Dedup against existing corpus.
- Removed: ~1% of new (exact match) + 0.5% (near-dup).

Quarterly: full-corpus dedup; ~0.3% removed.

### 8.6 The content-type normalization

EHR clinical notes:

- Source: structured XML.
- Extract: text + metadata.
- Normalize: standardized clinical-note format.
- Metadata: patient_id (encrypted), encounter_id, date, author.

Formulary:

- Source: tabular.
- Normalize: per-medication structured entry.
- Metadata: medication name, generic / brand, formulary tier.

Guidelines:

- Source: PDF / Word.
- Extract: text via pdfminer.
- Normalize: section structure preserved.
- Metadata: title, version, last-updated.

Per-type.

### 8.7 The Q2 2026 corpus rebuild

Embedding model upgraded (BGE-large v2):

- Decision: rebuild entire clinical corpus with new embeddings.
- Cost: ~$5k in embedding compute.
- Quality: retrieval recall@10 improved from 0.85 to 0.92.

Worth it.

### 8.8 The "we want to add a new corpus" governance

When new corpus considered:

- Justification.
- Owner identified.
- SLOs defined.
- Approved by AI platform + data engineering.

Not "let's just add it."

### 8.9 The corpus management cost

- Data engineering FTE: ~0.5 allocated to corpus management.
- Storage: ~$200/month.
- Embedding compute: ~$500/month.
- Pinecone vector store: ~$1k/month.

Total: modest; high value.

### 8.10 The lessons

- Corpus as product transforms quality.
- Per-source freshness SLOs catch issues early.
- Periodic full rebuild handles structural changes.
- Connector resilience is essential.

---

## 9. Anti-patterns

### 9.1 The "we ingested some docs that one time" un-owned corpus

**Pattern.** Corpus has no owner; nobody updates.

**Corrective.** Owner per §2.2.

### 9.2 The unmonitored connector

**Pattern.** Connector failed; nobody knew; data stale.

**Corrective.** Per-source freshness alerts per §4.

### 9.3 The unbounded growth

**Pattern.** Corpus grows; nobody curates; deprecated content accumulates.

**Corrective.** Lifecycle management per §7.

### 9.4 The "we never re-embedded" stale-embeddings

**Pattern.** New embedding model; corpus uses old embeddings.

**Corrective.** Rebuild per §7.6.

### 9.5 The "different doc types; same handling" naïveté

**Pattern.** PDFs, HTML, plain text all parsed identically; quality varies.

**Corrective.** Per-type normalization per §6.

### 9.6 The "we dedup once at ingestion; never again" leakage

**Pattern.** Dedup at first ingest; future updates introduce duplicates.

**Corrective.** Ongoing dedup per §5.6.

### 9.7 The schema-change cascade

**Pattern.** Upstream schema changes; connector silently broken; data half-ingested.

**Corrective.** Schema-change detection per §3.5.

### 9.8 The cross-source duplication unhandled

**Pattern.** Same document in EHR and in policy database; duplicates in corpus.

**Corrective.** Cross-source dedup per §5.5.

### 9.9 The "the corpus is fine; we don't need a roadmap" complacency

**Pattern.** No roadmap; reactive maintenance.

**Corrective.** Roadmap per §2.5.

### 9.10 The "we'll fix it later" stale-source

**Pattern.** Connector broken; "we'll fix it next sprint"; sprints pass.

**Corrective.** Severity per §4.5; address.

---

## 10. Findings (sprint-assignable)

### DATA-RCE-001 — Severity: Critical
**Finding.** Corpus has no owner.
**Recommendation.** Per §2.2.
**Owner.** data engineering + AI platform, sprint N+1.

### DATA-RCE-002 — Severity: Critical
**Finding.** Per-source freshness SLOs absent.
**Recommendation.** Per §4.
**Owner.** data engineering, sprint N+1.

### DATA-RCE-003 — Severity: Critical
**Finding.** Connector monitoring absent.
**Recommendation.** Per §3.4 and §4.5.
**Owner.** SRE + data engineering, sprint N+1.

### DATA-RCE-004 — Severity: High
**Finding.** Dedup not ongoing.
**Recommendation.** Per §5.6.
**Owner.** data engineering, sprint N+2.

### DATA-RCE-005 — Severity: High
**Finding.** Content-type normalization inconsistent.
**Recommendation.** Per §6.
**Owner.** data engineering, sprint N+2.

### DATA-RCE-006 — Severity: High
**Finding.** Schema-change detection absent.
**Recommendation.** Per §3.5.
**Owner.** data engineering + observability, sprint N+2.

### DATA-RCE-007 — Severity: High
**Finding.** Corpus catalog absent.
**Recommendation.** Per §2.3.
**Owner.** data engineering, sprint N+2.

### DATA-RCE-008 — Severity: Medium
**Finding.** Corpus changelog absent.
**Recommendation.** Per §2.4.
**Owner.** data engineering, sprint N+3.

### DATA-RCE-009 — Severity: Medium
**Finding.** Document lifecycle (update / deprecate / delete) ad-hoc.
**Recommendation.** Per §7.
**Owner.** data engineering, sprint N+3.

### DATA-RCE-010 — Severity: Medium
**Finding.** Cross-source dedup absent.
**Recommendation.** Per §5.5.
**Owner.** data engineering, sprint N+3.

### DATA-RCE-011 — Severity: Medium
**Finding.** Corpus roadmap absent.
**Recommendation.** Per §2.5.
**Owner.** data engineering, sprint N+3.

### DATA-RCE-012 — Severity: Medium
**Finding.** Periodic full rebuild not scheduled.
**Recommendation.** Per §7.6.
**Owner.** data engineering, sprint N+3.

### DATA-RCE-013 — Severity: Medium
**Finding.** Embedding-model upgrade doesn't trigger corpus rebuild.
**Recommendation.** Per §8.7.
**Owner.** data engineering + AI platform, sprint N+4.

### DATA-RCE-014 — Severity: Medium
**Finding.** Per-tenant corpus separation not enforced.
**Recommendation.** Per [multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-vector-namespacing.md). Owner: AI platform + security, sprint N+4.

### DATA-RCE-015 — Severity: Low
**Finding.** Corpus governance for new sources absent.
**Recommendation.** Per §2.8 and §3.7.
**Owner.** data engineering + AI platform, sprint N+5.

### DATA-RCE-016 — Severity: Low
**Finding.** Corpus growth not tracked.
**Recommendation.** Per §2.9.
**Owner.** data engineering + FinOps, sprint N+5.

### DATA-RCE-017 — Severity: Low
**Finding.** Cost of corpus operations not surfaced.
**Recommendation.** Per §8.9.
**Owner.** FinOps + data engineering, sprint N+6.

### DATA-RCE-018 — Severity: Low
**Finding.** Annual corpus review absent.
**Recommendation.** Roadmap per §2.5; annual review.
**Owner.** data engineering + product, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Assign corpus owner per §2.2.**
- [ ] **Build corpus catalog per §2.3.**
- [ ] **Define per-source freshness SLOs per §4.**
- [ ] **Implement connector monitoring per §3.4.**
- [ ] **Implement schema-change detection per §3.5.**
- [ ] **Implement dedup pipeline per §5.**
- [ ] **Implement content-type normalization per §6.**
- [ ] **Define document lifecycle per §7.**
- [ ] **Build corpus changelog per §2.4.**
- [ ] **Plan periodic full rebuild per §7.6.**
- [ ] **Per-tenant separation enforced.**
- [ ] **Annual corpus review.**

---

## 12. References

**In this folder.**
- [dataset-versioning.md](./dataset-versioning.md) — corpus versioning.
- [data-quality-for-ai.md](./data-quality-for-ai.md) — corpus quality.
- [labeling-and-annotation.md](./labeling-and-annotation.md) — labeled corpus data.
- [data-contracts-for-ai.md](./data-contracts-for-ai.md) — upstream data contracts.

**Elsewhere in this repo.**
- [rag-engineering/retrieval-engineering.md](../rag-engineering/retrieval-engineering.md) — retrieval depends on corpus.
- [rag-engineering/embedding-pipeline-engineering.md](../rag-engineering/embedding-pipeline-engineering.md) — embedding the corpus.
- [rag-engineering/chunking-engineering.md](../rag-engineering/chunking-engineering.md) — chunking.
- [rag-engineering/ingestion-pipeline-engineering.md](../rag-engineering/ingestion-pipeline-engineering.md) — ingestion.

**Sibling repos.**
- [ai-architecture-reference-architecture / data-architecture-for-ai / data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) — data-contracts architecture.
- [ai-architecture-reference-architecture / data-architecture-for-ai / vector-store-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/vector-store-architecture.md) — vector store.
- [ai-architecture-reference-architecture / data-architecture-for-ai / freshness-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/freshness-architecture.md) — freshness architecture.

**External.**
- Pinecone, Weaviate, OpenSearch documentation.
- pdfminer, PyMuPDF, BeautifulSoup, readability docs.
- ETL / data-pipeline patterns.
