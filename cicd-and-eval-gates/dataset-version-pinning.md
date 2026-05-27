# Dataset Version Pinning

> **Audience.** Engineers whose AI system's behavior depends on datasets (retrieval corpora, fine-tune data, eval golden sets) and who cannot currently answer "which version of the corpus was running when the incident happened?" Platform leads building the artifact-pinning discipline already in place for code, prompts, and models out to include data. **Scope.** The *engineering* pattern for pinning dataset versions in release artifacts and enforcing the pin at deploy time: which datasets to pin, the content-hash discipline, the manifest entries, the deploy gate, the reproducibility guarantee. Pair with [prompt-version-pinning.md](./prompt-version-pinning.md) (the prompt-side companion) and [model-version-pinning.md](./model-version-pinning.md) (the model-side companion). Cross-link to [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md) (the upstream versioning discipline this pin enforces) and [data-engineering-for-ai/retrieval-corpus-engineering.md](../data-engineering-for-ai/retrieval-corpus-engineering.md) (the corpus this pin pins). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Code, prompts, and models are the three artifacts most teams instinctively recognize as "things that should be pinned in a release." The fourth — datasets — is often forgotten. The consequence: when an AI system's behavior depends on a retrieval corpus, a fine-tune dataset, or an eval golden set, and that data changes between releases without anyone tracking it, the team has lost a reproducibility property they did not realize they had.

The Care Coordinator clinical-knowledge model's RAG pipeline retrieves from a corpus of curated clinical reference documents. The corpus is updated weekly as new guidelines are published and old ones are revised. A user asks "what is the recommended dosing for atorvastatin in patients with hepatic impairment?" The model retrieves three documents and synthesizes an answer. Two weeks later, the same user asks the same question and gets a different answer — because the corpus was updated and the retrieval surfaced different documents. Both answers may be correct for their time, but the team has lost the ability to *re-derive* the older answer. Audit, debugging, regression analysis, and incident response all suffer.

The fix is to pin datasets the same way the team pins prompts and models: the release manifest captures the dataset version (or content hash), the deployment is bound to that version, and the system can be reproduced from the manifest alone. This document is the pattern.

The discipline is also where eval credibility lives. An eval suite's golden set is itself a dataset. If the golden set changes between eval runs, the runs are not comparable. If the golden set is pinned per release, the runs are. The pinning discipline is what makes eval results an audit-grade artifact rather than a snapshot of a moving target.

The honest framing: dataset pinning is the least-glamorous of the four pinning patterns and the most commonly skipped. The cost of skipping it is invisible until the audit, the regression analysis, or the incident — and then the cost is large because the team cannot reproduce what was running.

This document is opinionated about four things:

1. **Every behavior-affecting dataset is pinned in the release manifest.** Retrieval corpora, fine-tune datasets, eval golden sets, any reference data the model reads at runtime.
2. **The pin is to a content-addressed version, not a label.** A hash, a git commit, a versioned object store key — something that cannot silently change after the pin.
3. **The deploy gate enforces the pin.** A release that pins a non-existent dataset version is refused; a runtime that reads from an unpinned source is refused.
4. **Datasets at runtime are immutable from the running release's perspective.** Edits to a dataset go into a *new* version; the running release continues to read its pinned version.

Structure: (2) which datasets need pinning; (3) what "pin" means for a dataset; (4) the release manifest entry; (5) the deploy gate; (6) the build-time vs runtime-fetch trade-off; (7) integration with data-engineering versioning; (8) reproducibility and audit; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. Which datasets need pinning

Not every dataset in the system needs pinning. The rule: pin datasets whose contents *affect production behavior* or *attest to production behavior*.

### 2.1 Datasets that need pinning

**Retrieval corpora.** The documents the RAG pipeline retrieves from. The corpus contents directly affect what the model sees and therefore what it answers. Pin per release.

