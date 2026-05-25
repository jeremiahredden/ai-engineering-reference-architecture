# Chunking Engineering

> **Audience.** Engineers building or refactoring the chunking stage of a RAG pipeline. Tech leads whose retrieval quality is poor because chunks split mid-table or mid-sentence and nobody noticed. **Scope.** The *engineering* practice of chunking — strategy selection, calibration, structural awareness, failure modes. Pair with [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) (upstream), [embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md) (downstream), [retrieval-engineering.md](./retrieval-engineering.md) (consumer). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Chunking is the most under-engineered stage of most production RAG pipelines. Teams adopt the default chunking strategy from their framework (fixed 1000-token windows with 100-token overlap), notice retrieval quality is okay, and move on. Months later, retrieval recall plateaus, the team investigates, and traces back to chunks split mid-sentence, tables ripped in half, code blocks bisected.

The pattern is: chunking quality directly bounds retrieval quality. A chunk that's not individually retrievable (because it lacks context or coherent meaning) cannot be retrieved well no matter how good the embedding model or reranker is. Chunking is the bottleneck nobody notices until they investigate it specifically.

The discipline this document codifies: chunking is a workload-specific eval-validated decision, not a default. The team measures multiple strategies; picks the one that lifts retrieval recall on their corpus; tunes parameters per document type; preserves structure where it matters.

This document is opinionated about three things:

1. **Default chunking is rarely optimal.** Fixed-window chunking is a starting point but is almost always beaten by document-structure-aware chunking on real corpora.
2. **Chunking strategy is workload-specific.** Different document types within the same corpus may benefit from different chunking strategies. The pipeline can dispatch per content-type.
3. **Chunking decisions are eval-validated.** The team picks a strategy because the eval shows it lifts recall, not because it sounds right.

Structure: (2) the chunking strategies; (3) strategy selection; (4) chunk-size and overlap calibration; (5) structural awareness; (6) per-content-type dispatch; (7) failure modes; (8) eval integration; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The chunking strategies

Multiple strategies; each fits different document shapes.

### 2.1 Fixed-window

Split into chunks of N tokens (or characters) with optional overlap of M tokens.

```
[chunk 1: tokens 0-999] [overlap: tokens 900-1099] [chunk 2: tokens 1000-1999] ...
```

**Sweet-spot workload.** Uniform-format documents with no significant structure (plain text, simple prose).

**What it gives you.** Simplest implementation; predictable chunk sizes; works for any input.

**What it cannot do well.** Splits mid-sentence, mid-paragraph, mid-table. Loses structural context.

**Default parameters.** 500-1000 tokens per chunk; 50-100 token overlap. These defaults are rarely optimal for production workloads.

### 2.2 Sentence-based

Split on sentence boundaries; aggregate sentences into chunks up to a target size.

```
[chunk 1: sentences 1-12 (~800 tokens)] [chunk 2: sentences 13-23 (~750 tokens)] ...
```

**Sweet-spot workload.** Prose-heavy documents where sentence is the meaningful unit (articles, narratives, conversational transcripts).

**What it gives you.** Chunks don't split mid-sentence; chunk sizes vary slightly but stay close to target.

**What it cannot do well.** Doesn't respect higher-level structure (sections, lists, tables). May split a list across chunks.

### 2.3 Document-structure-aware

Split on structural boundaries (headings, sections, paragraphs); aggregate up to target size; preserve structural metadata.

```
[chunk 1: § 3.2 Introduction (paragraph + table + heading)] [chunk 2: § 3.3 Method] ...
```

**Sweet-spot workload.** Well-structured documents (clinical guidelines, technical documentation, legal documents, structured reports).

**What it gives you.** Chunks correspond to meaningful structural units. Tables, lists, code blocks preserved. Structural metadata (section title, heading hierarchy) attached to chunks.

**What it costs.** Requires the format converter (per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 4) to preserve structure. Chunk sizes vary more than fixed-window.

