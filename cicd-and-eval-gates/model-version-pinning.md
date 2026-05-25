# Model Version Pinning

> **Audience.** Engineers building or refactoring the release artifact / CI pipeline for AI features. Tech leads who have seen "the model auto-upgraded behind an alias" become a production incident. **Scope.** The *engineering* pattern for pinning specific model versions in release artifacts and enforcing the pin at deploy time. Pair with [model-registry.md](../model-lifecycle/model-registry.md) (the catalog) and [prompt-version-pinning.md](./) (the prompt-side companion, coming). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A model is a versioned dependency. The model in production today is not the same as the model in production yesterday if either the team or the provider changed which version is being used. Without explicit pinning, model version is a *runtime* property — whatever the provider's alias resolver returns at the moment of the call. With pinning, model version is a *release-time* property — explicitly committed in the release artifact, validated at deploy, and visible in every trace.

The Care Coordinator's `ARCH-CARE-004` finding (model versions referenced by alias) is the symptom. The pattern this document describes is the fix.

This document is opinionated about three things:

1. **Every release pins every model version.** No aliases. No "use the latest" semantics. The release artifact specifies, for each model the system uses, the exact provider+model+version that will be called.
2. **Deploy gate enforces the pin.** A release that pins a non-existent or deprecated model is refused at deploy time. The pin is a contract; the contract is verified before production.
3. **Rollback restores the full pinned set.** A code rollback restores the prior release's pinned models alongside the prior code. There is no partial-state rollback.

Structure: (2) the pin contract; (3) the release manifest; (4) the deploy gate; (5) the integration with the model registry; (6) deny-list patterns; (7) multi-model pinning; (8) rollback discipline; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The pin contract

Each release pins, for each AI feature in the release, the model version that feature will use. The pin is:

- **Specific.** Provider + model + version-string. `claude-opus-4-7@2026-04-12`, not `claude-opus-latest`.
- **Required.** No model-version is implicit. A release without a pin for a feature that uses a model is refused.
- **Validated.** The deploy gate validates the pin against the model registry.
- **Stable across the release.** All instances of the deployed release use the same pin. There is no per-instance variation.

### 2.1 What "pin" means

A pin is a specific, version-string-level reference to a model. It is the exact identifier the provider's API accepts as the `model` parameter, including the version suffix (where applicable):

- Anthropic: `claude-opus-4-7@2026-04-12` (the `@2026-04-12` is the version).
- OpenAI: `gpt-5-2026-01-10` (the date suffix is the version).
- Google: `gemini-2.5-pro-002` (the `-002` is the version).
- Self-hosted: `meta-llama/Llama-4-70b-instruct@sha:7fa3b1...` (the sha is the version).

Without the version component, the team is referencing an alias.

### 2.2 What pinning prevents

- **Silent provider-side version upgrades.** When a provider releases a new version of a model behind an alias, the team's calls silently shift to the new version. Behavior may change in ways no eval anticipated.
- **Inadvertent version drift between environments.** Dev points at `latest`, prod points at `latest`, and they end up calling different versions at the same time.
- **Reproducibility loss.** When asked "what model was running last week," the team has to reason about provider alias state rather than reading a manifest.
- **Audit gaps.** Regulatory reviewers cannot attest to which model was used for a clinical decision without the version captured at release time.

### 2.3 What pinning does NOT prevent

- **Behavior changes within a pinned version.** Some providers do release sub-version updates that change behavior; pinning protects against major version drift but not against vendor silent-fixes.
- **Pricing changes.** Provider prices can change for a pinned version. Pinning is a behavioral contract, not a pricing contract.
- **Deprecation.** Pinned versions still deprecate; the pin keeps the version stable until you choose to migrate.

---

## 3. The release manifest

The release manifest is the consolidated pin specification. Every release ships with one.

### 3.1 The manifest shape