**Fine-tune datasets.** The training data used to produce a fine-tuned model. The dataset version is what determines what behaviors the fine-tune learned. Pin both at fine-tune-creation time and in any release that uses the fine-tune.

**Eval golden sets.** The cases the eval suite scores against. Pin per eval-suite release; pin per CI/CD release that depends on a specific eval-pass result.

**Few-shot example libraries.** If the system pulls few-shot examples from a curated library at runtime, the library is a behavior-affecting dataset. Pin.

**Tool / function manifests.** If the system loads its available tool list from a dataset (rather than from code), the tool list is behavior-affecting. Pin.

**Reference data the model includes in prompts at runtime.** Drug interaction tables, dosing guidelines, formulary data — anything the prompt-assembly layer reads and inserts into the prompt. Pin.

### 2.2 Datasets that do not need pinning

**Conversation history.** The user's own chat history is stateful but per-conversation, not a system-level dataset. Pin the schema, not the contents.

**Telemetry data.** Logs, traces, metrics. These are *outputs* of the system, not inputs. Pin the schema; the contents are write-only and grow continuously.

**Training data for models not in the release.** A historical fine-tune dataset that was used to produce a model now deprecated; the dataset only matters if the deprecated model is in the release.

**User-provided data the system processes one-shot.** A document the user uploads for the system to summarize is data, but it is per-request input, not a pinnable system dataset.

### 2.3 The "behavior-affecting" test

If you ask "would replaying yesterday's traffic against today's system reproduce yesterday's outputs?" and the answer is "no, because the dataset changed" — the dataset needs pinning.

If the answer is "yes, but the user's input was different" — the dataset is not the variable; it's the input.

If the answer is "no, because telemetry data has accumulated" — telemetry is output, not behavior-affecting input.

---

## 3. What "pin" means for a dataset

The pin is a release-time commitment to a specific dataset state.

### 3.1 The reference forms

**Content hash.** A SHA256 (or similar) of the dataset's canonical serialization. For a small dataset (few hundred KB), the hash is easy. For a large dataset (GB-scale corpus with thousands of documents), the hash is of a manifest of the dataset (each file's hash) rather than the raw data.

**Versioned object store key.** S3 object key like `corpus/v2026.05.25/manifest.json` where the path is content-addressed and immutable.

**Git commit.** For datasets stored in git (small text corpora, eval cases as YAML/JSON), the git commit hash is the pin.

**Tool-native version.** DVC version, Pachyderm commit, LakeFS branch — if the team uses a dataset-versioning tool, its native version is the pin.

### 3.2 What the pin records

In the release manifest:

```yaml
datasets:
  care_coordinator_clinical_corpus:
    version: 2026.05.20
    type: retrieval_corpus
    storage: s3://meridian-corpora/clinical/v2026.05.20/
    manifest_sha256: 7f9a3b2c8d1e4f5a...
    document_count: 14237
    last_updated: 2026-05-20T08:14:22Z

  care_coordinator_few_shot_library:
    version: 1.4.2
    type: few_shot_examples
    storage: git://prompts-repo@a3c4f8b
    manifest_sha256: 3c8e2a1f4b9d8e7a...

  drug_interaction_table:
    version: 2026.05
    type: reference_data
    storage: s3://meridian-reference-data/drug-interactions/2026.05.json
    sha256: 9d2a8c4f1b7e3a8b...

  care_coordinator_golden_set:
    version: 4.2.0
    type: eval_golden_set
    storage: git://eval-repo@b4d7a2c
    case_count: 1247
    manifest_sha256: 1b8f7e3a2c9d4e5f...
```

### 3.3 What the pin guarantees

- **Reproducibility.** Given the manifest, the exact dataset state can be retrieved.
- **Immutability.** The pinned version cannot be edited after the pin (or rather, edits produce a new version, leaving the pinned version intact).
- **Auditability.** A regulator asking "what was the clinical reference data when this decision was made" gets a deterministic answer.
- **Rollback fidelity.** A code rollback to release N restores not just code and prompts but the datasets release N was reading.