**Why this is usually the right choice.** On structured corpora (most enterprise content), this beats fixed-window measurably on retrieval recall.

### 2.4 Parent-child (hierarchical)

Two-level chunking: smaller "child" chunks for retrieval (focused, embeddable); larger "parent" chunks for context expansion (provide surrounding context to the model).

```
Document
├── Parent chunk 1 (large, contextual)
│   ├── Child chunk 1a (small, retrievable)
│   ├── Child chunk 1b
│   └── Child chunk 1c
└── Parent chunk 2
    ├── Child chunk 2a
    └── Child chunk 2b

Retrieval: matches against child chunks; returns parent chunk for context.
```

**Sweet-spot workload.** Workloads where retrieval needs to find specific content but the model benefits from surrounding context (technical reference, regulated content with cross-references).

**What it gives you.** Best of both: precise retrieval (small chunks) + rich context (large chunks). Often lifts answer quality on multi-hop or context-dependent questions.

**What it costs.** More complex pipeline (two chunk hierarchies); higher storage (parent chunks stored alongside children); slightly more retrieval logic.

### 2.5 Summary-and-detail

Each large unit (section, document) has a summary chunk (compressed) and detail chunks (full content).

```
Document
├── Summary chunk (concise overview; high retrieval priority)
└── Detail chunks (full content; included when summary is relevant)
```

**Sweet-spot workload.** Long-document workloads (research papers, full books) where a summary serves as a navigation aid.

**What it gives you.** Retrieval-from-summary identifies which documents matter; detail is fetched for matched documents. Reduces context budget consumption.

**What it costs.** Requires summary generation (LLM call per unit); summaries can drift from detail content; pipeline complexity.

### 2.6 Sliding-window (paragraph-level overlap)

Like fixed-window but at paragraph granularity:

```
[chunk 1: paragraphs 1-5] [chunk 2: paragraphs 4-8] [chunk 3: paragraphs 7-11] ...
```

**Sweet-spot workload.** Long-form prose where context flows across paragraphs (long-form articles, narrative documents).

**What it gives you.** Cross-chunk context (each chunk shares paragraphs with neighbors); robust to retrieval cutting off mid-discussion.

**What it costs.** Chunk count grows; storage cost increases; some redundancy in retrieved content.

### 2.7 The strategy decision

| Document type | Default strategy |
|---|---|
| Structured (clinical guidelines, technical docs, legal) | Document-structure-aware |
| Prose-heavy (articles, narratives) | Sentence-based or sliding-window |
| Highly variable / unknown structure | Document-structure-aware with fallback to fixed-window |
| Long reference documents | Parent-child or summary-and-detail |
| Short Q&A or FAQ entries | One chunk per entry (no further split) |
| Code / structured data | Specialized (per-function, per-record) |

The choice is workload-specific; multiple strategies often coexist in one pipeline (per-content-type dispatch per section 6).

---

## 3. Strategy selection

The team picks a strategy through evaluation, not preference.

### 3.1 The selection workflow

For a new corpus or feature:

1. **Inventory document types** in the corpus.
2. **Hypothesize strategies** per document type (using the decision matrix in section 2.7 as a starting point).
3. **Build an eval set** representative of the corpus and retrieval workload (per [golden-set-design.md](../eval-engineering/golden-set-design.md)).
4. **For each candidate strategy:** chunk the corpus; embed; index; run the eval; measure retrieval recall, precision, MRR.
5. **Compare strategies on the eval metrics.** The winner is the strategy whose eval scores justify its operational complexity.
6. **Pick; document the choice; deploy.**

The workflow is structured; the choice is data-driven.

### 3.2 The minimum viable comparison

For a new corpus, compare at least:

- Fixed-window (baseline).
- Document-structure-aware (most common production winner).
- A third candidate appropriate to the document type (parent-child, sliding-window, etc.).

Three candidates is the minimum to surface the strategy decision space. Fewer is too few for a confident choice; more is overkill for an initial comparison.