```yaml
release:
  version: 2026.05.25-r3
  code_sha: 9c2a1f8b...
  timestamp: 2026-05-25T14:30:00Z
  released_by: ci-bot
  approval_chain:
    - ai-platform-eng
    - sre

  models:
    care_coordinator_supervisor: claude-opus-4-7@2026-04-12
    care_coordinator_clinical_knowledge: claude-opus-4-7@2026-04-12
    care_coordinator_drafting: claude-sonnet-4-6@2025-08-15
    care_coordinator_classifier: claude-haiku-4-5@2025-10-01
    care_coordinator_query_rewriter: claude-haiku-4-5@2025-10-01
    care_coordinator_supervisor_fallback: claude-sonnet-4-6@2025-08-15
    care_coordinator_degraded_mode: claude-haiku-4-5@2025-10-01
    embedding_corpus_ingestion: openai-text-embedding-3-large@2024-01-25
    cohere_reranker: cohere-rerank-3.5@2025-09-01

  prompts:
    care_coordinator_supervisor: 2.4.1
    care_coordinator_classifier: 1.2.0
    # ... (per [prompt-version-pinning.md] coming)

  datasets:
    clinical_guidelines_corpus: 2026-Q2-1
    drug_interaction_graph: 2026-05-01

  eval_suite:
    version: 2026-05-15
    pass_rates:
      clinical_golden_set: 95.2
      drug_interaction_subset: 98.1
      conversational_subset: 91.5
```

The manifest is committed to version control alongside the code. It is the reproducibility contract.

### 3.2 What the manifest covers

- **Models.** Per role in the system, the pinned model version. Every role that calls a model has a pin.
- **Prompts.** Per [prompt-version-pinning.md](./) (coming).
- **Datasets.** Per [dataset-version-pinning.md](./) (coming) — for systems where dataset versions are part of the behavior.
- **Eval suite version.** Which eval suite was used to validate this release.

### 3.3 Per-role pinning

Notice that the manifest pins per *role* (care_coordinator_supervisor, care_coordinator_classifier) rather than per *model*. This is important because:

- Multiple roles can use the same model version (supervisor and clinical_knowledge both use Opus 4.7).
- A role's model can be changed independently (the classifier could be moved from Haiku to Sonnet without affecting the supervisor).
- Fallback paths have their own pin (supervisor's fallback uses Sonnet; the regular supervisor uses Opus).
- The role names match the prompt names and the trace span names — the team's mental model is consistent.

### 3.4 Manifest generation

The manifest is generated by the CI pipeline as part of building the release:

1. CI reads the configuration from the codebase (which roles exist, what models they are configured for).
2. CI validates each pin against the model registry.
3. CI captures eval suite results.
4. CI generates the manifest YAML.
5. The manifest is committed alongside the release artifact.

The manifest is not written by hand; the codebase + the registry produce it.

---

## 4. The deploy gate

The deploy pipeline reads the manifest and validates before deploying.

### 4.1 The checks

For each pin in the manifest:

- **Model exists in registry.** Per [model-registry.md](../model-lifecycle/model-registry.md). Unregistered models cannot be deployed.
- **Model status is active.** Deprecated models can be deployed only with explicit override (warning, not block). Retired models cannot be deployed.
- **Model is allowed for the role.** The role's name is checked against the model's `allowed_features` in the registry.
- **BAA / regulatory coverage matches.** For features flagged as PHI / regulated, the pinned models must be appropriately covered.
- **Pricing is current.** The model's pricing in the registry is checked against the manifest's recorded pricing version; mismatch produces a warning (the manifest may need to be regenerated).

### 4.2 The fail-fast pattern

If any check fails, the deploy stops. The team is notified with the specific failure. The deploy does not partially execute.

This is the load-bearing discipline. Without fail-fast, partial deploys leave the system in an inconsistent state (some new roles using new models, some still using old models that may have been deprecated).

### 4.3 The override pattern

Some deploys need to use a deprecated model deliberately — e.g., the team is testing migration scenarios, or the new model has a known issue and the team is temporarily continuing with the old. The override:

- Label on the PR / deploy command: `[deploy-override: justified-deprecated-use]`.
- Reviewer approval required (typically the model's owner team).
- Override logged in the deploy audit.
- Override is bounded — typically expires after a defined period (e.g., 14 days).

Overrides are exceptional. Frequent overrides suggest the registry's deprecation policy is mis-calibrated.

### 4.4 The deploy gate vs the gateway

The deploy gate validates pins at deploy time; the gateway (per [ai-gateway-pattern.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md)) validates calls at runtime. Both are needed:

- Deploy gate catches manifest-time issues (pinned model retired since the previous deploy).
- Gateway catches runtime issues (model retired between deploy and runtime — rare but possible during long-running deploys).

The two-layer defense ensures that retired or unauthorized models cannot be called even if they slipped past one of the checks.

---

## 5. Integration with the model registry

The pin discipline depends on the model registry per [model-registry.md](../model-lifecycle/model-registry.md). The integration:

### 5.1 The registry is the source of truth for available pins

When the manifest is being generated, the CI pipeline reads from the registry to validate each pin. The set of "valid pins" is exactly the set of "active models in the registry."

### 5.2 The pin lookup

Application code references a role's pin via a manifest lookup:

```python
# At deploy time, MANIFEST is loaded
model_pin = MANIFEST.models["care_coordinator_supervisor"]
# model_pin = "claude-opus-4-7@2026-04-12"

# The wrapper resolves the pin to the registry artifact
model_artifact = registry.get(model_pin)
# model_artifact.provider = "anthropic"
# model_artifact.api_model_string = "claude-opus-4-7@2026-04-12"

# The gateway uses the artifact to dispatch the call
response = gateway.call(
    provider=model_artifact.provider,
    model=model_artifact.api_model_string,
    ...,
)
```

The pin in the manifest is the registry artifact's identifier. The application code is decoupled from the provider's specific API surface.

### 5.3 Manifest validation against registry

CI validates manifest-against-registry at:
- Pre-merge (PR checks).
- Pre-deploy (deploy gate).
- Pre-promotion (staging-to-prod checks).

Validation failures at any stage prevent the deploy.

---

## 6. Deny-list patterns

Some models should not be used regardless of registration status. The deny-list pattern:

### 6.1 The platform-wide deny list

A configuration that names models that cannot be deployed, even if registered. Uses:

- **Models with known security issues.** A model with a known prompt-injection vulnerability is denied until patched.
- **Models with deprecated provider contracts.** A model whose provider contract has lapsed is denied.
- **Models that failed eval validation.** A model that was registered but never passed eval is denied for production use.
- **Models that are in a "do not use" state for any reason.** Including emergency deprecations.

The deny list is a stricter version of the registry's status field. A model on the deny list is treated as if it were retired, even if the registry status is something else.

### 6.2 Per-feature deny lists

Some features cannot use specific models even when the platform allows them. Example: the analytics-copilot has a deny list of models that have not been evaluated against its workload. The per-feature deny list is more granular than the platform-wide one.

### 6.3 The deny-list enforcement

Same as the deploy gate: pinned models on the deny list are refused. Same as the gateway: deny-list calls are refused at runtime.

---

## 7. Multi-model pinning

The manifest pins multiple models. The discipline:

### 7.1 The bill of materials view

The manifest is the AI feature's bill of materials. For each release:

- Which models are used (and which versions).
- Which prompts are used (and which versions).
- Which datasets are used (and which versions).

The bill of materials is queryable; for any release, the team can list every dependency.

### 7.2 Dependency analysis

When a model is being deprecated, the team queries the bill of materials across releases to identify:
- Which current releases pin this model.
- Which roles in those releases use this model.
- The migration scope.

The query is the foundation of the migration project.

### 7.3 Per-environment manifests

Different environments (dev, staging, prod) often pin different model versions (dev tests the new version that prod has not yet adopted). Each environment has its own manifest; the manifests are tracked together so the team can see "this is what dev is doing vs what prod is doing."

The discipline: divergence between environments is deliberate, not accidental. A dev / prod manifest diff that the team did not intend is a signal.

---

## 8. Rollback discipline

Rollback restores the full pinned set. The discipline:

### 8.1 The rollback artifact

Each release's manifest is preserved (committed in version control, archived in deploy artifact storage). Rolling back to release N restores the manifest of release N.

### 8.2 The full-set rollback

When rolling back:
- The previous release's manifest is loaded.
- The application is deployed at the previous release's code SHA.
- Model pins from the previous manifest are now active.
- Prompt pins from the previous manifest are now active.

The rollback is atomic from the application's perspective. There is no partial state where code is at version N but prompts or models are at version N+1.

### 8.3 The forward-incompatible-rollback edge case

Some rollbacks are forward-incompatible — the previous release pinned a model that has since been retired. The rollback would fail the deploy gate.

The response: either an emergency override (with security-eng + ai-platform-eng + sre approval), or a forward-fix (a new release that addresses the rollback need while pinning to a non-retired model).

This is rare. Most rollback targets are recent enough that all pinned models are still active.

### 8.4 The model-only rollback

Sometimes the rollback need is model-version-specific — a new model version is misbehaving but the code change is fine. The pattern:

- A patch release is prepared with the previous model pin restored, code unchanged.
- The patch release deploys; production reverts to the previous model.
- The follow-up: investigate the model issue; decide whether to migrate back later or stay on the previous version permanently.

This pattern requires the release manifest to be the pin contract; if model pins live in code, model rollback is a code rollback (which has more side effects).

---

## 9. Worked Meridian Health example

### 9.1 The Care Coordinator release manifest

The example in section 3.1 is from a real Meridian Care Coordinator release. The manifest pins 9 model roles, 5+ prompts, and 2 datasets.

### 9.2 The deploy gate in action

When the Care Coordinator release 2026.05.25-r3 was deployed:

1. CI generated the manifest from the codebase + registry state.
2. Deploy gate validated:
   - All 9 model pins exist in the registry: pass.
   - All 9 model pins are active: pass.
   - All 9 pins are allowed for their roles: pass.
   - All BAA-required features pin BAA-covered models: pass.
3. Deploy proceeded.

A previous attempted release (2026.04.15-r2) failed the gate:
- The release pinned `claude-opus-4-6` for the supervisor.
- Opus 4.6 status was `deprecated`.
- Deploy gate refused.
- Team updated the manifest to pin Opus 4.7 instead.
- Re-tested with the updated pin; eval passed; deploy proceeded as 2026.04.15-r3.

The discipline caught the deprecation-and-deploy mismatch before production was affected.

### 9.3 The 2026-04-29 incident

The cost incident referenced in [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) and the model-pinning finding `ARCH-CARE-004`:

- Pre-incident: the supervisor was configured to use `claude-opus-4` (an alias).
- Anthropic released claude-opus-4-7 behind the alias.
- The team's calls silently shifted to the new version.
- Per-call cost rose ~15%; aggregate cost rose into the per-feature circuit-breaker zone.
- The circuit-breaker tripped; on-call investigated.
- Diagnosis: aliases caused silent version drift.

The remediation:
- All model references migrated to full version pins.
- Registry refuses aliases.
- Release manifests pin versions explicitly.
- Deploy gate validates against the registry.

Post-remediation, model version changes are deliberate decisions, not silent provider-side shifts.

### 9.4 The manifest publication

Each release's manifest is committed to the `meridian-releases` repository alongside the release. The manifest is searchable, diff-able, and historical.

The release dashboard surfaces:
- The current production release manifest.
- Diffs between successive manifests.
- The full history of model pins per role.

Cross-team consumers (security, compliance, finops) consume the manifest as the source of truth for what is running in production.

### 9.5 The platform discipline

- Every Meridian Care Coordinator release ships with a manifest.
- The manifest is generated by CI; not hand-edited.
- The deploy gate validates against the registry; failures block deploy.
- Model version changes require a manifest update (a PR), eval-validation, and approval.
- The audit log records every deployed manifest with timestamp and approver.

---

## 10. Anti-patterns

### 10.1 "Model versions in environment variables"

Production model version is in an environment variable that operations can change without a deploy. The "release" of a model change is just a config flip; there is no manifest, no eval gate, no audit.

**Corrective.** Model pins in the manifest, committed in version control. Environment variables are for runtime configuration (credentials, endpoints), not for model selection.

### 10.2 "Aliases in the manifest"

The manifest says `claude-opus-latest`. The manifest acquires the same drift problem the pinning was supposed to solve.

**Corrective.** Manifest validation refuses aliases. Only full version strings are accepted.

### 10.3 "Deploy gate is advisory"

The deploy gate produces warnings but does not block. Deploys with bad pins proceed; the team learns about the problem in production.

**Corrective.** Deploy gate is blocking. Override pattern (section 4.3) handles legitimate exceptions.

### 10.4 "Per-role pinning not used; models pinned globally"

The manifest pins "model = claude-opus-4-7" as one global setting. Every role uses the same model. The team cannot pin different versions for different roles or have a fallback path on a different version.

**Corrective.** Per-role pinning per section 3.3. Each role's model is its own pin.

### 10.5 "Rollback restores code but not models"

Rolling back code does not restore the previous release's model pins. The system runs new model pins against old code; behavior is hybrid.

**Corrective.** Rollback restores the full manifest. Atomic from the application's perspective.

### 10.6 "Manifest is hand-edited"

The manifest is a YAML file that engineers edit by hand. Drift between code and manifest accumulates.

**Corrective.** Manifest generated by CI from code + registry state. Hand-edited manifests are rejected.

### 10.7 "Pin validation happens at runtime only, not deploy time"

The deploy proceeds; runtime gateway then refuses the unregistered or retired model. The deploy "succeeded" but the application is broken in production.

**Corrective.** Deploy gate catches the issue before production. The gateway is the runtime defense-in-depth.

### 10.8 "No deny-list"

A model with a known issue (security vulnerability, regulatory issue) cannot be quickly excluded from production. The team has to deprecate it in the registry, but deprecation is a slow process.

**Corrective.** Deny-list pattern per section 6 for quick exclusion. The deny-list is the emergency mechanism; the registry's status field is the slower, more deliberate one.

---

## 11. Findings (sprint-assignable)

### CICD-PIN-001 — Severity: Critical
**Finding.** Production model references are aliases; silent version drift has caused observed cost or behavior incidents.
**Recommendation.** Migrate every model reference to a full version pin; registry refuses aliases.
**Owner.** ai-platform-eng, sprint N+1.

### CICD-PIN-002 — Severity: Critical
**Finding.** No release manifest captures the model pins; rollback restores code but not models.
**Recommendation.** Generate release manifest per section 3; commit alongside code; rollback restores the manifest.
**Owner.** ai-platform-eng + sre, sprint N+1.

### CICD-PIN-003 — Severity: Critical
**Finding.** Deploy gate does not validate model pins against the registry; deprecated or retired models can be deployed.
**Recommendation.** Deploy gate per section 4; fail-fast on any pin issue; override pattern for legitimate exceptions.
**Owner.** ai-platform-eng + sre, sprint N+1.

### CICD-PIN-004 — Severity: High
**Finding.** Model pins live in environment variables or operations-side configuration; the "release" of a model change is a config flip.
**Recommendation.** Model pins in the version-controlled manifest; env vars for runtime config only.
**Owner.** ai-platform-eng, sprint N+2.

### CICD-PIN-005 — Severity: High
**Finding.** Release manifest is hand-edited; drift between code and manifest accumulates.
**Recommendation.** CI generates the manifest from code + registry; hand-edited manifests rejected.
**Owner.** ai-platform-eng, sprint N+2.

### CICD-PIN-006 — Severity: High
**Finding.** Pin is global (one model for all roles); per-role pinning not available.
**Recommendation.** Per-role pinning per section 3.3.
**Owner.** ai-platform-eng, sprint N+2.

### CICD-PIN-007 — Severity: High
**Finding.** Rollback restores code but model pins are unchanged; hybrid-state production results.
**Recommendation.** Atomic rollback per section 8; previous manifest is the rollback target.
**Owner.** ai-platform-eng + sre, sprint N+2.

### CICD-PIN-008 — Severity: High
**Finding.** Deploy gate is warning-only, not blocking; deploys with bad pins proceed.
**Recommendation.** Make blocking. Override pattern per section 4.3.
**Owner.** ai-platform-eng + sre, sprint N+2.

### CICD-PIN-009 — Severity: High
**Finding.** No deny-list pattern; models with known issues cannot be quickly excluded without going through full deprecation.
**Recommendation.** Deny-list per section 6; emergency exclusion mechanism documented.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### CICD-PIN-010 — Severity: High
**Finding.** Per-environment manifests are not tracked or diff-able; dev / staging / prod divergence is not visible.
**Recommendation.** Per-environment manifests committed; release dashboard surfaces diffs.
**Owner.** ai-platform-eng + sre, sprint N+3.

### CICD-PIN-011 — Severity: Medium
**Finding.** Manifest does not include eval suite results; cannot verify which eval version validated the release.
**Recommendation.** Include eval-suite version and pass rates in the manifest per section 3.1.
**Owner.** ai-platform-eng, sprint N+3.

### CICD-PIN-012 — Severity: Medium
**Finding.** Manifest publication / dashboard does not exist; cross-team consumers cannot query "what is in production."
**Recommendation.** Release dashboard surfaces current manifest and history per section 9.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### CICD-PIN-013 — Severity: Medium
**Finding.** Forward-incompatible-rollback scenarios (rollback target pins a retired model) are not handled in the runbook.
**Recommendation.** Document the scenario per section 8.3; emergency override + forward-fix pattern.
**Owner.** ai-platform-eng + sre, sprint N+3.

### CICD-PIN-014 — Severity: Medium
**Finding.** Per-feature deny lists do not exist; only platform-wide deny list is available.
**Recommendation.** Per-feature deny lists per section 6.2 for the workloads where granularity is needed.
**Owner.** ai-platform-eng, sprint N+4.

### CICD-PIN-015 — Severity: Medium
**Finding.** Override usage is not tracked; frequent overrides go unnoticed.
**Recommendation.** Track override usage; monthly report; high override rates trigger registry-policy review.
**Owner.** ai-platform-eng + sre, sprint N+4.

### CICD-PIN-016 — Severity: Low
**Finding.** Manifest format is undocumented; new engineers learning the system have to read the CI code.
**Recommendation.** Schema documentation for the manifest; commit alongside the codebase.
**Owner.** ai-platform-eng, sprint N+5.

### CICD-PIN-017 — Severity: Low
**Finding.** Long-running deploys can have model pins become invalid mid-deploy (model retired between manifest generation and deploy execution).
**Recommendation.** Gateway runtime validation per section 4.4 catches this; document the failure mode.
**Owner.** ai-platform-eng, sprint N+5.

### CICD-PIN-018 — Severity: Low
**Finding.** Per-tenant model overrides (premium tier on dedicated infrastructure with different model pins) are not surfaced in the manifest.
**Recommendation.** Manifest captures per-tenant overrides where applicable; deploy gate validates each.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team without model-version pinning:

- [ ] **Sprint 0 — inventory.** Catalog every model reference in the codebase. Identify aliases vs full version strings.
- [ ] **Sprint 1 — model registry.** Stand up the registry per [model-registry.md](../model-lifecycle/model-registry.md). Pin discipline depends on it.
- [ ] **Sprint 1 — full version strings.** Migrate every model reference to a full version string. The registry refuses aliases.
- [ ] **Sprint 2 — manifest format.** Define the release manifest schema per section 3.
- [ ] **Sprint 2 — CI manifest generation.** CI generates the manifest from code + registry.
- [ ] **Sprint 3 — deploy gate.** Deploy pipeline validates the manifest against the registry per section 4; fail-fast.
- [ ] **Sprint 3 — rollback.** Rollback restores the full previous manifest per section 8.
- [ ] **Sprint 4 — per-role pinning.** Manifest pins per-role rather than per-model; supports independent role evolution.
- [ ] **Sprint 4 — deny lists.** Deny-list pattern for emergency exclusion.
- [ ] **Sprint 5 — dashboards.** Manifest publication / release dashboard for cross-team visibility.
- [ ] **Sprint 5 — runbooks.** Rollback runbook; forward-incompatible-rollback handling.
- [ ] **Ongoing — discipline.** Override usage tracked; manifest reviews quarterly; deprecation tracking.

A team that completes this sequence has the model-as-dependency discipline that turns "silent provider-side version drift" into "deliberate, eval-validated, reviewable model adoption."

---

## 13. References

- Software dependency-pinning practices (package-lock.json, Gemfile.lock, Pipfile.lock) — the source pattern this adapts to models.
- Container image pinning by digest (Docker / OCI image references with `@sha256:...`) — the equivalent pattern for container deployments.
- Anthropic, OpenAI, Google API documentation on model versioning and aliases.
- This repo: [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — the registry that this document's pinning consumes.
- This repo: [model-lifecycle/model-promotion.md](../model-lifecycle/) (coming) — the dev → staging → prod promotion that the manifest supports.
- This repo: [model-lifecycle/rollback-procedures.md](../model-lifecycle/) (coming) — the rollback discipline this document hooks into.
- This repo: [cicd-and-eval-gates/prompt-version-pinning.md](./) (coming) — the prompt-side companion.
- This repo: [cicd-and-eval-gates/release-artifacts-for-ai.md](./) (coming) — the broader release-artifact framework.
- This repo: [cicd-and-eval-gates/dataset-version-pinning.md](./) (coming) — the dataset-side companion.
- Sibling repo: [ai-architecture-reference-architecture/model-strategy/](https://github.com/jeremiahredden/ai-architecture-reference-architecture/tree/main/model-strategy) — the architecture-side model-selection framework.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/ai-gateway-pattern.md) — the runtime gateway that validates pins at call time.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — ARCH-CARE-004 is the cross-link finding this closes.