### 3.4 What the pin does NOT guarantee

- **Quality of the data.** Pinning a bad dataset still ships a bad dataset. Data-quality checks ([data-engineering-for-ai/data-quality-for-ai.md](../data-engineering-for-ai/data-quality-for-ai.md)) are upstream of pinning.
- **Freshness.** A pinned dataset is by definition not the latest; pin updates are intentional release events.
- **Behavior across model versions.** A pinned dataset paired with a different model version may behave differently. The model pin is the companion.

---

## 4. The release manifest entry

The dataset entries sit alongside the prompt and model entries in the release manifest ([release-artifacts-for-ai.md](./release-artifacts-for-ai.md)).

### 4.1 Entry shape

Each pinned dataset includes:

- **Logical name.** What the system references it by. Stable across versions.
- **Version.** Semantic or date-based version label, human-readable.
- **Type.** Category (retrieval_corpus / few_shot_examples / fine_tune_dataset / eval_golden_set / reference_data).
- **Storage location.** The canonical URI from which the dataset can be retrieved.
- **Content hash.** A cryptographic hash of the dataset's manifest or canonical serialization.
- **Cardinality metadata.** Document count, case count, row count — whatever is meaningful for the dataset type.
- **Last-updated timestamp.** When this version was created.
- **Schema version.** If the dataset has a schema (eval cases conform to a Pydantic model, for example), pin the schema version.

### 4.2 Storage-system specifics

**S3 / object store.** The URI is the immutable path. The bucket policy enforces immutability on versioned keys (no overwrites at a versioned key).

**Git.** The pin is the commit SHA. The dataset is materialized by `git checkout`.

**Versioned dataset tool (DVC, Pachyderm, LakeFS).** The pin is the tool's commit / version identifier.

**Custom store.** The pin is whatever the store's immutable-version identifier is. The store must support immutability; if it does not, the dataset is not pinnable.

### 4.3 Per-tenant datasets

For multi-tenant systems where different tenants get different datasets (per-tenant retrieval corpus, per-tenant fine-tune data):

```yaml
datasets:
  care_coordinator_clinical_corpus:
    default:
      version: 2026.05.20
      storage: s3://meridian-corpora/clinical/v2026.05.20/
      manifest_sha256: 7f9a3b...
    tenants:
      hospital_a:
        version: 2026.05.20-hospital_a
        storage: s3://meridian-corpora/clinical-tenant-a/v2026.05.20/
        manifest_sha256: 4d2c8b...
      hospital_b:
        version: 2026.05.20-hospital_b
        storage: s3://meridian-corpora/clinical-tenant-b/v2026.05.20/
        manifest_sha256: 8e1a9f...
```

The deploy gate verifies all tenant variants resolve; the runtime routing layer selects per-tenant.

### 4.4 The "snapshot" pattern for derived datasets

Some datasets are derived from other systems (a daily snapshot of an external drug-interaction database, for example). The derived dataset is pinned by snapshot date:

```yaml
drug_interaction_table:
  version: 2026.05.20
  type: reference_data
  source: external-pharm-db
  snapshot_date: 2026-05-20
  storage: s3://meridian-reference-data/drug-interactions/2026.05.20.json
  sha256: 9d2a8c4f1b7e3a8b...
```

The snapshot is immutable; new snapshots produce new versions; the pin references a specific snapshot.

---

## 5. The deploy gate

The deploy gate enforces dataset pins the same way it enforces prompt and model pins.

### 5.1 What the deploy gate checks

Before a release deploys:

1. **Every pinned dataset resolves at the specified URI.** Fetch the manifest; verify it exists.
2. **The fetched manifest's hash matches the pinned hash.** Tamper-evidence.
3. **The deploying code requests no datasets that are not pinned.** Static analysis or runtime-config-check of the code's dataset-load calls.
4. **No deprecated datasets are pinned in new releases.** Deprecation status is in the dataset registry; pinning a deprecated dataset fails the gate.
5. **Schema compatibility.** If the dataset has a schema and the code expects a specific schema version, the pinned dataset's schema matches.

### 5.2 The failure modes

- "Dataset `care_coordinator_clinical_corpus@2026.05.20@sha:7f9a...` not found at the specified URI."
- "Dataset manifest hash mismatch: expected `7f9a...`, found `8a2b...`. Dataset has been modified since pin."
- "Code requests dataset `care_coordinator_intent_examples` but manifest has no pin."
- "Pinned dataset `legacy_eval_set@1.0.0` is deprecated and cannot ship in new releases."

### 5.3 Bypass discipline

Same as prompt and model pinning bypass:

- Documented justification.
- Senior approval (SRE on-call + data-engineering lead).
- Logged in the release artifact.
- Expiration.

Bypasses are for emergencies. Not for convenience.

### 5.4 Bypass that is sometimes legitimate

For an emergency hotfix where the dataset is the *fix* (e.g., a corrupted document is removed from the corpus, and the team wants the new corpus version live now), the emergency-hotfix path applies:

- The new dataset version is published to the immutable store.
- The release manifest is updated to pin the new version.
- The standard pipeline runs in abbreviated form.

This is *not* a bypass of pinning; it is a fast pinning update.

---

## 6. The build-time vs runtime-fetch trade-off

Datasets are larger than prompts. The question is whether to bundle them into the release artifact (build-time) or fetch them at runtime from the pinned location.

### 6.1 Build-time bundling

Pros:

- Reliability. A pinned-dataset-store outage does not affect a running deployment.
- Performance. No network fetch at boot or hot-path.
- Reproducibility. The release artifact is fully self-contained.

Cons:

- Image / artifact size grows. A 5 GB corpus bundled into a container image is a heavy image.
- Deploy time grows. Pulling a multi-GB image to every instance takes longer.
- Wasted storage. If 10 deployment regions all pull the same 5 GB image, that's 50 GB of stored data for one dataset.

### 6.2 Runtime fetch

Pros:

- Smaller release artifacts.
- Faster deploys.
- The dataset is fetched once per deployment region, cached locally.

Cons:

- Pinned-store availability becomes a runtime dependency.
- A boot-time fetch failure means the instance fails to start.
- The runtime must verify the fetched dataset against the pinned hash (otherwise the pin is decoration).

### 6.3 The hybrid pattern

For most teams, the right answer is hybrid:

- **Small datasets (< 100 MB):** bundle into the release. Few-shot libraries, reference tables, small eval sets.
- **Large datasets (≥ 100 MB):** fetch at boot, verify hash, cache locally.

The runtime's dataset-loader:

```python
class DatasetLoader:
    def __init__(self, manifest: ReleaseManifest, cache_dir: str):
        self.manifest = manifest
        self.cache_dir = cache_dir

    def load(self, dataset_name: str) -> Dataset:
        entry = self.manifest.datasets[dataset_name]
        cache_path = f"{self.cache_dir}/{dataset_name}/{entry.version}/"
        if not self._cache_valid(cache_path, entry.sha256):
            self._fetch_and_verify(entry, cache_path)
        return Dataset.load(cache_path)

    def _cache_valid(self, path: str, expected_sha: str) -> bool:
        # Check cache exists and its hash matches the pinned hash.

    def _fetch_and_verify(self, entry: DatasetEntry, dest: str):
        # Fetch from entry.storage; compute hash; raise if mismatch.
```

### 6.4 Cache discipline

The local cache is:

- **Read-only after fetch.** The runtime does not write to the cache except during fetch.
- **Per-version-keyed.** Different versions live in different paths; old versions are not overwritten.
- **Pruned periodically.** Versions not referenced by any live release for > 30 days are eligible for pruning.

