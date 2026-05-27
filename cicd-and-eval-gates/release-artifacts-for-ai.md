# Release Artifacts for AI

> **Audience.** Engineers wiring the release-artifact format for an AI service. Platform leads who have asked "what's actually in a release?" and gotten different answers from different team members. Anyone who has tried to debug a production incident by reconstructing what was running and given up. **Scope.** The *engineering* design of the release artifact for an AI service: contents (code, prompts, models, datasets, eval-suite, eval-pass-results, cost-baseline, canary results, shadow results, approvals); the reproducibility guarantee; the rollback target role; the audit-trail role; retention; signing. Pair with [prompt-version-pinning.md](./prompt-version-pinning.md), [model-version-pinning.md](./model-version-pinning.md), [dataset-version-pinning.md](./dataset-version-pinning.md) (the pinning-discipline docs whose pins land in the artifact). Cross-link to [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) (the pipeline that produces the artifact). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The release artifact is the canonical, immutable, complete description of what is deployed. For traditional software, a release artifact is a container image, a binary, a tagged commit — a single unit that captures "this is what is running." For AI services, the artifact must capture *more* than the code: prompts, models, datasets, eval results, cost baseline, canary outcomes, shadow outcomes, approvals. Without the full set, the team cannot answer the questions that show up in audits, post-incident reviews, regression analyses, and rollback drills.

The Care Coordinator's release history before this discipline: a sequence of git tags and "looked good in staging, shipped Tuesday" Slack messages. After: a sequence of structured release artifacts where each release records the exact pins, eval results, canary outcomes, and approvals. The difference shows up in every retrospective: "what was running when X happened?" used to require detective work; now it requires reading one file.

The pattern in this document is the *artifact format*: what fields are in it, how it is built, how it is signed and approved, how it is retained, and how it is consumed by rollback / audit / compliance flows. It is the consolidation of the pinning patterns (prompts, models, datasets) plus the operational outcomes (eval results, canary results, shadow results) into one canonical document per release.

The honest framing: most teams ship AI features with implicit release artifacts — the deployed code, plus whatever happened to be in the prompt store at deploy time, plus whatever happened to be in the model registry, plus whatever happened to be in the data store. The implicit-ness is the problem. The explicit release artifact is the closure: one file that says, definitively and unforgeably, this is what was deployed.

This document is opinionated about four things:

1. **One release = one artifact.** Not three — the code artifact and the prompts and the models and the data are all part of one logical release.
2. **The artifact is signed and approved.** Tampered artifacts fail deploy. Approval chain is in the artifact, not in chat.
3. **Retention is long.** Six years (US healthcare audit retention) or longer per regulatory requirement. The artifact is the audit-trail anchor.
4. **The artifact is consumed by rollback, audit, and compliance.** It is not just a deploy input; it is the canonical record.

Structure: (2) the artifact's contents; (3) the artifact format; (4) building the artifact; (5) signing and approval; (6) retention and archival; (7) the rollback consumer; (8) the audit / compliance consumer; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The artifact's contents

A complete release artifact for an AI service includes all of the following.

### 2.1 Release identity

- **Release ID.** Unique, immutable, monotonic. Typically date + counter (`2026.05.25-r3`).
- **Built timestamp.** ISO-8601 UTC.
- **Build environment.** CI system identifier, runner version.
- **Built from.** Source commit, branch, PR number.

### 2.2 Code

- **Code version.** Git commit SHA.
- **Code-image hash.** Container image digest (if containerized).
- **Code-bundle hash.** For non-containerized deploys, the bundle's hash.

### 2.3 Prompts (per [prompt-version-pinning.md](./prompt-version-pinning.md))

For each prompt:

- Logical name.
- Version (semantic version or label).
- Content hash (SHA256).
- Expected variables.
- Schema version.
- Per-tenant overrides if any.

### 2.4 Models (per [model-version-pinning.md](./model-version-pinning.md))

For each model the system uses:

- Logical name (e.g., `care_coordinator_clinical_knowledge`).
- Provider + specific version pin (e.g., `claude-opus-4-7@2026-04-12`).
- Hash where applicable (for self-hosted: weight hash; for vendor-hosted: the version is the pin).
- Runtime config (for self-hosted: which serving stack, GPU, quantization).
- Per-tenant assignments if different tenants get different models.

### 2.5 Datasets (per [dataset-version-pinning.md](./dataset-version-pinning.md))

For each behavior-affecting dataset:

- Logical name.
- Version, type, storage URI, content hash, cardinality metadata.
- Per-tenant assignments if applicable.

### 2.6 Eval suite

- **Eval suite version.** The suite that ran against this release.
- **Eval suite hash.** Tamper-evident.
- **Golden set version.** The pinned cases.
- **Judge config.** Judge model, judge prompt, judge rubric version.

### 2.7 Eval results

- **Pass rate.** Overall pass rate of the eval against this release.
- **Critical-case pass rate.** 100% required.
- **Safety-case pass rate.** 100% required.
- **Cross-feature pass rate.** Required for release.
- **Cost per case.** Mean, p50, p95.
- **Latency per case.** Mean, p50, p95.
- **Per-segment results.** If stratified by query type, tenant tier, etc.
- **Failures.** Specific cases that failed, with diff and judge reasoning.
- **Comparison to baseline.** Delta from previous release's eval.

### 2.8 Cost and latency profiles

- **Estimated cost per request.** From eval profiling.
- **Estimated p50 / p95 latency.** From eval profiling.
- **Concurrency profile.** For high-traffic features.

### 2.9 Canary results (added at deploy time)

For each canary step:

- Step (1% / 10% / 50% / 100%).
- Window (start, end).
- Metrics: quality, cost, latency, reliability vs baseline.
- Decision: ramp / rollback / human-gated promotion approval.
- Approver (for human-gated steps).
- Anomalies noted.

### 2.10 Shadow results (if shadow ran)

- Shadow window (start, end).
- Aggregate metrics vs production baseline.
- Diff distribution.
- Decision (proceed to canary / discard / further investigation).

### 2.11 Approvals

- **Author.** The engineer / team that prepared the release.
- **Reviewers.** PR reviewers (clinical content, AI Platform, etc.).
- **Approvers.** Tech lead, product owner, safety lead (for safety-relevant changes), SRE lead (for high-risk deploys).
- **Approval timestamps.** When each approval was granted.
- **Approval rationale.** For releases with overrides, the rationale for the override.

### 2.12 Overrides

For releases that used eval / cost / latency / pinning overrides:

- Override target (which check was overridden).
- Justification.
- Approver.
- Expiration.

### 2.13 Signature

A cryptographic signature over the artifact contents. The signing key is held by the CI system / release authority. The signature is verified at deploy time.

### 2.14 Deploy outcome (recorded post-deploy)

- Deploy timestamp.
- Promotion timestamp (when canary reached 100%).
- Initial production metrics (first 24 hours).
- Any incidents or rollbacks within the release window.

---

## 3. The artifact format

The artifact is a structured, machine-readable file with a documented schema.

### 3.1 YAML / JSON

Either works. Pick one for the team and stick with it. YAML is more readable for humans; JSON is more universally machine-parseable. Most teams use YAML for the release manifest and accept that some tools will parse it as YAML and others as JSON-after-yaml-to-json conversion.

Example structure:

```yaml
release:
  id: 2026.05.25-r3
  built_at: 2026-05-25T14:30:00Z
  built_from:
    commit: 9c2a1f8b3d2e7a4c8f1b6e9d3a5c7b2f8e4d1a9c
    branch: main
    pr: 1247
  build_env:
    ci: github-actions
    runner_image: ubuntu-22.04-llm-runner@v3.4.1

  code:
    image_digest: sha256:7f9a3b2c8d1e4f5a8b2c9d3e7a4c8f1b6e9d3a5c7b2f
    image_uri: meridian-registry.local/care-coordinator:2026.05.25-r3

  prompts:
    # (per prompt, see §2.3 and prompt-version-pinning.md)

  models:
    # (per model, see §2.4 and model-version-pinning.md)

  datasets:
    # (per dataset, see §2.5 and dataset-version-pinning.md)

  eval:
    suite_version: 4.2.0
    suite_sha256: 1b8f7e3a2c9d4e5f6a7b8c9d0e1f2a3b4c5d6e7f
    golden_set_version: 4.2.0
    judge:
      model: claude-opus-4-7@2026-04-12
      prompt_version: 2.1.0
      prompt_sha256: 3c8e2a1f4b9d8e7a6b5c4d3e2f1a8b7c6d5e4f3a
      rubric_version: 3.0.0
    results:
      passed_at: 2026-05-25T03:14:22Z
      duration: 2h 14m
      cost: $214.32
      cases_total: 1247
      cases_passed: 1239
      pass_rate: 99.36%
      baseline_pass_rate: 99.42%
      delta_pp: -0.06
      critical:
        total: 24
        passed: 24
      safety:
        total: 89
        passed: 89
      cross_feature:
        total: 156
        passed: 156
      cost_per_case:
        mean: $0.063
        p95: $0.142
      latency_per_case:
        p50_ms: 1840
        p95_ms: 4210
      failures:
        - case_id: med-term-normalization-047
          previous_status: pass
          new_status: pass # but borderline
          notes: "Output borderline; flagged for review"
      artifact_uri: s3://meridian-eval-archive/2026.05.25-r3/fulleval/

  canary:
    - step: 1%
      started: 2026-05-25T16:00:00Z
      ended: 2026-05-25T20:00:00Z
      decision: ramp
      approver: engineer-name
      metrics:
        quality_delta: +0.03
        cost_delta_pct: +1.2
        p95_latency_delta_ms: -20
    - step: 10%
      started: 2026-05-25T20:00:00Z
      ended: 2026-05-26T00:00:00Z
      decision: ramp (auto)
      metrics: { ... }
    - step: 50%
      started: 2026-05-26T00:00:00Z
      ended: 2026-05-26T22:00:00Z
      decision: ramp
      approver: ai-platform-lead-name
      metrics: { ... }
    - step: 100%
      promoted_at: 2026-05-26T22:30:00Z
      approver: ai-platform-lead-name

  shadow: null  # this release did not shadow first

  approvals:
    author: engineer-name
    reviewers:
      - clinical-content-reviewer-name
      - ai-platform-reviewer-name
    approvers:
      - ai-platform-lead-name
      - product-owner-name
    approved_at: 2026-05-25T15:30:00Z

  overrides: []  # no overrides on this release

  signature:
    algorithm: ed25519
    key_id: ci-bot-2026
    signed_at: 2026-05-25T15:30:00Z
    signature: <base64-encoded signature>

  deploy_outcome:
    deployed_at: 2026-05-25T16:00:00Z
    promoted_to_100_at: 2026-05-26T22:30:00Z
    initial_24h:
      quality: 7.43
      cost_per_conversation: $0.083
      p95_latency: 1.32s
      error_rate: 0.03%
      incidents: 0
```

### 3.2 Schema validation

The artifact has a published schema (JSON Schema or equivalent). The build pipeline validates the artifact against the schema before signing.

### 3.3 Backward compatibility

The schema evolves. New fields can be added (minor schema bump); existing fields cannot be removed or change meaning (major schema bump, which is rare and intentional). Releases record which schema version they conform to.

### 3.4 Discoverability

Each release artifact is published to a well-known location (object store, dedicated repo, internal portal). Engineers can fetch any release's artifact by its release ID.

---

## 4. Building the artifact

The artifact is built progressively through the pipeline.

### 4.1 At PR-merge / release-candidate creation

The build pipeline creates the initial artifact:

- Release identity.
- Code section.
- Prompts, models, datasets sections (from the release manifest).
- Build environment.

The artifact is incomplete at this stage; eval results and canary results are not yet attached.