### 3.3 The comparison metrics

- **Retrieval recall@K.** Did the eval's gold-source chunks appear in the top-K? Most directly correlated with chunking quality.
- **Retrieval precision@K.** Of the top-K returned, what fraction were actually relevant?
- **MRR.** Where in the ranking did the gold chunk appear? Measures whether good chunks rank high.
- **Answer correctness (downstream).** Given the retrieved chunks, does the model produce a correct answer? End-to-end signal.
- **Chunk count and storage.** Operational cost of the strategy.

A strategy may win on recall but lose on precision (returns more chunks, some irrelevant); the team weighs the trade-off.

### 3.4 The per-tenant variant evaluation

For multi-tenant systems, the eval may need per-tenant evaluation:

- Different tenants' content may have different structure.
- One strategy may be optimal for most tenants but suboptimal for one.
- Per-tenant chunking dispatch (per section 6) may be warranted.

### 3.5 The re-evaluation triggers

The chunking strategy is re-evaluated when:

- The corpus changes meaningfully (new document types added, existing documents change format).
- The embedding model changes (new model may benefit from different chunk sizes).
- The retrieval pattern changes (hybrid vs vector-only may favor different chunk shapes).
- Retrieval quality drifts (online judge flags retrieval-stage issues).

Annual re-evaluation as a baseline cadence; trigger-based for the rest.

---

## 4. Chunk-size and overlap calibration

Within a chosen strategy, the parameters matter.

### 4.1 The chunk-size trade-off

- **Smaller chunks** (200-500 tokens): more precise retrieval; the matched chunk is highly relevant. But less context per chunk; the model may need multiple chunks to answer.
- **Larger chunks** (1000-2000 tokens): more context per chunk; the model can answer from fewer chunks. But less precise retrieval; the embedding represents a broader topic.

The sweet spot is workload-dependent. For Meridian clinical: 500-800 tokens per chunk (medium).

### 4.2 The overlap trade-off

- **Larger overlap** (20-30% of chunk size): more context continuity; chunks share boundary content; reduces mid-thought splits. But more chunks (higher storage and retrieval cost).
- **Smaller overlap** (5-10%): less redundancy; less storage; but more risk of mid-thought cuts.

For Meridian: 10-15% overlap (modest).

### 4.3 The calibration workflow

Within the chosen strategy:

1. **Define the parameter grid.** Chunk sizes: [300, 500, 800, 1200]. Overlaps: [50, 100, 150].
2. **For each parameter combination:** chunk, embed, index, eval.
3. **Compare on retrieval metrics.** Pick the combination that optimizes recall while staying within precision tolerances.

The grid search is straightforward; the work is in setting up the eval pipeline to compare quickly.

### 4.4 The token vs character debate

Chunk sizes can be in:

- **Tokens.** Aligned with embedding model context limits; consistent across documents.
- **Characters.** Simpler to compute; varies by tokenization across documents.