---

## 7. Integration with data-engineering versioning

Pinning works only if the upstream dataset-versioning discipline is sound.

### 7.1 The dependency on dataset-versioning.md

[data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md) defines:

- The version-numbering convention (semantic or date-based).
- What counts as a new version (any content change; schema changes are major).
- The immutability discipline (versions never change after publication).
- The deprecation policy.

Without this discipline, "pinning" is just hash-pinning with no semantic meaning.

### 7.2 What this layer adds

This document adds the *release-time integration*: how versioned datasets get into the release, get verified, get enforced at deploy.

### 7.3 The dataset registry

A dataset registry catalogs every behavior-affecting dataset:

- Name, type, current latest version, all historical versions.
- Deprecation status.
- Owner (the team responsible).
- Schema (if any).

The deploy gate reads from the registry to verify deprecation status.

### 7.4 The two-PR pattern for dataset changes

For behavior-affecting datasets:

- PR 1: Publish a new dataset version (to the immutable store, register it).
- PR 2: Update the release manifest to pin the new version.

The two-PR pattern ensures the dataset is published and verifiable *before* a release pins it. The release pipeline runs full eval against the new dataset version.

---

## 8. Reproducibility and audit

The reproducibility property is what dataset pinning earns.

### 8.1 The full reproduction

Given a release-id, the team can reconstruct exactly what was running:

- Code at the pinned commit.
- Prompts at the pinned hashes.
- Models at the pinned versions.
- Datasets at the pinned versions.
- Eval results from the pinned eval suite against the pinned everything-else.

This is the audit-grade reproducibility. Compliance, post-incident review, and regression analysis all become tractable.

### 8.2 The audit query

"What clinical reference data was the Care Coordinator using on 2026-05-22 at 14:30 UTC for tenant H?"

The answer chain:

1. Deployment log: release `2026.05.25-r3` was live at that time for tenant H's region.
2. Release manifest for `2026.05.25-r3`: pins `care_coordinator_clinical_corpus` for tenant H to `2026.05.20-hospital_h@sha:8e1a9f...`.
3. The pinned URI returns the manifest at that hash.
4. The manifest enumerates the documents in the corpus.
5. The compliance reviewer has the answer with cryptographic guarantees.

### 8.3 The regression analysis

A user reports: "The Care Coordinator gave me a wrong answer last week."

The investigation:

1. From the user's trace, find the timestamp and release-id.
2. From the release manifest, find the pinned dataset versions.
3. Reproduce the retrieval call against the pinned corpus.
4. Reproduce the prompt assembly against the pinned prompt.
5. Reproduce the model call against the pinned model.
6. Compare the reproduced output to the user's reported output.

Without pinning, step 3 is impossible: the corpus has moved on.

### 8.4 The eval credibility

The eval suite produces a pass-rate against the pinned golden set. If the golden set is pinned, the pass-rate is comparable across releases. If the golden set drifts, the pass-rate is not.

The eval-result section of the release artifact records:

- The eval-suite version.
- The eval golden-set version.
- The pass-rate.

Three months later, the team can re-run the same eval against the same dataset and compare against the recorded pass-rate. The eval becomes a stable reference, not a moving target.

---

## 9. Worked Meridian example: pinning the clinical reference corpus

The Care Coordinator's RAG pipeline retrieves from a clinical reference corpus. The corpus has been updated weekly for months without explicit pinning; the team has no way to reproduce older retrievals.

### 9.1 The starting state

- Corpus stored in S3 at `s3://meridian-corpora/clinical/current/` (a *mutable* path that always points to "the latest").
- Updates done by the clinical content team via a web tool that overwrites files in-place.
- Application code reads from the `current/` path at runtime.

Problem: an audit asks "what was the corpus on 2026-04-15?" — unanswerable.

### 9.2 The refactor

Step 1: introduce versioned paths.