### 4.2 At full-eval pass

The full-eval job appends its results section to the artifact:

- Eval suite version + hash.
- Pass rates, per-segment, cost, latency.
- Failures and analysis.
- Comparison to baseline.

The artifact is now "release-candidate-validated."

### 4.3 At cost / latency gate pass

Cost and latency gate results are appended (or noted within the eval results section).

### 4.4 At approvals

Approvers sign the artifact:

- Reviewer approvals appended.
- Author signature appended.
- Senior approver signature(s) appended.

The artifact is now "approved."

### 4.5 At deploy

When the deploy begins:

- The artifact is verified against the deploy gate (per pinning docs).
- Deploy timestamp recorded.

### 4.6 At canary stages

Each canary step appends a record to the canary section.

### 4.7 At promotion

The promotion-to-100% timestamp and approver are recorded.

### 4.8 At first 24h

Post-promotion, the initial production metrics are appended.

### 4.9 The "frozen" point

After the first 24h block is appended, the artifact is *frozen*: no further additions. Any subsequent state changes (rollback, incident) are tracked in *separate* records linked to the release ID.

This freeze is critical: the artifact represents the release as built and shipped. Operational events after that are operational records, not modifications to the release.

---

## 5. Signing and approval

The artifact is signed; signatures are verified at deploy time.

### 5.1 Why sign

Without signing, an artifact could be tampered with between approval and deploy. Signing closes the gap: a tampered artifact has an invalid signature; the deploy gate refuses.

### 5.2 The signing key

The CI / release authority holds a signing key. Common choices:

- **Sigstore / cosign.** Open-source standard for artifact signing.
- **Cloud KMS-backed keys.** AWS KMS, GCP Cloud KMS, Azure Key Vault.
- **HSM-backed keys.** For highest-assurance environments.

The signing key is rotated periodically; the artifact records which key signed it.

### 5.3 The signing flow

1. The artifact is built and finalized through §4.1–4.4.
2. The CI / release authority computes the cryptographic hash of the canonical form.
3. The signing key produces a signature over the hash.
4. The signature is embedded in the artifact (or attached as a sidecar).

### 5.4 The verification flow

At deploy:

1. The deploy gate fetches the artifact and its signature.
2. The signature is verified against the public key.
3. If valid: proceed. If invalid: refuse deploy, alert.

### 5.5 The approval chain

Approvals are separate from the artifact's signature:

- Approvals are records of "person P approved release R at time T."
- The artifact captures who approved.
- The signature attests that the artifact (including the approval records) was not modified after signing.

### 5.6 Override approvals

When a release ships with overrides (eval / cost / latency), the override is approved separately:

- Tech lead approves the override.
- Override approval is captured in the artifact.
- The signature covers the override record.

A tampered override (after signing) invalidates the signature; the deploy refuses.

---

## 6. Retention and archival

The artifact is the audit anchor. It is retained for the regulatory retention window.

### 6.1 Retention period

- **US healthcare (HIPAA + state laws):** 6+ years typical.
- **EU GDPR / clinical:** varies by jurisdiction; 10+ years for some clinical records.
- **Financial services:** 7+ years typical.
- **Default:** 7 years for general engineering reproducibility.

The team's retention policy is documented; the release artifacts and their referenced resources (eval suites, datasets, prompts, model weights for self-hosted) are retained for that period.

### 6.2 What is retained

Retention applies to:

- The release artifact itself.
- The eval suite version (the case content).
- The pinned prompts (content at the pinned hash).
- The pinned models (vendor-hosted: the version label; self-hosted: the weights).
- The pinned datasets (corpus contents, fine-tune data, golden sets).
- Canary and shadow result detail.
- Approval records.

If any of these is lost, the reproducibility property is broken for that release.

### 6.3 Storage

- Release artifacts: object store with versioning, retention policy, write-once-read-many (WORM) bucket policy.
- Referenced resources: their own retention; the artifact references the URI.
- Backup: cross-region replication for critical workloads.