Tokens is the better choice. Embedding models have token limits; computing tokens (using the embedding model's tokenizer) is straightforward.

### 4.5 The model-context-window consideration

Some retrieval patterns return many chunks to the model:

- Top-10 chunks at 800 tokens each = 8000 tokens of context, plus the prompt, plus the question.
- If the model's context window is 200K tokens, plenty of room.
- If the model's context window is 8K tokens, this is the entire window.

The chunking decision interacts with the retrieval top-K and the model's context budget. The team plans the joint budget per the architecture sibling's `cost-and-performance-architecture/`.

---

## 5. Structural awareness

The discipline that beats default chunking.

### 5.1 What "structural" means

The pipeline preserves and uses:

- **Headings and section hierarchy.** Section title is attached as metadata to chunks within that section.
- **Lists.** List items kept together; if a list is long, the chunk may include the list intro + several items.
- **Tables.** Tables kept whole if they fit in the chunk size; if not, the table heading and column names are preserved at chunk boundaries.
- **Code blocks.** Preserved as units.
- **Citations and references.** Preserved as units.

The format converter (per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 4.3) extracts the structure; the chunker uses it.

### 5.2 The chunking algorithm

A document-structure-aware chunker walks the document tree:

```
For each section:
    If section fits in target chunk size: emit as one chunk.
    Else: split section by sub-section / paragraph; aggregate up to target size.
For each list (within a section):
    If list fits: keep with surrounding paragraph.
    Else: emit list as its own chunk (with list intro context).
For each table (within a section):
    If table fits in remaining chunk space: include.
    Else: emit table as its own chunk; include section heading and table heading as context.
For each code block:
    Treat as atomic; do not split mid-block.
```

The algorithm is workload-tunable; the principle is "respect the structure."

### 5.3 The metadata attached

Each chunk carries:

- Section path (e.g., `"3.2.1 Heart Failure Discharge Bundle"`).
- Heading hierarchy as breadcrumbs.
- Position within the document.
- Adjacent-chunk identifiers (for context expansion).

The metadata is used:
- During retrieval (for metadata filtering — e.g., "only chunks from the procedures section").
- During context formatting (for the model to see the section context).
- For citation accuracy (the chunk's source location is precise).

### 5.4 The chunk-coherence check

A chunk that is structurally incoherent (e.g., split mid-table) is detectable:

- Chunk starts mid-sentence.
- Chunk contains "(continued)" without prior context.
- Chunk has unbalanced HTML / markup.

Per-chunk validation flags incoherent chunks; investigation traces back to format-conversion or chunking bugs.

### 5.5 The trade-off

Structural awareness adds pipeline complexity. The cost: more code in the chunker (per-content-type handling); some chunks are smaller than target (because section boundaries don't always align with target size).

The benefit: measurably higher retrieval recall on structured corpora. The cost is paid back many times over by the quality lift.

---

## 6. Per-content-type dispatch

One pipeline; multiple chunking strategies based on content type.

### 6.1 The pattern

The ingestion pipeline's chunker dispatches per content-type tag (assigned by enrichment per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 6):

```
content_type = doc.metadata["content_type"]

if content_type == "clinical-guideline":
    chunker = DocumentStructureAwareChunker(target_size=600, overlap=80)
elif content_type == "patient-education":
    chunker = SentenceBasedChunker(target_size=400, overlap=50)
elif content_type == "drug-interaction":
    chunker = SemanticUnitChunker()  # one chunk per interaction record
elif content_type == "tenant-protocol":
    chunker = DocumentStructureAwareChunker(target_size=500, overlap=60)
else:
    chunker = FixedWindowChunker(target_size=800, overlap=100)  # fallback

chunks = chunker.chunk(doc)
```

Each content type uses its optimal strategy. The pipeline orchestrator handles the dispatch.

### 6.2 The benefit

Workloads with multiple content types in one corpus benefit substantially. A single strategy applied to all content types is a compromise; per-type dispatch optimizes each.

### 6.3 The cost

- More chunker code; each per-type strategy is its own component.
- Eval per content type; not a single eval covers all types.
- Maintenance: new content types require new chunker decisions.

### 6.4 The fallback

A default chunker handles content types without specific dispatch. The default is conservative (document-structure-aware with mid-range parameters); good enough for new content types until per-type optimization is done.

### 6.5 The per-content-type eval

Per-content-type chunking decisions need per-content-type eval cases. The golden set per [golden-set-design.md](../eval-engineering/golden-set-design.md) section 3.5 supports this with class tags.

---

## 7. Failure modes

Chunking failures are often silent.

### 7.1 The failure catalog

- **Mid-sentence cuts.** Chunk boundaries land mid-sentence; retrieved chunks read awkwardly; embedding quality suffers.
- **Tables split mid-row.** Table semantics destroyed; retrieval cannot identify the table.
- **Lists broken across chunks.** Items missing context; partial lists in different chunks.
- **Code blocks bisected.** Code becomes uninterpretable; documentation chunks become useless.
- **Sections lost.** A document section is processed but not chunked (e.g., a sub-section is ignored).
- **Section context lost.** Chunks lack the section title metadata; retrieval cannot reason about location.
- **Empty chunks.** Format conversion produced empty text; chunker emits an empty chunk.
- **Oversized chunks.** A single section larger than the embedding model's context limit; the embed step rejects.
- **Undersized chunks.** Very small chunks (10-token paragraphs); embedding quality is poor for these.

Each failure has detection patterns.

### 7.2 The detection patterns

Per-chunk validation:

- Empty / near-empty chunk → reject.
- Oversized chunk → split further or alert.
- Starts mid-sentence → check structural-aware processing; possibly re-chunk.
- Contains "see Figure X" without the figure → expected for text-only content; the lineage should track the reference.

Per-corpus validation:

- Chunk-size distribution: outliers (very small or very large) are investigated.
- Per-document chunk count: documents producing zero chunks or unusually many are investigated.
- Average chunk size by content type: drift over time may indicate a converter or chunker issue.

### 7.3 The mid-sentence-cut detection

A common silent failure:

- Heuristic: chunk starts with lowercase letter or punctuation; chunk ends mid-sentence.
- Per-corpus: percentage of chunks starting/ending mid-sentence (target < 5%).
- Outliers investigated; usually a structural-preservation issue in the converter.

### 7.4 The table-integrity detection

For corpora with meaningful tables:

- Per-document table count vs per-chunk table-marker count: should match.
- Tables split across chunks: counted; alerts on increase.

### 7.5 The chunk-quality eval

The team's eval suite includes cases that specifically exercise chunk quality:

- Cases where the answer depends on table content: tests table preservation.
- Cases where the answer requires a list item plus its parent: tests list preservation.
- Cases where the answer is in a specific section: tests structural awareness.

Failures on these cases trace to chunking issues.

---

## 8. Eval integration

Chunking decisions are eval-validated.

### 8.1 The chunking eval

Per [eval-of-rag.md](../eval-engineering/eval-of-rag.md), RAG cases have `expected_sources` (which chunks should be retrieved). The retrieval recall measurement evaluates the chunking + retrieval pipeline together; a chunking issue reduces recall.

For chunking-specific analysis:

- Run the eval suite with the current chunking.
- Run with an alternative chunking.
- Compare per-case-class recall.
- Identify cases where chunking choice matters most.

### 8.2 The chunking strategy change

When the team considers changing chunking:

- Build the alternative chunking in shadow (new chunks emitted to a parallel index).
- Run the eval suite against both indexes.
- Compare; pick the winner.
- If the new chunking wins: schedule the cutover (full re-chunk per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 8.2).

The shadow approach prevents production disruption during evaluation.

### 8.3 The per-case-class chunking signal

Per [golden-set-design.md](../eval-engineering/golden-set-design.md) section 3.5, cases have class tags. Per-class retrieval recall surfaces which case classes are most affected by chunking:

- Lookup cases: low sensitivity to chunking (small chunks suffice).
- Multi-source cases: moderate sensitivity.
- Multi-hop cases: high sensitivity (chunking affects which combinations of chunks can be retrieved together).

The class-specific signal informs chunking parameter tuning.

### 8.4 The continuous monitoring

Production-side signal:

- Empty-retrieval rate (per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md)).
- Online judge recall on production traffic.
- User feedback on missed-information cases.

Drift detection triggers chunking re-evaluation.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The chunking decisions

Meridian's per-content-type chunking:

| Content type | Strategy | Target size | Overlap | Rationale |
|---|---|---|---|---|
| Clinical guidelines | Document-structure-aware | 600 tokens | 80 | Well-structured with sections; preserve table integrity for dosing tables |
| Tenant protocols | Document-structure-aware | 500 tokens | 60 | Similar to guidelines but typically shorter sections |
| Drug-interaction records | Semantic-unit | One per interaction | n/a | Each interaction is atomic; no further split |
| Patient-education articles | Sentence-based | 400 tokens | 50 | Prose-heavy; patient-readable; smaller chunks for precise retrieval |
| FDA-SPL feed | Document-structure-aware | 700 tokens | 100 | Mixed structure; standardized XML schema preserved |

Each was chosen via the selection workflow (section 3.1); the eval validated the choice.

### 9.2 The chunking decision history

For clinical guidelines (the most consequential):

- 2024 (pilot): fixed-window 1000/100. Baseline recall: 71%.
- 2025-Q1: switched to document-structure-aware 800/80. Recall: 86% (+15 points).
- 2025-Q3: tuned to 600/80 (smaller chunks for finer retrieval). Recall: 89% (+3 points).
- 2026-Q1: stayed at 600/80 (no further improvement from further tuning).

The progression demonstrates: defaults are a starting point; structure-awareness lifts substantially; parameter tuning produces marginal additional gains.

### 9.3 The structural preservation in practice

Clinical guidelines have tables (dosing tables, medication interaction tables). The chunker:

- Detects tables in the upstream HTML.
- If table fits in target chunk size: include with surrounding context.
- If table exceeds target: emit as a standalone chunk with the section heading and table title as preserved metadata.

The result: dosing tables are individually retrievable; the model can cite the specific table when answering dosing questions.

### 9.4 The mid-sentence cut detection

Production monitoring shows:

- Mid-sentence chunk starts: < 2% of chunks (well within tolerance).
- Outliers investigated when the rate ticks up. In 2025-Q4, the rate jumped to 8% after a clinical-guideline CMS update changed HTML structure; investigation traced to the format converter not preserving paragraph boundaries in the new HTML; converter updated.

### 9.5 The chunking failure incident

In 2026-Q2, retrieval quality on cardiology questions dropped (online judge alert per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md)). Investigation:

- Per-content-type retrieval recall: cardiology guidelines dropped from 89% to 78%.
- Per-document chunk count: cardiology guidelines were producing fewer chunks than expected.
- Format conversion check: AHA released updated cardiology guidelines with embedded SVG figures; the converter's text extraction was failing on certain SVG-heavy sections.
- Fix: converter updated; affected documents re-chunked; recall restored to 89%.

The incident reinforced: chunking quality depends on conversion quality; both need monitoring.

### 9.6 The per-tenant variant

Tenant-protocol chunking is per-tenant-content-type but uses the same strategy as global clinical guidelines (with slightly different target size). Per-tenant differences in protocol structure are accommodated by the document-structure-aware chunker's adaptability.

No per-tenant chunking divergence has been needed; the dispatcher uses one strategy across all tenant protocols.

### 9.7 The platform discipline

- Per-content-type chunking decisions documented.
- Eval-validated; quarterly re-evaluation cadence.
- Per-chunk validation flags structural failures.
- Per-document and per-corpus chunk-quality metrics surfaced.
- Re-chunking is treated as a corpus migration per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) section 8.3.