- `s3://meridian-corpora/clinical/v2026.05.20/` — immutable.
- A manifest at `s3://meridian-corpora/clinical/v2026.05.20/manifest.json` lists every document and its individual hash.
- The `current/` path is removed; nothing reads from it.

Step 2: introduce the dataset registry.

- Register `care_coordinator_clinical_corpus` with type `retrieval_corpus`.
- Record each historical version (where possible to reconstruct from S3 versioning).
- Document the schema (document format: title, body, source, retrieval-tags, last-reviewed-by).

Step 3: update the release manifest.

- Add `datasets:` section with the pin.
- The deploy gate now enforces it.

Step 4: update the application.

- Application reads `manifest.datasets.care_coordinator_clinical_corpus.storage` and loads from the pinned path.
- A hash-verification check at startup confirms the loaded corpus matches the pinned hash.

Step 5: the content workflow.

- Content team updates via a Pull Request flow against a content repo.
- Approved changes are published as a new version (next date-based label).
- A release-prep PR updates the manifest to pin the new version.
- The standard pipeline runs.

### 9.3 First production release

- Release `2026.05.25-r3` is the first release with corpus pinning.
- Manifest: `care_coordinator_clinical_corpus@2026.05.20@sha:7f9a3b...`.
- Application reads from the pinned path; verifies hash at startup; loads.
- Full eval runs against the pinned corpus; pass rate is now stable across re-runs of the same release.

### 9.4 The audit win

Two months later, a compliance review asks: "Show me the clinical references for the dosing answer the system gave on 2026-05-28."

- Deployment log: release `2026.05.25-r3` live then.
- Manifest: `care_coordinator_clinical_corpus@2026.05.20@sha:7f9a3b...`.
- S3: `s3://meridian-corpora/clinical/v2026.05.20/manifest.json` returns the document list.
- For the specific user's trace: the trace records which documents were retrieved (per [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md)).
- The reviewer can read the exact documents the model saw.

### 9.5 The corpus-update incident retro

A content reviewer notes that one document in the corpus had an error that survived multiple updates. The team can now:

