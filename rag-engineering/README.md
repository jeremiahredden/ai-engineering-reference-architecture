# RAG Engineering

## What this folder is

The engineering practice of shipping retrieval-augmented systems — ingestion pipelines, embedding strategy, retrieval at scale, reranking, query rewriting, and the observability and operational disciplines that make a RAG system survive contact with production. The material here is what I put in front of a team when the question is: *the notebook RAG demo worked, the production RAG system is returning empty results 8% of the time and wrong results another 12%, and nobody can tell us why — how do we engineer this properly?*

## The organizing principle

RAG looks simple in the architecture diagram and breaks in twenty places in production. The diagram has three boxes (ingest → embed → retrieve) and a single arrow into the prompt. The production system has six pipelines (ingest, embed, index, retrieve, rerank, format) with six failure modes each, two vendor APIs that have rate limits and outages, a vector store that needs capacity management, a chunking strategy that is wrong for the documents it actually receives, an embedding model that is one version behind, and a reranker that earns its cost on the head of the distribution but not the tail.

So the patterns here treat RAG as *a pipeline engineering problem first* and a model engineering problem second. The architectural choices (which vector store, which embedding model, hybrid vs vector-only) belong in the sibling architecture repo; this folder is about *making the chosen architecture work in production*: pipeline observability, retrieval debugging, the discipline of separating retrieval failures from generation failures so the diagnosis is actionable, and the operational practices that keep retrieval quality from drifting silently.

The folder is opinionated about three things specifically. First, that retrieval observability — *what was retrieved on every call, what was used, what was ignored* — is the highest-leverage engineering investment in any RAG system, and most teams do not have it. Second, that chunking is a workload-specific engineering decision that needs to be eval-validated, not a default-pick from a tutorial. Third, that the embedding-model and reranker are *dependencies* in the same sense any library is a dependency — they need version pins, deprecation tracking, and migration playbooks.

## Planned documents

- **ingestion-pipeline-engineering.md** — Document ingestion as a pipeline engineering problem: source connectors, format conversion (HTML / PDF / DOCX / Markdown / images), document-aware splitting, deduplication, metadata enrichment, idempotency for re-ingest, incremental vs full rebuild, and the failure modes (truncation on large docs, encoding issues, OCR errors silently producing garbage).
- **chunking-engineering.md** — Chunking strategies (fixed-window, semantic, document-structure-aware, parent-child, summary-and-detail), the eval pattern that picks the right strategy for the specific document corpus, chunk-size / overlap calibration, and the most common failure mode — chunks that split mid-sentence or mid-table and become non-retrievable.
- **embedding-pipeline-engineering.md** — Embedding generation at scale: batching, throughput optimization, model-version pinning, deprecation tracking, the rebuild-vs-incremental-update pattern, drift detection between embedding model versions, and the operational playbook for migrating an entire production index from embedding model A to embedding model B.
- **retrieval-engineering.md** — Hybrid retrieval implementation (BM25 + vector + filter), the merging strategies (RRF, weighted, score-normalized), multi-stage retrieval (recall → rerank → context-fit), metadata filtering at scale, and the latency / quality trade-offs.
- **reranking-engineering.md** — Reranker selection (Cohere, BGE, in-context-LLM-rerank), placement in the pipeline, the cost / latency / quality trade-offs, the eval pattern that validates the reranker actually lifts quality on your workload (it does not always), and the calibration discipline for reranker-score thresholds.
- **query-rewriting.md** — Query expansion, query decomposition for multi-hop questions, conversation-history-aware rewriting, LLM-as-query-rewriter patterns, and the eval pattern that measures whether rewriting actually improves retrieval (it often does not on simple workloads).
- **retrieval-observability.md** — Per-call traces of what was retrieved, scores, what was used, what was ignored; aggregate dashboards of recall, precision, citation accuracy, empty-result rate, stale-result rate; the alerting on retrieval quality regressions; integration with the sibling `observability-and-telemetry/` content.
- **rag-failure-modes-and-debugging.md** — The twenty production RAG failure modes and the diagnostic patterns for each: empty retrievals, wrong retrievals, stale retrievals, retrieval-but-not-used, used-but-not-cited, off-topic-but-confident, plus the systematic debugging playbook for "why did the model say that?"
- **rag-eval-integration.md** — How RAG-specific eval (retrieval recall, retrieval precision, citation accuracy, faithfulness) integrates with the broader eval practice — separately scoring the retrieval and generation stages so the eval failures point at the right component. Aligned with sibling `eval-engineering/`.

## How to use this section

**If you have a RAG system in production with quality issues of unknown cause**, `retrieval-observability.md` and `rag-failure-modes-and-debugging.md` are the diagnostic pair. Adding retrieval observability is almost always the highest-leverage engineering investment in an existing RAG system.

**If you are building a RAG ingestion pipeline from scratch**, `ingestion-pipeline-engineering.md` and `chunking-engineering.md` are the first reads. Most production RAG quality problems trace back to ingestion and chunking decisions made early and not revisited.

**If you are evaluating whether a reranker pays back on your workload**, `reranking-engineering.md` has the eval pattern. Rerankers add cost and latency; they earn it on some workloads and not on others — the only way to know is to measure on yours.

## What this section is not

- **A vector-store benchmark.** Vector store performance is workload-specific. This folder is about engineering with whatever vector store you have; the choice itself is in the sibling architecture repo's `data-architecture-for-ai/`.
- **A RAG-architecture decision guide.** The pattern-selection decision (naive vs hybrid vs agentic vs graph) is in the architecture sibling's `reference-patterns/`. This folder takes the architectural decision as given and engineers it.