---

## 10. Anti-patterns

### 10.1 "Default chunking is good enough"

Team uses the framework's default (1000/100 fixed-window); never re-evaluates. Retrieval quality is leaving recall points on the table.

**Corrective.** Strategy selection workflow per section 3.1; evaluate alternatives.

### 10.2 "Same chunking for all content types"

Mixed-format corpus uses one chunking strategy. Some content types are well-served; others suffer.

**Corrective.** Per-content-type dispatch per section 6.

### 10.3 "Structure ignored"

Chunker doesn't see structural metadata; tables and lists are split mid-content; chunks read awkwardly.

**Corrective.** Document-structure-aware chunking per section 5; converter preserves structure.

### 10.4 "Chunk parameters set once"

Target size and overlap were set at adoption; never re-tuned. Embedding model has changed; retrieval pattern has changed; parameters are no longer optimal.

**Corrective.** Calibration per section 4.3; periodic re-tuning.

### 10.5 "Chunk-quality monitoring absent"

Chunks are produced; nobody verifies quality. Mid-sentence cuts, empty chunks, oversized chunks accumulate.

**Corrective.** Per-chunk validation per section 7.2; per-corpus distribution monitoring.

### 10.6 "Chunking strategy change without re-chunk"

Team changes the chunking strategy; the new strategy applies to new documents only; the corpus has mixed-strategy chunks. Retrieval quality is inconsistent.