- Identify exactly which releases shipped with the bad document (via the manifest's pinned corpus versions).
- Estimate impact (volume of queries that retrieved that document).
- Publish a corrected corpus version.
- Open a release-prep PR to pin the new version.
- Document the fix in the release notes.

### 9.6 Findings closed

- **ARCH-CARE-074** (corpus pulled from a mutable `current/` path; older retrievals not reproducible).
- **ARCH-CARE-075** (corpus updates went through admin UI; no PR record, no version trail).
- **ARCH-CARE-076** (no dataset registry; "what datasets affect production" was tribal knowledge).
- **ARCH-CARE-077** (eval pass-rates not comparable across runs because golden set drifted).
- **ARCH-CARE-078** (compliance audit could not reproduce historical retrievals; the audit was a flag for the company).

---

## 10. Anti-patterns

### 10.1 The mutable "current" path

The corpus lives at a path like `s3://.../current/` that is overwritten on every update. The pin records the path but the path's contents change. The pin is a name, not a version.

The fix: immutable versioned paths. The `current/` symlink-style pattern is removed from production reads.

### 10.2 The hash-of-nothing

The manifest records a "version" but no hash. The version is a label; the underlying data can change silently. The team thinks they have pinning; they don't.

The fix: hash everything. The hash is the cryptographic guarantee; the version is the human-readable convenience.

### 10.3 The eval-golden-set drift

The team runs evals against a golden set that is continuously edited. Pass rates rise mysteriously (cases are silently removed) or fall mysteriously (harder cases are added). Trend analysis is impossible.

The fix: golden sets are pinned per eval-suite version. Edits produce a new eval-suite version; the prior version remains comparable.

### 10.4 The per-tenant data unpinned

Multi-tenant systems load per-tenant data at runtime from tenant-specific paths. The default corpus is pinned; the per-tenant ones are not. Tenant-specific regressions slip through.

The fix: every per-tenant dataset is pinned per-tenant. The manifest carries the tenants section per dataset.

### 10.5 The forgotten fine-tune dataset

A fine-tuned model was trained on a dataset that was assembled ad-hoc and never preserved. The team wants to re-fine-tune with one more example; they cannot re-create the original dataset; they cannot meaningfully extend the training.

The fix: fine-tune datasets are versioned and pinned. The exact training set is preserved alongside the model artifact.

### 10.6 The runtime fetch with no hash verify

The application fetches a pinned dataset at runtime but does not verify the fetched contents against the pin's hash. If someone modifies the pinned-version path (in violation of immutability), the runtime cheerfully loads the modified data.

The fix: hash verify at fetch. A mismatch is a deploy-time / boot-time failure.

### 10.7 The "we'll worry about pinning when audit asks"

The team postpones dataset pinning until the first audit. The audit happens; the team cannot reproduce historical state; the audit produces findings; the team scrambles.

The fix: pin from the start. Audit-readiness is a property of the pinning discipline, not a project to start when audit shows up.

### 10.8 The deprecation-immune pin

A deprecated dataset can still be pinned in new releases because the deploy gate does not check deprecation status. The deprecation policy is convention only.

The fix: deploy gate reads the dataset registry's deprecation status and refuses deprecated datasets in new releases.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-DVP-001 | Behavior-affecting datasets not pinned in release manifest | High | Pin per §2.1; manifest carries datasets section | AI Platform + Data Eng |
| CICD-DVP-002 | Datasets stored at mutable paths (`current/`, `latest`) | High | Move to immutable versioned paths; remove mutable read paths from production | Data Eng + AI Platform |
| CICD-DVP-003 | Pins record version label but no content hash | High | Hash everything; manifest captures both version and hash per §3.1 | AI Platform |
| CICD-DVP-004 | Deploy gate does not verify dataset pins | High | Implement gate checks per §5.1; failure blocks deploy | AI Platform + SRE |
| CICD-DVP-005 | Runtime fetch from pinned store does not verify hash | High | Hash-verify at fetch per §6.3; mismatch fails startup | AI Platform |
| CICD-DVP-006 | Eval golden set drifts across runs; pass-rates not comparable | High | Pin golden set per eval-suite version; edits produce new versions | Eval Eng + Data Eng |
| CICD-DVP-007 | Fine-tune datasets not preserved; cannot reproduce training | High | Version and preserve every fine-tune dataset; pin to the model artifact | ML Eng + Data Eng |
| CICD-DVP-008 | Per-tenant datasets unpinned; per-tenant regressions slip through | Medium | Per-tenant pins in manifest per §4.3 | AI Platform + Architecture |
| CICD-DVP-009 | Dataset registry absent; "what datasets affect production" untracked | High | Build dataset registry with name, type, version, owner, deprecation status | Data Eng + AI Platform |
| CICD-DVP-010 | Reference data updates via admin UI; no PR or version trail | High | Move content updates to PR-based flow; publish as new immutable versions | Data Eng |
| CICD-DVP-011 | Build-time vs runtime-fetch decision undocumented; bundle sizes inconsistent | Low | Document per-dataset choice per §6.3 thresholds | AI Platform |
| CICD-DVP-012 | Deprecated datasets can be pinned in new releases | Medium | Deploy gate reads registry's deprecation status per §5.1 | AI Platform |
| CICD-DVP-013 | Rollback rolls back code only; datasets not restored | High | Full pinned-set rollback per [prompt-version-pinning.md §8.1](./prompt-version-pinning.md) | AI Platform + SRE |
| CICD-DVP-014 | Snapshot-based datasets (external data refreshes) not version-tracked | Medium | Snapshot per snapshot-date; pin per release per §4.4 | Data Eng |
| CICD-DVP-015 | No cache discipline for runtime-fetched datasets; cache corruption risk | Low | Per-version-keyed cache; hash-verify on every load; pruning policy | AI Platform |
| CICD-DVP-016 | Audit cannot reproduce historical retrievals or fine-tunes | High | Reproducibility-by-pinning per §8; verify by quarterly audit drill | AI Platform + Compliance |
| CICD-DVP-017 | Application code reads datasets outside the pinned path | Medium | Static-analysis or runtime-config-check that all reads go through pinned path | AI Platform |
| CICD-DVP-018 | Two-PR pattern (publish dataset + pin in manifest) skipped; ad-hoc updates | Medium | Enforce two-PR pattern per §7.4 for behavior-affecting datasets | Data Eng + AI Platform |

---

## 12. Adoption checklist

- [ ] Every behavior-affecting dataset identified and registered (per §2.1).
- [ ] Datasets stored at immutable versioned paths; no production reads from mutable paths.
- [ ] Each pin records: name, version, type, storage URI, content hash, cardinality metadata, last-updated timestamp.
- [ ] Per-tenant datasets pinned per-tenant in manifest.
- [ ] Release manifest includes datasets section alongside prompts, models, code.
- [ ] Deploy gate verifies: every pin resolves, hash matches, no deprecated datasets, schema compatible.
- [ ] Runtime fetch (where used) hash-verifies at boot; mismatch fails startup.
- [ ] Build-time bundling decision documented per dataset (size threshold per §6.3).
- [ ] Dataset registry maintained; deprecation status machine-readable.
- [ ] Eval golden sets pinned per eval-suite version; edits produce new versions.
- [ ] Fine-tune datasets preserved alongside model artifacts; pinned at train and inference time.
- [ ] Snapshot-based datasets (external refreshes) versioned per snapshot date.
- [ ] Full pinned-set rollback discipline; datasets restored alongside code/prompts/models.
- [ ] Cache discipline for runtime-fetched datasets: per-version-keyed, hash-verified, pruned.
- [ ] Quarterly reproducibility-audit drill: pick a past release, reproduce a query, compare output.

---

## 13. References

**Internal:**

- [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) — the pipeline that enforces the pin.
- [eval-gate-design.md](./eval-gate-design.md) — the eval that runs against the pinned golden set.
- [prompt-version-pinning.md](./prompt-version-pinning.md) — companion pin for prompts.
- [model-version-pinning.md](./model-version-pinning.md) — companion pin for models.
- [release-artifacts-for-ai.md](./release-artifacts-for-ai.md) — the artifact format containing the pinned set.
- [canary-rollouts.md](./canary-rollouts.md) — the canary mechanic for releases including new datasets.
- [shadow-traffic.md](./shadow-traffic.md) — the shadow alternative for dataset migrations.
- [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md) — upstream versioning discipline.
- [data-engineering-for-ai/retrieval-corpus-engineering.md](../data-engineering-for-ai/retrieval-corpus-engineering.md) — corpus engineering this pin pins.
- [data-engineering-for-ai/data-quality-for-ai.md](../data-engineering-for-ai/data-quality-for-ai.md) — quality checks upstream of pinning.
- [data-engineering-for-ai/data-contracts-for-ai.md](../data-engineering-for-ai/data-contracts-for-ai.md) — schema discipline.
- [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — trace shape recording dataset version per call.
- [model-lifecycle/fine-tuning-operations.md](../model-lifecycle/fine-tuning-operations.md) — fine-tune dataset lifecycle.
- [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) — full-pinned-set rollback discipline.

**Cross-repo (architecture sibling):**

- [data-architecture-for-ai/vector-store-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/vector-store-architecture.md) — vector-store-side of corpus engineering.
- [data-architecture-for-ai/lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — architecture-side framing of dataset provenance.
- [data-architecture-for-ai/freshness-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/freshness-architecture.md) — when fresh data matters versus when pinned matters.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