### 6.4 Pruning

After retention period expires, artifacts and their referenced resources can be pruned. Pruning is intentional, logged, and approved.

Auto-prune jobs run nightly; their action log is itself retained for audit.

### 6.5 Per-tenant retention overrides

Some tenants require longer retention (a customer's BAA mandates 10 years instead of 6). The retention policy supports per-tenant overrides; the longest-required retention applies to any artifact involving that tenant's data.

---

## 7. The rollback consumer

The artifact's first downstream consumer is rollback.

### 7.1 Rollback as artifact-flip

Per [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md), rollback is:

- Identify the prior known-good release.
- Fetch its artifact.
- Verify signature.
- Deploy according to the artifact's pinned set.

The "deploy according to the artifact" is the canonical replay of what the artifact describes: code at the pinned commit, prompts at the pinned hashes, models at the pinned versions, datasets at the pinned versions.

### 7.2 Rollback testing

Rollback is *tested* with the artifact:

- Quarterly rollback drill in staging.
- Pick a past release; restore from its artifact; verify all components match.
- A failed drill means the artifact is incomplete or the rollback path is broken — both are bugs to fix.

### 7.3 Rollback window

The rollback window is typically 14 days (one release back; the prior release's artifact is the rollback target). For longer windows, more prior releases are kept actively-deployable.

After the rollback window, restoring a release becomes an *archive restoration* — slower, may require fetching from cold storage. Still possible, but not a hot-path operation.

### 7.4 Multi-step rollback

If multiple releases need to be rolled back (`r3` → `r2` → `r1`), each is a separate rollback step, each with its own artifact verification.

The team does not do "rollback to release of two weeks ago" as a single operation; instead, the sequence of rollbacks is executed and verified one step at a time.

---

## 8. The audit / compliance consumer

The artifact's second downstream consumer is audit.

### 8.1 The audit questions

Common audit questions and how the artifact answers:

- "What clinical reference data was the Care Coordinator using on 2026-05-22 at 14:30 UTC for tenant H?"
  - Answer: Release `2026.05.25-r3` was live then; artifact pins `care_coordinator_clinical_corpus` for tenant H to `2026.05.20-hospital_h@sha:8e1a9f...`.

- "What was the eval pass rate at the time of release?"
  - Answer: Artifact records `99.36%` against `eval_suite@4.2.0`.

- "Who approved this release?"
  - Answer: Artifact records the approvers and their roles.

- "Was the canary successful?"
  - Answer: Artifact records each canary step's metrics and decision.

- "Was there any override on this release?"
  - Answer: Artifact's overrides section is empty (or, if not, the override target + justification + approver are recorded).

- "How do we know the artifact wasn't tampered with?"
  - Answer: The signature verifies; the signing key was held by the CI authority; the public key is published.

### 8.2 The audit interface

The team provides auditors:

- A read-only interface to fetch any release artifact by ID.
- A search interface for "which release was live at time T."
- A verification tool that checks any artifact's signature.
- Documentation of the schema.

### 8.3 The audit drill

The team runs an audit drill quarterly:

- Pick a past production decision (a specific user's interaction at a specific time).
- Use the artifacts to reproduce what was running.
- Verify all components are restorable.

A failed drill means a regulatory exposure; treat it as a high-severity finding.

### 8.4 Per-tenant audit

For multi-tenant systems, audits often ask "what was running for this specific tenant?" The artifact's per-tenant pins make this answerable:

- Look up the release-id for the tenant's region at the timestamp.
- Look up the artifact.
- Read the per-tenant section of each pinned resource (prompts, models, datasets).

The answer is precise.

---

## 9. Worked Meridian example: a release artifact end-to-end

Following the prompt update from [pipeline-architecture-for-ai.md §10](./pipeline-architecture-for-ai.md), here is the artifact lifecycle.

### 9.1 Artifact creation

At PR merge, CI creates `release/2026.05.25-r3/artifact.yaml`:

- Release identity, code section, prompts section, models section, datasets section.

The artifact is registered in the release registry with status `pending-eval`.

### 9.2 Full-eval pass

The full-eval job appends results to the artifact:

- 1247 cases run; 1239 passed; pass rate 99.36%.
- All critical and safety cases passed.
- Cost +3.3%, latency +1.7% — within thresholds.

The artifact status changes to `eval-passed`.

### 9.3 Approval

The release author requests approval:

- AI Platform reviewer approves.
- Clinical content reviewer approves.
- Product owner approves.
- AI Platform lead approves.

Each approval is appended with timestamp. The artifact status changes to `approved`.

### 9.4 Signing

The CI bot signs the artifact:

- Hash computed over the canonical form.
- Ed25519 signature attached.
- Public key reference recorded.

The artifact status changes to `signed-ready`.

### 9.5 Deploy

The deploy gate verifies:

- Signature valid.
- Every prompt pin resolves.
- Every model pin resolves.
- Every dataset pin resolves.
- No deprecated artifacts pinned.
- Schema compatibility confirmed.

Deploy proceeds. The deploy timestamp is appended to the artifact.

### 9.6 Canary

Each canary step appends to the artifact:

- 1%: ramp, engineer-name approved at T+4h, metrics within tolerance.
- 10%: ramp (auto), at T+8h, metrics within tolerance.
- 50%: ramp, AI Platform lead approved at T+32h, metrics within tolerance.
- 100%: promoted at T+32h, AI Platform lead approved.

### 9.7 24h baseline

24 hours after 100% promotion, the initial production metrics are appended:

- Quality: 7.43 (vs baseline 7.42).
- Cost per conversation: $0.083.
- p95 latency: 1.32s.
- Error rate: 0.03%.
- Incidents: 0.

The artifact is now frozen.

### 9.8 Six weeks later: an audit question

Compliance asks: "What was the Care Coordinator running for tenant H on 2026-05-28 at 14:30 UTC?"

The answer chain:

- Deploy log: release `2026.05.25-r3` was live (promoted at 2026-05-26T22:30Z; next promotion later than 2026-05-28).
- Artifact `release/2026.05.25-r3/artifact.yaml`:
  - Code: `meridian-registry.local/care-coordinator@2026.05.25-r3`.
  - Prompts for tenant H: `care_coordinator_supervisor@2.4.1-hospital_h@sha:8e1a9f...` (and 11 others).
  - Models for tenant H: `claude-opus-4-7@2026-04-12` (and others, same as default for this release).
  - Datasets for tenant H: `care_coordinator_clinical_corpus@2026.05.20-hospital_h@sha:8e1a9f...`.
- All pinned resources fetched from their retained storage; signatures verified.
- Reviewer has a complete, cryptographically attestable answer.

### 9.9 Three months later: a rollback drill

SRE runs the quarterly drill:

- Pick `release/2026.05.25-r3`.
- Stand up the deployment in staging from the artifact.
- Verify: code matches commit; prompts match hashes; models pinned correctly; datasets fetched and hash-verified.
- Run smoke test: a known query produces a response indistinguishable from production.
- Drill PASS.

### 9.10 Findings closed

- **ARCH-CARE-088** (release records were Slack messages; "what was running" was unanswerable).
- **ARCH-CARE-089** (no release artifact format; releases varied by who built them).
- **ARCH-CARE-090** (no signature; artifacts could in theory be modified post-approval).
- **ARCH-CARE-091** (rollback worked but was undocumented; rollback drill had not been run).
- **ARCH-CARE-092** (audit interface absent; compliance asks were ad-hoc engineering tasks).
- **ARCH-CARE-093** (retention policy not aligned with HIPAA 6-year requirement).

---

## 10. Anti-patterns

### 10.1 The implicit release

The release is "whatever was in the prompt store + model registry + data store at deploy time." Nothing is captured in one place. Reproducibility is theoretical; debugging is detective work.

The fix: explicit artifact. One file that captures the full pinned set.

### 10.2 The artifact-without-eval

The artifact captures code, prompts, models — but not the eval result. Six months later, an incident review asks "what did the eval say at release time?" The answer is "we ran it, but we didn't save the result."

The fix: eval result is part of the artifact. The release pipeline refuses to promote an artifact missing the eval result.

### 10.3 The unsigned artifact

Approvals are recorded in the artifact but no signature attests to the artifact's integrity. A clever attacker could modify the artifact between approval and deploy.

The fix: signatures. The deploy gate verifies; tampered artifacts refuse to deploy.

### 10.4 The post-hoc artifact

The artifact is constructed *after* deploy, from logs and chat history, when an audit asks. The construction is fragile, partial, and unverifiable.

The fix: the artifact is built progressively through the pipeline, frozen at promotion. No after-the-fact reconstruction.

### 10.5 The retention-blind storage

Artifacts are stored in a default bucket with default retention. Some artifacts age out within months. Audit-window compliance fails.

The fix: WORM bucket with retention policy aligned to regulatory requirements. Per-tenant overrides applied where needed.

### 10.6 The schema-creep

The artifact format evolves ad-hoc. Different releases have different fields; no central schema. Tooling that reads artifacts breaks on cross-version reads.

The fix: published schema with version. Forward-compatible evolution; major bumps are rare and coordinated.

### 10.7 The fork-of-artifact

A release ships; someone notices an issue with the artifact (a misspelled approver name). They edit the artifact directly. Now the artifact and its signature disagree; the signature is invalid; trust is broken.

The fix: the artifact is immutable after freeze. Errors trigger a *new* artifact with an amendment record, not a silent edit.

### 10.8 The orphan artifact

Artifacts reference resources (datasets, prompts, models) that have been pruned from their backing stores. The artifact survives but cannot be replayed because the resources are gone.

The fix: retention applies to *referenced resources* too, not just the artifact. Pruning a resource that is referenced by a retained artifact is a hard error.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| CICD-RA-001 | No release artifact format; releases vary by author | High | Define and publish schema per §3; CI builds artifact per §4 | AI Platform |
| CICD-RA-002 | Eval results not part of release artifact | High | Append eval results at gate-pass per §4.2; refuse promotion without them | AI Platform + Eval Eng |
| CICD-RA-003 | Artifacts unsigned; tamper undetectable | High | Sign artifacts per §5; deploy gate verifies | AI Platform + SRE |
| CICD-RA-004 | Retention period unaligned with regulatory requirements | High | Document retention policy per §6.1; WORM bucket with per-tenant overrides | AI Platform + Compliance |
| CICD-RA-005 | Approvals captured in chat, not artifact | High | Approvals appended to artifact per §4.4; signed | AI Platform |
| CICD-RA-006 | Canary results not preserved per release | High | Append canary results per §4.6 | AI Platform |
| CICD-RA-007 | Shadow results, when shadow ran, not captured in artifact | Medium | Append shadow results per §4.6 | AI Platform |
| CICD-RA-008 | Per-tenant pins not enumerated in artifact | Medium | Per-tenant sections per pinning docs; artifact carries them | AI Platform + Architecture |
| CICD-RA-009 | Overrides not captured | High | Override target + justification + approver in artifact per §2.12 | AI Platform |
| CICD-RA-010 | Artifact mutable after freeze; corrections via silent edits | High | Immutable after freeze; corrections are new artifacts with amendment record | AI Platform |
| CICD-RA-011 | Referenced resources (datasets, prompts, models) prunable while artifact retained | High | Retention applies to referenced resources; pruning blocked while retained | AI Platform + Data Eng |
| CICD-RA-012 | Audit interface absent; "what was running" asks are ad-hoc | High | Build audit interface per §8.2; quarterly drill | AI Platform + Compliance |
| CICD-RA-013 | Rollback drill not run; rollback path untested | Medium | Quarterly drill in staging per §7.2 | SRE + AI Platform |
| CICD-RA-014 | Signing key rotation undocumented | Medium | Document key rotation; record key id in artifact | AI Platform + SRE |
| CICD-RA-015 | Schema evolution ad-hoc; cross-version reads break | Low | Published schema with version; forward-compatible evolution | AI Platform |
| CICD-RA-016 | Per-tenant retention overrides not enforced | Medium | Longest-required retention applies; per-tenant override mechanism per §6.5 | Compliance + AI Platform |
| CICD-RA-017 | 24h baseline metrics not captured | Low | Append initial production metrics per §4.8 | AI Platform + Observability |
| CICD-RA-018 | Deploy outcome (incidents, rollbacks) tracked separately from artifact, no link | Low | Operational events link to release-id; artifact stays frozen but referenced | SRE + AI Platform |

---

## 12. Adoption checklist

- [ ] Release artifact schema published; CI builds artifacts conforming to the schema.
- [ ] Artifact built progressively through pipeline: identity → code → pins → eval → approvals → signature → deploy → canary → 24h → frozen.
- [ ] Eval results captured: pass rate, critical, safety, cross-feature, cost, latency, failures, comparison.
- [ ] Pinning sections (prompts, models, datasets) carry version + hash + per-tenant overrides.
- [ ] Canary results recorded per ramp step with metrics, decision, approver.
- [ ] Shadow results recorded when shadow ran.
- [ ] Approval chain captured: author, reviewers, approvers, timestamps, rationale.
- [ ] Overrides captured with target, justification, approver, expiration.
- [ ] Artifact signed by CI / release authority; deploy gate verifies signature.
- [ ] Retention policy aligned with regulatory requirements; WORM bucket; per-tenant overrides.
- [ ] Referenced resources retained alongside artifact; pruning blocked while artifact retained.
- [ ] Audit interface available; quarterly audit drill.
- [ ] Rollback drill quarterly in staging; full-restore from artifact verified.
- [ ] Artifact immutable after freeze; corrections are new artifacts with amendment records.
- [ ] Signing key rotation documented; key id recorded in artifact.
- [ ] Schema evolution coordinated; forward-compatible by default.
- [ ] 24h baseline production metrics appended post-promotion.
- [ ] Operational events (incidents, rollbacks) linked to release-id without modifying frozen artifact.

---

## 13. References

**Internal:**

- [pipeline-architecture-for-ai.md](./pipeline-architecture-for-ai.md) — the pipeline that builds the artifact.
- [eval-gate-design.md](./eval-gate-design.md) — produces the eval result that goes into the artifact.
- [canary-rollouts.md](./canary-rollouts.md) — produces the canary results.
- [shadow-traffic.md](./shadow-traffic.md) — produces the shadow results.
- [prompt-version-pinning.md](./prompt-version-pinning.md) — the prompt pins in the artifact.
- [model-version-pinning.md](./model-version-pinning.md) — the model pins in the artifact.
- [dataset-version-pinning.md](./dataset-version-pinning.md) — the dataset pins in the artifact.
- [model-lifecycle/rollback-procedures.md](../model-lifecycle/rollback-procedures.md) — consumes the artifact for rollback.
- [model-lifecycle/model-promotion.md](../model-lifecycle/model-promotion.md) — promotion event that finalizes the artifact.
- [model-lifecycle/canary-and-shadow-rollout.md](../model-lifecycle/canary-and-shadow-rollout.md) — model-level mechanic feeding canary section.
- [model-lifecycle/ab-model-testing.md](../model-lifecycle/ab-model-testing.md) — A/B results referenced from artifact when applicable.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — upstream eval design.
- [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — live-judge for canary results.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — trace shape recording release-id per call.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — cost signal for the artifact's cost baseline.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — operational events linked to release-id.

**Cross-repo (architecture sibling):**

- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
- [reference-systems/adoption-sequencing-across-systems.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/adoption-sequencing-across-systems.md) — adoption sequence including artifact discipline.
- [data-architecture-for-ai/lineage-and-provenance.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/lineage-and-provenance.md) — architecture-side framing of provenance and reproducibility.
