# Data Engineering for AI

## What this folder is

The engineering practice of building and maintaining the datasets that AI systems depend on — training data, fine-tune data, eval golden sets, retrieval corpora, labeling pipelines, synthetic data generation, and the data-quality disciplines that keep all of them from drifting silently. The material here is what I put in front of a team when the question is: *we have a 40,000-document retrieval corpus, half of it was scraped from somewhere nobody remembers, the eval golden set has six contributors with no shared scoring rubric, and we just discovered the labeling team has been using a different definition of "high quality" for two months — how do we engineer the data side?*

## The organizing principle

The data side of AI systems is at least as consequential as the model side and gets a fraction of the engineering attention. Most quality problems in production AI systems trace back to a data issue — drift in the retrieval corpus, contamination in the eval set, label noise in the fine-tune data, missing edge cases in the golden set — and not to a model issue. But the data side has fewer satisfying tools, fewer published patterns, and less prestige than the model side, so it tends to be neglected until it breaks.

So the patterns here treat AI data engineering as a *first-class engineering discipline* with the same versioning, lineage, quality gates, and ownership patterns that the broader data engineering canon has matured. The disciplines are mostly translations — dataset versioning is dataset-DVC; data contracts are data contracts; labeling workflows are workflow design — applied to the specific shapes that AI systems consume.

The folder is opinionated about three things specifically. First, that *every dataset the AI system depends on must have a version, an owner, and a refresh discipline* — undocumented "we ingested some docs that one time" corpora are the largest source of unreproducible production behavior. Second, that *eval contamination is the silent killer* — when training, fine-tuning, or example-selection accidentally includes eval cases, the eval becomes meaningless, and the only protection is engineered separation rather than goodwill. Third, that *labeling is an engineering workflow with quality metrics*, not a vendor-procurement problem to be solved by hiring a labeling service.

## Planned documents

- **dataset-versioning.md** *(coming)* — DVC / LakeFS / vendor-versioning patterns for the AI datasets a team owns (eval golden sets, fine-tune datasets, retrieval corpora, labeled data), the integration with model and prompt versioning so a release pins all three, and the dataset-as-release-artifact discipline.
- **labeling-and-annotation.md** *(coming)* — Labeling workflow design: rubric, guidelines, inter-annotator agreement, calibration, vendor-vs-internal-vs-hybrid, the quality-control gates, and the failure modes (rubric drift, guideline ambiguity, annotator fatigue, gaming).
- **synthetic-data-generation.md** *(coming)* — When synthetic data earns its place (augmenting rare-case coverage, generating adversarial test cases, bootstrapping evals before real production data exists), the LLM-as-generator pattern, the calibration discipline (synthetic-distribution must match real-distribution where it counts), and the failure modes (synthetic data that teaches the model to do well only on synthetic data).
- **data-quality-for-ai.md** *(coming)* — Label noise quantification, distribution-drift detection between dataset versions, contamination detection (training data leaked into eval), deduplication, near-duplicate-handling, and the quality dashboard pattern.
- **retrieval-corpus-engineering.md** *(coming)* — Building and maintaining the retrieval corpus: source connectors, freshness SLOs per source, deduplication, content-type normalization, the corpus-as-product discipline (a corpus with an owner, a changelog, and a versioning policy), and integration with the architecture sibling's data-contracts-for-retrieval.
- **eval-data-contamination-prevention.md** *(coming)* — The engineering controls that prevent eval cases from leaking into training, fine-tuning, few-shot examples, or retrieval. Hash-based separation, time-based separation (eval cases from a held-out time window), and the periodic contamination-audit pattern.
- **training-eval-split-discipline.md** *(coming)* — The split discipline for fine-tune workloads (train / validation / test), the held-out test set that is touched only at release time, and the patterns for time-based splits in workloads where the data has temporal structure.
- **data-contracts-for-ai.md** *(coming)* — Treating upstream data sources as contract-bound suppliers: schema, freshness, content-type guarantees, change-notification protocol, the contract-violation alerting pattern, and the integration with the sibling architecture repo's `data-architecture-for-ai/` content.

## How to use this section

**If your team has datasets but no versioning**, `dataset-versioning.md` is the first move. The single biggest leverage in AI data engineering is making "which dataset version was this trained / evaluated on" a question with a deterministic answer.

**If you have labeling vendors or internal annotators**, `labeling-and-annotation.md` is the operational pattern. The pattern that works is rubric + calibration + inter-annotator-agreement metric, not "trust the vendor."

**If you have an eval suite and are not certain it is uncontaminated**, `eval-data-contamination-prevention.md` is the audit. Contamination is usually present at low levels and occasionally present at catastrophic levels.

## What this section is not

- **A general data engineering reference.** Pipelines, warehouses, lakehouses, dbt, Airflow / Dagster / Prefect — these are well-covered elsewhere. This folder is about the *AI-specific overlays* on those tools.
- **A labeling-vendor recommendation.** Vendors change quickly. The discipline (rubric, calibration, IAA, audit) is what matters; the vendor is replaceable.