**Corrective.** Full re-chunk on strategy change per section 8.2.

### 10.7 "Oversized chunks rejected silently"

Chunks larger than the embedding model's limit are rejected by the embed step; the corpus has gaps; nobody knows.

**Corrective.** Pre-embed size check; alerts on rejection; chunker tuned to stay within limits.

### 10.8 "Chunking decisions undocumented"

Engineer who chose the chunking left; nobody knows why these parameters were chosen; nobody dares to change them.

**Corrective.** Document per-content-type chunking decisions with rationale (which eval scores justified the choice).

---

## 11. Findings (sprint-assignable)

### CHUNK-001 — Severity: Critical
**Finding.** Default chunking in production; never evaluated against alternatives.
**Recommendation.** Strategy selection workflow per section 3.1; comparison of at least three candidates.
**Owner.** ai-platform-eng, sprint N+1.

### CHUNK-002 — Severity: Critical
**Finding.** Same chunking applied across mixed-format corpus; some content types underperform.
**Recommendation.** Per-content-type dispatch per section 6.
**Owner.** ai-platform-eng, sprint N+2.

### CHUNK-003 — Severity: High
**Finding.** Structural metadata is not preserved; chunks split mid-table and mid-list.
**Recommendation.** Document-structure-aware chunking per section 5; converter updates per [ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md).
**Owner.** ai-platform-eng, sprint N+2.

