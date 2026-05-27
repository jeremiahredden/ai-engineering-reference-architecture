# Dataset Versioning

> **Audience.** Engineers whose "which dataset version was this trained on" question is currently answered with "let me check the slack history." Tech leads whose eval results vary by 5-10% between runs and they suspect the dataset has shifted. Anyone whose AI release process pins the model and prompt but not the data. **Scope.** The *engineering* practice of dataset versioning: DVC / LakeFS / vendor-versioning patterns for the AI datasets a team owns (eval golden sets, fine-tune datasets, retrieval corpora, labeled data); integration with model and prompt versioning; dataset-as-release-artifact. Not the data quality discipline itself (see [data-quality-for-ai.md](./data-quality-for-ai.md), companion). Not the architectural data contracts (see sibling [data-architecture-for-ai/data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Most AI engineering teams version their code and their models. Far fewer version their data.

The consequences:

- Eval results vary between runs (data shifted).
- Fine-tune produces different outputs between training runs (data shifted).
- Retrieval quality changes silently (corpus shifted).
- "Which data trained this model" answer is uncertain.

Dataset versioning fixes these. The discipline:

- Every dataset has a version (semantic, timestamp, or hash-based).
- Releases pin the dataset version alongside model and prompt.
- Changes are tracked and reviewable.
- Old versions retained for reproducibility.

The general data-engineering canon has mature tools for this (DVC, LakeFS, Pachyderm). The AI-specific overlays:

- AI datasets are typically smaller than enterprise data warehouses (helpful: simpler).
- AI datasets need to be paired with models / prompts / evals (integration challenge).
- Some AI datasets have lineage requirements (training data → model → output traceability).

This document covers the engineering practice.

This document is opinionated about four things:

1. **Every dataset that AI depends on must be versioned.** Eval golden sets, fine-tune data, retrieval corpora, labeled data — all of them.
2. **Releases pin all three: model, prompt, dataset.** Otherwise reproducibility is impossible.
3. **Old versions are retained.** Compliance and reproducibility require it.
4. **Versioning is engineering hygiene, not a research nicety.** Production AI requires it.

Structure: (2) what to version; (3) versioning schemes; (4) tooling options; (5) integration with model + prompt versioning; (6) dataset-as-release-artifact; (7) retention policy; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. What to version

The AI datasets that need versioning.

### 2.1 Eval golden sets

The test cases used to evaluate AI systems:

- Each case: input + expected output (or rubric).
- Curated over time.
- Drift = different evals over time.

Cross-link to [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md).

### 2.2 Fine-tune datasets

For fine-tuned models:

- Training data.
- Validation data.
- Test data (held out).

Each version of the dataset → can re-train if needed.

### 2.3 Retrieval corpora

Documents loaded into vector stores:

- Document sources.
- Versions over time (documents updated).
- Embedding model used (different model = different vectors).

Cross-link to [retrieval-corpus-engineering.md](./retrieval-corpus-engineering.md).

### 2.4 Labeled data

Data labeled for various uses:

- Classification labels.
- Annotation labels.
- Rubric scores.

Cross-link to [labeling-and-annotation.md](./labeling-and-annotation.md).

### 2.5 Few-shot example sets

Few-shot examples (cross-link to [ai-architecture-reference-architecture / context-and-prompt-architecture / few-shot-vs-fine-tune-vs-system-prompt.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/few-shot-vs-fine-tune-vs-system-prompt.md)):

- Example sets versioned.
- Per-feature curated.

### 2.6 Reference data

For tools / lookups:

- Formulary tables.
- Policy documents.
- Configuration data.

If AI references it: versioned.

### 2.7 The "this isn't actually data" exclusion

Some things are versioned elsewhere:

- Code: source control.
- Models: model registry (cross-link to [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md)).
- Prompts: prompt versioning (cross-link to [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md)).

These have their own versioning systems; dataset versioning complements.

### 2.8 The catalog

Per team:

```
Datasets versioned by Meridian:
  - care-coordinator-eval-golden-set (v8.2.0)
  - patient-api-chat-eval-set (v3.1.0)
  - document-classification-fine-tune-data (v12.0.0)
  - clinical-corpus (v23.1.0)
  - meridian-formulary (v5.4.0)
  - few-shot-care-coordinator (v4.0.0)
  ...
```

Catalogued; tracked.

---

## 3. Versioning schemes

How datasets are versioned.

### 3.1 Semantic versioning

```
v1.0.0
v1.1.0 — added 20 new cases
v1.2.0 — refined rubric (backwards-compatible)
v2.0.0 — major rubric change (breaking)
```

Familiar; works for AI datasets.

Cross-link to [ai-architecture-reference-architecture / context-and-prompt-architecture / prompt-as-api-discipline.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/prompt-as-api-discipline.md) for semver philosophy.

### 3.2 Timestamp-based

```
2026-05-27-1430
2026-05-28-0900
```

For frequently-updated datasets (retrieval corpora, news feeds).

Coordinates with provider updates.

### 3.3 Hash-based

```
v.sha:a3f7b2c
v.sha:b8e9c1d
```

Each version is the content hash.

Immutable; deterministic.

For ML / research workflows.

### 3.4 The choice per dataset

```
Dataset                          Versioning
─────────────────────────────────────────────────
Eval golden set                  Semver (versions when cases change)
Fine-tune dataset                Semver
Retrieval corpus                 Timestamp (frequently updated)
Labeled data                     Semver
Few-shot examples                Semver
Reference data                   Timestamp or semver (depends on stability)
```

Per dataset's evolution characteristics.

### 3.5 The "we use timestamps for everything" simplification

Some teams: timestamp all datasets.

- Simple; consistent.
- Loses semantic information (was this a breaking change?).
- Acceptable for many teams.

### 3.6 The dataset-version identifier in metadata

Each dataset version has:

- Version identifier.
- Created timestamp.
- Author / owner.
- Change summary (changelog).
- Hash of contents.

Stored as metadata.

### 3.7 The dataset-version comparison

For two versions:

- Diff: what changed.
- Stats: cases added / removed / modified.
- Quality metrics: distribution, label distribution.

Auditable.

### 3.8 The dataset-version sources of truth

Versioning info stored where:

- Per dataset, a manifest file.
- Plus a central registry (catalog of all datasets).

Discoverable.

---

## 4. Tooling options

What tools support dataset versioning.

### 4.1 DVC (Data Version Control)

Git-like for data:

- Tracks data as files (or blobs).
- Stores in S3 / GCS / etc.
- Per-version metadata.

**Pros.** Familiar; Git-native; widely adopted.
**Cons.** For large datasets, can be slow.

### 4.2 LakeFS

Git-like for data lakes:

- Branches / merges on data.
- Suited for larger-scale data.

**Pros.** Larger-scale; full lake versioning.
**Cons.** Heavier infrastructure.

### 4.3 Pachyderm

Data versioning + pipelines:

- Data versioned alongside the pipelines processing it.
- Reproducible.

**Pros.** Full reproducibility.
**Cons.** Heavier; less common in 2026.

### 4.4 Vendor-specific

Some platforms:

- Weights & Biases artifacts.
- MLflow datasets.
- Hugging Face datasets (with version history).

**Pros.** Integrated with ML workflows.
**Cons.** Vendor lock-in.

### 4.5 The custom approach

Some teams:

- Manifests + S3 buckets with version-prefix.
- No third-party tool.

**Pros.** No dependency; explicit.
**Cons.** Reinvention; less feature-rich.

### 4.6 The choice per team

Most teams: DVC or vendor-specific.

For 2026 platforms with serious AI investment: dedicated tooling (DVC, LakeFS).

For smaller teams: manifests + S3.

### 4.7 The tool selection criteria

- Dataset size (DVC handles up to TB; LakeFS for PB).
- Team familiarity (Git users → DVC).
- Integration with existing stack.
- Compliance requirements (audit logs).

### 4.8 The migration path

From custom → DVC:

- Tag existing data as v1.0.0.
- Initialize DVC.
- Apply going forward.
- Backfill if needed.

Not painful for moderate datasets.

---

## 5. Integration with model + prompt versioning

How datasets compose with other artifacts.

### 5.1 The release manifest

A release pins all three:

```yaml
care-coordinator-release-v14.2.0:
  model: anthropic:claude-sonnet:4-6
  prompt: care-coordinator-v2.3.0
  datasets:
    eval_golden_set: care-coordinator-eval-v8.2.0
    few_shot_examples: care-coordinator-few-shot-v4.0.0
    retrieval_corpus: clinical-corpus-v23.1.0
  deployed_at: 2026-05-10
```

Reproducible: re-pulling all three artifacts re-creates the release.

### 5.2 The release as the unit of versioning

The release is the artifact:

- Code version.
- Model version.
- Prompt version.
- Dataset version(s).
- Eval suite version.

All change together; pinned together.

### 5.3 The "what changed in this release" diff

For comparing releases:

- Code: git diff.
- Model: catalogue diff.
- Prompt: prompt-changelog diff.
- Datasets: dataset-version diff.

Auditable; investigable.

### 5.4 The rollback consistency

Roll back to previous release:

- Revert all artifacts together.
- Don't roll back model without rolling back data.

Cross-link to [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md §7.8](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md).

### 5.5 The "we changed the eval but not the model" case

Sometimes only one artifact changes:

- Eval suite updates (adding cases).
- New release with same model + same prompt + updated eval.

Valid; minor version bump.

### 5.6 The cross-team dataset sharing

When multiple features use the same dataset:

- Single source of truth.
- Multiple releases pin the same dataset version.

Versioning shared.

### 5.7 The catalog-of-releases

Per feature, a catalog of releases:

- Per release: artifacts pinned.
- Deployment history.

Cross-link to [cicd-and-eval-gates/](../cicd-and-eval-gates/) *(coming)*.

### 5.8 The "we don't pin datasets" failure

Without dataset pinning:

- Reproducibility lost.
- Eval drift undetected.
- Lineage broken.

Pin from day one.

---

## 6. Dataset-as-release-artifact

Treating data as a release artifact.

### 6.1 The release-artifact concept

Each release is a bundle of artifacts:

- Code (binary / image).
- Model (weights or API endpoint pin).
- Prompt (versioned).
- Dataset (versioned).

All are artifacts; all are released together.

### 6.2 The release pipeline

```
PR / commit →
  Build code →
    Run tests →
      If pass → tag release →
        Pin model version →
          Pin prompt version →
            Pin dataset version(s) →
              Deploy →
                Eval gate (eval suite at this dataset version against this model + prompt)
```

Pipeline integrates datasets.

### 6.3 The dataset eval-gate

Per release:

- Run eval suite (using pinned eval golden set).
- Compare to threshold.
- Pass → deploy; fail → reject.

Datasets as part of CI/CD.

### 6.4 The "dataset version is required" check

In CI:

- Dataset version must be specified in release manifest.
- Missing → release fails.

Forces discipline.

### 6.5 The dataset-pin in production

In production:

- Each deployed instance knows which dataset versions it depends on.
- Pinned in configuration.

For retrieval: queries hit the pinned corpus version.
For eval: uses the pinned golden set.

### 6.6 The "dataset version mismatch" detection

If production starts using a different dataset version than was tested:

- Alert.
- Investigation.
- Reconcile.

Drift detection.

### 6.7 The compliance audit support

Per compliance audit:

- "What model + prompt + data was used for this output?"
- Answer: look up the release that produced it.

Lineage answered.

### 6.8 The release storage

Releases stored:

- Artifacts in respective stores (model registry, prompt store, dataset store).
- Manifest in central release log.

Retained per retention policy.

---

## 7. Retention policy

How long to keep old versions.

### 7.1 The retention drivers

- **Reproducibility.** Need old versions for re-running.
- **Compliance.** Some regulations require N-year retention.
- **Audit.** Historical investigations need old versions.
- **Storage cost.** Old versions cost money.

Balance.

### 7.2 The per-dataset retention

```yaml
care-coordinator-eval-golden-set:
  retention: indefinite (small; compliance-sensitive)

care-coordinator-fine-tune-data:
  retention: 7 years (compliance)

clinical-corpus:
  retention: 1 year (latest sufficient; daily updates)

few-shot-examples:
  retention: indefinite (small)

document-classification-fine-tune-data:
  retention: 3 years (after model deprecated)
```

Per dataset.

### 7.3 The compliance considerations

For some industries:

- 7-year retention required for some data classes.
- Audit-trail retention.

Cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / data-residency-patterns.md §7](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/data-residency-patterns.md).

### 7.4 The storage tiering

For old versions:

- Hot storage (recent / active): standard.
- Warm (within retention; rarely accessed): infrequent-access.
- Cold (compliance retention): archive (Glacier, etc.).

Cost-efficient.

### 7.5 The deletion policy

Past retention:

- Automated deletion.
- Audit log of deletion.
- Catalog entry marked "deleted."

Periodic cleanup.

### 7.6 The "we accidentally deleted important data" risk

Mitigation:

- Soft-delete before hard-delete.
- 30-day recovery window.
- Approval workflow for hard-delete.

### 7.7 The retention-vs-cost trade-off

For very large datasets:

- 7 years × growing dataset = significant storage cost.
- Balance: retain critical versions; deprecate intermediates.

### 7.8 The retention-for-litigation hold

If litigation is active:

- All related data preserved indefinitely.
- Even past normal retention.

Legal-hold mechanism.

---

## 8. Worked Meridian example

Meridian's dataset versioning practice.

### 8.1 The dataset catalog

```
Datasets at Meridian (15 total):

Eval golden sets (7):
  - care-coordinator-eval-v8.2.0
  - patient-api-chat-eval-v3.1.0
  - clinical-decision-support-eval-v2.0.0
  - document-classification-eval-v5.4.0
  - analytics-warehouse-copilot-eval-v1.3.0
  - internal-copilot-eval-v0.5.0
  - billing-code-eval-v1.2.0

Fine-tune datasets (3):
  - document-classification-fine-tune-data-v12.0.0
  - billing-code-fine-tune-data-v3.0.0
  - meridian-classifier-fine-tune-data-v8.0.0

Retrieval corpora (2):
  - clinical-corpus-v23.1.0 (40k docs)
  - reference-corpus-v5.2.0 (formulary, guidelines)

Few-shot example sets (3):
  - care-coordinator-few-shot-v4.0.0 (8 examples)
  - patient-api-few-shot-v2.1.0 (5 examples)
  - clinical-decision-few-shot-v3.0.0 (10 examples)
```

Catalog maintained.

### 8.2 The tooling

DVC + Git:

```
meridian-data-repo/
  eval-golden-sets/
    care-coordinator/
      v8.2.0/
        cases.jsonl (DVC-tracked)
        manifest.yaml
        changelog.md
  fine-tune-data/
    ...
  retrieval-corpora/
    ...
```

Standard layout; familiar.

### 8.3 The release manifest

Every Care Coordinator release pins:

```yaml
care-coordinator-release-v14.2.0:
  model: anthropic:claude-sonnet:4-6
  prompt: care-coordinator-v2.3.0
  datasets:
    eval_golden_set: care-coordinator-eval-v8.2.0
    few_shot_examples: care-coordinator-few-shot-v4.0.0
    retrieval_corpus: clinical-corpus-v23.1.0
  eval_results_at_release:
    pass_rate: 96.4%
    cases_run: 252
  deployed_at: 2026-05-10
  release_owner: clinical-ai-team
```

Reproducible.

### 8.4 The Q1 2026 reproducibility incident

Customer asked: "Why did the AI say X two months ago when it now says Y?"

Investigation:

- Looked up the release at the time.
- Pulled the model + prompt + dataset versions from manifest.
- Re-ran in pre-production.
- Confirmed: output Y was correct; the customer's recollection was wrong.

Without dataset versioning: investigation would have been "we think these versions but can't be sure."

### 8.5 The Q2 2026 fine-tune-data audit

For a compliance audit:

- "What data was used to train the document-classification model in production now?"
- Answer: document-classification-fine-tune-data-v12.0.0, with hash and creation date.
- Compliance team verified the training data was within approved scope.

Auditable.

### 8.6 The retention practice

- Active golden sets: hot storage; ~$50/month.
- Old eval versions (>1 year): warm storage; ~$10/month.
- Fine-tune data (7-year retention): archive after 1 year; ~$5/month.
- Retrieval corpus snapshots: weekly cold archive; ~$30/month.

Total storage: ~$95/month for all dataset versioning.

### 8.7 The CI integration

Every release PR:

- Check: dataset versions specified?
- Check: dataset versions exist in DVC?
- Eval gate: runs against pinned dataset; pass threshold.

CI rejects releases that miss the pinning.

### 8.8 The cross-team dataset sharing

The clinical-corpus is used by:

- Care Coordinator.
- Clinical decision support.
- Patient API chat.

All three pin the same corpus version per release; consistent across features.

### 8.9 The dataset-changelog

Per dataset, a changelog:

```markdown
# care-coordinator-eval-golden-set Changelog

## v8.2.0 - 2026-05-15
- Added 12 cases for new clinical scenarios.
- Rubric refined for clinical-recommendation cases.
- Backwards-compatible.

## v8.1.0 - 2026-04-01
- Added 8 cases for elderly patient context.

## v8.0.0 - 2026-03-15 [BREAKING]
- Rubric changed from numeric (1-5) to categorical (pass/fail/needs-review).
- Migration: see migration guide.
```

Visible to consumers.

### 8.10 The infrastructure cost

- DVC + S3 storage: ~$100-200/month.
- Engineering: 2 weeks initial setup + ongoing 5% of platform team.

Total: modest; high value.

### 8.11 The lessons

- Dataset versioning is non-optional for production AI.
- DVC works well for the scale.
- Release manifest is the load-bearing artifact.
- Compliance audits are dramatically easier with versioning.

---

## 9. Anti-patterns

### 9.1 The unversioned dataset

**Pattern.** Datasets evolve; nobody tracks. Drift; reproducibility lost.

**Corrective.** Version per §2.

### 9.2 The "we pin model + prompt but not data" gap

**Pattern.** Releases pin code, model, prompt — but not data. Data drifts unmonitored.

**Corrective.** Pin all per §5.1.

### 9.3 The model trained on "the dataset"

**Pattern.** "We trained on the dataset"; no version specified.

**Corrective.** Specific version pinned per §5.1.

### 9.4 The unversioned eval golden set

**Pattern.** Eval cases added / removed without versioning. Eval results vary unpredictably.

**Corrective.** Per §2.1.

### 9.5 The "we don't have storage for old versions" complaint

**Pattern.** Old versions deleted to save storage.

**Corrective.** Tiered storage per §7.4; storage cost manageable for AI datasets.

### 9.6 The "DVC is overkill" simplification

**Pattern.** Custom S3-based versioning; reinvents wheel.

**Corrective.** Use established tool per §4.6.

### 9.7 The dataset-deletion-without-recovery

**Pattern.** Hard-delete dataset; oops, need it back.

**Corrective.** Soft-delete + recovery window per §7.6.

### 9.8 The cross-team dataset duplication

**Pattern.** Each team maintains "their copy" of clinical corpus; multiple versions.

**Corrective.** Shared dataset; multiple releases pin same version per §5.6.

### 9.9 The "we have lineage in spreadsheets" gap

**Pattern.** Versioning info in spreadsheets / wikis. Not integrated; drifts.

**Corrective.** Tooling per §4.

### 9.10 The "the release manifest is in someone's head"

**Pattern.** What was deployed where: knowledge in engineering's head.

**Corrective.** Explicit manifest per §5.1.

---

## 10. Findings (sprint-assignable)

### DATA-DV-001 — Severity: Critical
**Finding.** Datasets not versioned.
**Recommendation.** Version per §2; tooling per §4.
**Owner.** AI platform + data engineering, sprint N+1.

### DATA-DV-002 — Severity: Critical
**Finding.** Releases don't pin dataset versions.
**Recommendation.** Release manifest per §5.1.
**Owner.** AI platform + engineering management, sprint N+1.

### DATA-DV-003 — Severity: Critical
**Finding.** Eval golden sets not versioned.
**Recommendation.** Per §2.1 and §3.1.
**Owner.** AI platform + eval, sprint N+1.

### DATA-DV-004 — Severity: High
**Finding.** Fine-tune data versioning absent.
**Recommendation.** Per §2.2.
**Owner.** AI platform, sprint N+2.

### DATA-DV-005 — Severity: High
**Finding.** Retrieval corpus not versioned.
**Recommendation.** Timestamp-based per §3.2.
**Owner.** AI platform + data engineering, sprint N+2.

### DATA-DV-006 — Severity: High
**Finding.** Tooling for dataset versioning absent.
**Recommendation.** Choose per §4.6.
**Owner.** AI platform + data engineering, sprint N+2.

### DATA-DV-007 — Severity: High
**Finding.** Retention policy absent.
**Recommendation.** Per §7.2.
**Owner.** AI platform + compliance, sprint N+2.

### DATA-DV-008 — Severity: High
**Finding.** Release manifest doesn't include datasets.
**Recommendation.** Per §5.1.
**Owner.** AI platform, sprint N+2.

### DATA-DV-009 — Severity: Medium
**Finding.** Dataset changelog absent.
**Recommendation.** Per §8.9; changelog per dataset.
**Owner.** AI platform + dataset owners, sprint N+3.

### DATA-DV-010 — Severity: Medium
**Finding.** Dataset-as-release-artifact integration absent.
**Recommendation.** Per §6.
**Owner.** AI platform, sprint N+3.

### DATA-DV-011 — Severity: Medium
**Finding.** Cross-team dataset sharing not coordinated.
**Recommendation.** Per §5.6.
**Owner.** AI platform + data engineering, sprint N+3.

### DATA-DV-012 — Severity: Medium
**Finding.** Old versions on hot storage; cost.
**Recommendation.** Tiered storage per §7.4.
**Owner.** data engineering + FinOps, sprint N+3.

### DATA-DV-013 — Severity: Medium
**Finding.** Soft-delete + recovery absent.
**Recommendation.** Per §7.6.
**Owner.** data engineering, sprint N+4.

### DATA-DV-014 — Severity: Medium
**Finding.** Compliance retention not aligned with retention policy.
**Recommendation.** Per §7.3.
**Owner.** compliance + data engineering, sprint N+4.

### DATA-DV-015 — Severity: Low
**Finding.** Dataset-version mismatch alerting absent.
**Recommendation.** Per §6.6.
**Owner.** observability + AI platform, sprint N+5.

### DATA-DV-016 — Severity: Low
**Finding.** Cross-tool integration absent (DVC + model registry + prompt store).
**Recommendation.** Manifest-level integration per §5.7.
**Owner.** AI platform, sprint N+5.

### DATA-DV-017 — Severity: Low
**Finding.** Legal hold mechanism absent.
**Recommendation.** Per §7.8.
**Owner.** legal + data engineering, sprint N+6.

### DATA-DV-018 — Severity: Low
**Finding.** Storage cost not tracked.
**Recommendation.** Per §8.10.
**Owner.** FinOps + data engineering, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Audit existing datasets per §2.**
- [ ] **Choose tooling per §4.6.**
- [ ] **Tag existing datasets as v1.0.0.**
- [ ] **Build release manifest per §5.1.**
- [ ] **CI integration per §6.4.**
- [ ] **Retention policy per §7.2.**
- [ ] **Tiered storage per §7.4.**
- [ ] **Soft-delete + recovery per §7.6.**
- [ ] **Cross-team dataset coordination per §5.6.**
- [ ] **Dataset changelog per dataset.**
- [ ] **Annual retention audit.**

---

## 12. References

**In this folder.**
- [labeling-and-annotation.md](./labeling-and-annotation.md) — labeled data versioning (companion).
- [data-quality-for-ai.md](./data-quality-for-ai.md) — quality of versioned data (companion).
- [retrieval-corpus-engineering.md](./retrieval-corpus-engineering.md) — corpus versioning (companion).
- [synthetic-data-generation.md](./synthetic-data-generation.md) *(coming)* — synthetic data versioning.
- [eval-data-contamination-prevention.md](./eval-data-contamination-prevention.md) *(coming)* — versioning supports contamination prevention.
- [training-eval-split-discipline.md](./training-eval-split-discipline.md) *(coming)* — splits are versioned.
- [data-contracts-for-ai.md](./data-contracts-for-ai.md) *(coming)* — contracts version with data.

**Elsewhere in this repo.**
- [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — eval data design.
- [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — model versioning.
- [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) — prompt versioning.
- [cicd-and-eval-gates/](../cicd-and-eval-gates/) — release pipeline.

**Sibling repos.**
- [ai-architecture-reference-architecture / data-architecture-for-ai / data-contracts-for-retrieval.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/data-contracts-for-retrieval.md) — data contracts architecture.
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / data-residency-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/data-residency-patterns.md) — residency for datasets.

**External.**
- DVC documentation.
- LakeFS documentation.
- Pachyderm documentation.
- HuggingFace Datasets documentation.
- MLflow datasets documentation.
- Weights & Biases artifacts documentation.