### CHUNK-004 — Severity: High
**Finding.** Per-chunk validation is absent; chunk quality issues accumulate silently.
**Recommendation.** Validation per section 7.2; per-corpus distribution monitoring.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### CHUNK-005 — Severity: High
**Finding.** Mid-sentence-cut rate is not measured; structural-preservation failures unnoticed.
**Recommendation.** Detection per section 7.3; alerts on rate increase.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### CHUNK-006 — Severity: High
**Finding.** Chunking parameters were set at adoption; never re-tuned despite embedding model changes.
**Recommendation.** Annual calibration per section 4.3.
**Owner.** ai-platform-eng, sprint N+3.

### CHUNK-007 — Severity: High
**Finding.** Chunking strategy change shipped without full re-chunk; corpus has mixed-strategy chunks.
**Recommendation.** Migration per section 8.2; full re-chunk on strategy change.
**Owner.** ai-platform-eng, sprint N+3.

### CHUNK-008 — Severity: High
**Finding.** Oversized chunks rejected silently by embed step; corpus has gaps.
**Recommendation.** Pre-embed size check; alerts on rejection; chunker tuned.
**Owner.** ai-platform-eng, sprint N+2.

### CHUNK-009 — Severity: Medium
**Finding.** Chunking eval cases are absent; chunking-quality issues are not surfaced by the eval suite.
**Recommendation.** Cases that test table preservation, list integrity, structural awareness per section 7.5.
**Owner.** ai-platform-eng + clinical-domain-experts, sprint N+3.

### CHUNK-010 — Severity: Medium
**Finding.** Chunk-size distribution is not monitored; outliers go undetected.
**Recommendation.** Per-corpus distribution dashboards per section 7.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### CHUNK-011 — Severity: Medium
**Finding.** Chunking eval re-run is not scheduled; strategy choice may have drifted.
**Recommendation.** Annual re-evaluation per section 3.5; trigger-based for model / corpus changes.
**Owner.** ai-platform-eng team lead, sprint N+4.

### CHUNK-012 — Severity: Medium
**Finding.** Chunking decisions are undocumented; team cannot articulate why current parameters were chosen.
**Recommendation.** Document per-content-type decisions; commit alongside the pipeline.
**Owner.** ai-platform-eng, sprint N+4.

### CHUNK-013 — Severity: Medium
**Finding.** Parent-child or summary-and-detail strategies were not considered; long-document handling is suboptimal.
**Recommendation.** Evaluate per section 2.4 / 2.5 for long-document content types.
**Owner.** ai-platform-eng, sprint N+4.

### CHUNK-014 — Severity: Medium
**Finding.** Chunker dispatch fallback is too aggressive (default fixed-window); new content types underperform.
**Recommendation.** Fallback to document-structure-aware per section 6.4.
**Owner.** ai-platform-eng, sprint N+4.

### CHUNK-015 — Severity: Medium
**Finding.** Chunk size is in characters, not tokens; alignment with embedding model context limit is approximate.
**Recommendation.** Token-based sizing per section 4.4.
**Owner.** ai-platform-eng, sprint N+4.

### CHUNK-016 — Severity: Low
**Finding.** Per-chunk section-path metadata is not attached; downstream context formatting cannot show section.
**Recommendation.** Section path as chunk metadata per section 5.3.
**Owner.** ai-platform-eng, sprint N+5.

### CHUNK-017 — Severity: Low
**Finding.** Chunking framework choice is constrained by initial framework selection; alternatives not considered.
**Recommendation.** Periodic review of chunking framework / library choices.
**Owner.** ai-platform-eng team lead, sprint N+5.

### CHUNK-018 — Severity: Low
**Finding.** Chunking documentation does not explain trade-offs; new engineers do not understand the decisions.
**Recommendation.** Documentation alongside the chunker code.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team using default chunking:

- [ ] **Sprint 0 — inventory.** Document types in the corpus; current chunking strategy.
- [ ] **Sprint 1 — eval setup.** Build a RAG-eval suite with `expected_sources` per [eval-of-rag.md](../eval-engineering/eval-of-rag.md).
- [ ] **Sprint 1 — comparison.** Evaluate at least 3 strategies on the eval; pick winner.
- [ ] **Sprint 2 — parameter calibration.** Within winner strategy, tune target size and overlap.
- [ ] **Sprint 2 — structural awareness.** If winner is document-structure-aware, ensure converter preserves structure.
- [ ] **Sprint 3 — per-content-type dispatch.** Per-type chunking decisions.
- [ ] **Sprint 3 — validation.** Per-chunk validation; per-corpus monitoring.
- [ ] **Sprint 4 — full re-chunk.** Migration per section 8.2; shadow + cutover.
- [ ] **Sprint 4 — observability.** Mid-sentence-cut rate; chunk-size distribution; per-content-type metrics.
- [ ] **Sprint 5 — recalibration cadence.** Annual re-evaluation scheduled; trigger-based re-evaluation.
- [ ] **Ongoing — discipline.** Chunking decisions eval-validated; chunking changes are migrations.

A team that completes this sequence has chunking that earns its retrieval recall. A team that ships defaults pays in unrealized recall.

---

## 13. References

- This repo: [rag-engineering/ingestion-pipeline-engineering.md](./ingestion-pipeline-engineering.md) — the upstream pipeline.
- This repo: [rag-engineering/embedding-pipeline-engineering.md](./embedding-pipeline-engineering.md) — the downstream pipeline.
- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — the consumer of chunks.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](./) (coming) — diagnostic patterns.
- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — the eval discipline for chunking evaluation.
- This repo: [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — case structure with expected_sources.
- LangChain text-splitter documentation.
- LlamaIndex node-parser documentation.
- Anthropic Contextual Retrieval reference.
- Pinecone, Weaviate documentation on chunking patterns.
