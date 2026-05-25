# Model Registry

> **Audience.** Engineers building or refactoring the platform component that catalogs every model in production use. Tech leads tired of "wait, which model is this feature using right now?" **Scope.** The *engineering* pattern for the central model registry — schema, ownership, deployment integration, lifecycle. Pair with [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) for the release-side discipline. Not the model-selection decision (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `model-strategy/` owns that). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Models are first-class production dependencies in 2026. In any non-trivial AI system, the team uses multiple models from multiple providers: Anthropic Opus for the supervisor, Anthropic Sonnet for drafting, Anthropic Haiku for classification, OpenAI text-embedding-3-large for embeddings, Cohere Rerank-3.5 for reranking. Each model has its own version, its own deprecation timeline, its own pricing, its own BAA coverage, its own context-window limits, its own quirks.

Without a central registry, this information lives in tribal knowledge: "we use Opus for the supervisor — wait, is it Opus 4.7 or 4.6?" "We need to migrate off the deprecated embedding model — which features use it?" "Does this model have BAA coverage for clinical data?" The questions are slow to answer; the answers are sometimes wrong; the team's ability to do model lifecycle work (migrations, audits, cost analysis) is impaired.

The corrective pattern is a *central model registry*: a single source of truth for every model the platform uses. Each model is a registered artifact with version pin, owner team, allowed contexts (which features can use it), regulatory status, deprecation timeline, pricing, usage telemetry. The deployment pipeline reads from the registry; the gateway reads from the registry; cost dashboards read from the registry. Models that are not in the registry cannot be called.

The Care Coordinator's `ARCH-CARE-004` finding (model aliases instead of pinned versions) is the symptom this document closes.

This document is opinionated about three things:

1. **The registry is the only path to use a model.** The gateway (per [ai-gateway-pattern.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md)) checks every model invocation against the registry; unregistered models cannot be called. The discipline forces deliberate model adoption.
2. **Every model has an owner team.** The owner is accountable for: maintaining the version pin, tracking deprecation timelines, coordinating migrations, surfacing usage telemetry. Without an owner, the model's lifecycle is unmanaged.
3. **The registry integrates with deployment.** A release manifest pins model versions; the deploy gate validates against the registry; pinned-but-deprecated models block production deploys.

Structure: (2) the registry contract; (3) the model artifact schema; (4) ownership; (5) the registration workflow; (6) integration with deployment / gateway / circuit breakers; (7) the model lifecycle; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The registry contract

The registry is a platform-level component (database, structured file, or vendor product) that catalogs every model the platform uses.

### 2.1 What the registry stores

Per model:

- Identity (provider, model name, version pin).
- Status (registered, active, deprecated, retired).
- Owner team.
- Allowed features (which platform features can call this model).
- BAA / regulatory coverage status.
- Pricing (per-token-class).
- Context-window limit.
- Deprecation timeline (when the provider plans to deprecate, the team's migration plan).
- Usage telemetry references (links to cost dashboards, call-volume dashboards).

### 2.2 What the registry refuses

The registry enforces invariants at registration and at use:

| Registration-time refusal | What it prevents |
|---|---|
| No owner team | Unowned model |
| No version pin (alias only) | Alias drift |
| No BAA-coverage status (when required by feature class) | Regulatory accidents |
| Conflicting pricing | Cost mis-attribution |

| Use-time refusal (enforced by gateway) | What it prevents |
|---|---|
| Model not registered | Ad-hoc model adoption |
| Model status is deprecated | Use of soon-to-be-removed model |
| Feature not in allowed_features | Feature using a model it shouldn't |
| BAA-required call against non-BAA-covered model | Regulatory accident |

### 2.3 What the registry is not

- **Not the model-selection mechanism.** Selection is the architecture's job; the registry tracks what was selected.
- **Not the cost-aggregation pipeline.** Cost is computed at the gateway (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) and aggregated in the cost store; the registry links to those dashboards but does not own them.
- **Not the eval database.** Evals (per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md)) test the model's quality on the workload; the registry tracks the eval pass status (e.g., "registered as approved for clinical use after passing the clinical eval suite") but does not run the evals.

---

## 3. The model artifact schema

Each registered model is a structured artifact. The schema:

```yaml
model:
  identifier: claude-opus-4-7
  provider: anthropic
  version_pin: "2026-04-12"          # the actual API model version
  status: active                      # registered | active | deprecated | retired

  owner: ai-platform-eng

  description: |
    Anthropic's Claude Opus 4.7. Top-tier reasoning model for clinical
    decision support and supervisor roles. BAA-covered for PHI workloads.

  allowed_features:
    - care-coordinator-supervisor
    - care-coordinator-clinical-knowledge
    - analytics-copilot

  regulatory_coverage:
    baa_covered: true
    baa_signed_date: 2025-06-15
    hipaa_compatible: true
    fedramp_status: in_evaluation
    data_residency_regions: [us-east-1, us-west-2]

  pricing:
    pricing_table_ref: "anthropic_2026_05"
    input_per_million_usd: 15.00
    input_cached_per_million_usd: 1.50
    output_per_million_usd: 75.00
    last_updated: 2026-05-01

  capabilities:
    context_window_tokens: 200000
    supports_tools: true
    supports_streaming: true
    supports_prompt_caching: true
    supports_vision: true
    supports_structured_output: true

  deprecation:
    provider_announced_deprecation: null  # provider has not announced
    team_planned_deprecation: null         # team has no internal deprecation plan
    notes: ""

  registration:
    registered_by: jeremiah.redden@meridian.example
    registered_date: 2026-04-12
    registration_pr: "platform-team/models#147"

  approvals:
    security_review: {status: approved, reviewer: security-eng, date: 2026-04-13}
    eval_validation: {status: approved, eval_suite: clinical_golden_set,
                       pass_rate: 95.2, date: 2026-04-13}

  usage:
    cost_dashboard_link: "..."
    call_volume_dashboard_link: "..."
```

The schema is enforced. A registration without `owner` is rejected. A registration with `version_pin: "latest"` is rejected (aliases not allowed). A registration with `baa_covered: null` for a feature that requires BAA coverage is rejected.

### 3.1 The minimum viable schema

For teams starting out, the minimum schema:
- identifier
- provider
- version_pin
- status
- owner
- allowed_features

Other fields can be added incrementally. The minimum is what makes the registry useful from day one.

### 3.2 What the schema does NOT include

- **Per-tenant overrides.** Model-class-per-tenant routing (premium tenant gets Opus, standard gets Sonnet) is the model-routing layer's concern, not the registry's.
- **Prompt or context information.** Prompts are managed by [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md); the registry tracks models, not prompts.
- **Conversation or interaction history.** Not the registry's concern.

The boundary is "the registry tracks what models exist and their static metadata."

---

## 4. Ownership

Every model has an owner team. The team is accountable for:

- **Maintaining the version pin.** When the provider releases a new version, the owner decides whether to migrate, when, and through what process.
- **Tracking deprecation timelines.** When the provider announces deprecation, the owner is the primary point of contact for the migration project.
- **Coordinating with consuming features.** When the model changes, the owner notifies the feature teams.
- **Cost analysis.** Per-model cost trends are reviewed by the owner; cost-spike investigations route to the owner.
- **Eval-coverage assertions.** The owner ensures the model has been eval-validated for the workloads using it.

### 4.1 Ownership boundaries

- **Frontier models from major providers (Anthropic, OpenAI, Google).** Owned by the platform team (ai-platform-eng for Meridian).
- **Self-hosted open-weight models.** Owned by the platform team (the team running the inference infrastructure).
- **Fine-tuned models.** Owned by the team that did the fine-tuning (often a domain team — e.g., clinical-knowledge-engineering for a fine-tuned clinical model).
- **Specialty models (rerankers, embeddings).** Owned by the team most invested in the use case (often platform).

For Meridian, all production-frontier models are owned by ai-platform-eng. Fine-tuned models (if any) would be owned by their domain teams.

### 4.2 Orphan-model detection

Quarterly: scan the registry. Flag:
- Models whose owner is no longer on the team.
- Models that have not been used in production in >60 days.
- Models with no security or eval approval.
- Models whose pricing was last updated >6 months ago (likely stale).

The flagged models are reviewed; either re-assigned, retired, or refreshed.

---

## 5. The registration workflow

Adding a new model to the registry is a structured process. It is not "the engineer wants to try a new model so they just call it."

### 5.1 The PR

A model registration is a PR adding the YAML artifact to the registry. The PR includes:

- The artifact with all required fields.
- A justification (why is this model being added? What workload? Why not an existing registered model?).
- The eval results (the model passed the relevant eval suite for the workloads it will be used for).
- Security review (for any model touching PHI / regulated data).
- BAA confirmation (for any model touching PHI).

### 5.2 The review

- Owner team approves.
- Security-eng approves (for BAA-required or new-provider models).
- Finops approves (for material new cost line items).
- Platform team approves (for impact on the broader platform).

### 5.3 The merge

On merge:
- The artifact lands in the registry.
- The status is `registered` (not yet `active`).
- The gateway updates its cache of registered models.

### 5.4 The activation

A separate step promotes the model from `registered` to `active`:
- An initial deployment uses the model in a canary configuration.
- After validation (eval pass on canary traffic, no operational issues), the status moves to `active`.
- Production releases can now pin this model version.

The two-step process (registered → activated) prevents a registered model from being used in production before it has been validated.

### 5.5 The change workflow

Changes to an active model's artifact (pricing update, allowed-features change, regulatory status change) are PRs with appropriate review.

Changes that increment the version_pin (provider released a new version, the team is adopting it) are essentially a new registration — a new artifact for the new version, with its own approvals and activation.

---

## 6. Integration with deployment, gateway, circuit breakers

The registry is consumed by multiple platform components.

### 6.1 Deployment gate

The release manifest pins model versions per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md). The deploy gate validates each pinned model against the registry:

- The model must exist in the registry.
- The model's status must be `active` (not `deprecated`, not `retired`).
- The model must be allowed for the feature being deployed.

If any check fails, the deploy fails. This prevents production deploys that reference unregistered, deprecated, or unauthorized models.

### 6.2 Gateway enforcement

The AI gateway (per [ai-gateway-pattern.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md)) checks every call:

- The requested model must be in the registry.
- The requesting feature must be in the model's `allowed_features`.
- The call's regulatory context must match the model's coverage (a PHI-flagged call routes only to BAA-covered models).

Calls that fail any check are rejected with a structured error. This is the runtime enforcement that backs the registration discipline.

### 6.3 Cost circuit-breaker integration

The cost circuit-breaker (per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)) uses the registry's pricing data for cost computation. Pricing updates flow from the registry to the circuit-breaker without code changes.

### 6.4 LLM-call wrapper integration

The wrapper (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) records the model version on every span:

- The wrapper resolves the requested model against the registry to get the pinned version.
- The span attribute `ai.llm.model_version` is the resolved pinned version.
- If the caller passed an alias and the registry resolved to a version, `ai.llm.model_alias_resolved` is recorded.

The result: every trace has the actual model version used, not an alias.

### 6.5 Observability integration

Per-model dashboards consume from the registry's metadata + the observability stack's per-call data. The registry provides the canonical model identity; the observability stack provides the runtime measurements (call volume, cost, latency, quality SLIs per model).

---

## 7. The model lifecycle

Models move through a lifecycle. The registry tracks the state.

### 7.1 The states

```
registered → active → deprecated → retired
```

**Registered.** The model is in the registry but not yet approved for production. Used in canary, in eval, in dogfooding. Not for production releases.

**Active.** The model is approved for production. Production releases can pin this model. Active models are uncommonly removed; deprecation is the path.

**Deprecated.** The model is marked for retirement on a specified date. New production releases SHOULD not pin to deprecated models (but the deploy gate may allow it with a warning, not a refusal — to support gradual migration). Existing pinned releases continue to work.

**Retired.** The model is no longer available for use. Production releases cannot pin to retired models; existing pinned references that try to call retired models fail at the gateway.

### 7.2 The transitions

**Registered → Active.** After canary validation and eval pass. Approved by owner + security-eng.

**Active → Deprecated.** Triggered by: provider announcement of deprecation; better alternative now available; security or compliance issue; cost or capability change.

**Deprecated → Retired.** After the deprecation period (default 90 days; for major migrations, 6+ months). All consuming features must have migrated by retirement.

**Active → Retired (emergency).** Rare. Used when a model has a critical security or compliance issue and must be removed immediately. Aligned with the broader incident response.

### 7.3 The deprecation period

Deprecation periods are calibrated to the migration effort needed:

- **Provider-driven deprecation.** Aligned with the provider's deprecation timeline (typically 6 months from announcement).
- **Team-driven deprecation (new model preferred).** 90 days default.
- **Emergency deprecation (security / compliance).** Whatever is fastest while maintaining safe operation.

The deprecation date is communicated to all consuming features at the time of marking. Migration progress is tracked.

### 7.4 The retirement enforcement

When a model is retired:
- The gateway refuses calls to it (structured error: `model_retired`).
- The deploy gate refuses to deploy releases that pin to it.
- The registry's artifact remains for audit purposes (never deleted).

### 7.5 The migration project

When a model is being deprecated, the consuming features need to migrate. The migration project per consuming feature:

1. Identify the replacement model.
2. Eval the replacement against the feature's eval suite.
3. Update the prompt if needed (some prompts are model-tuned and need adaptation).
4. Stage the new model in canary.
5. Promote through the standard release flow.
6. Update the release manifest to pin the new model.
7. Deploy.

The owner team coordinates across feature teams. Migration tracking is part of the deprecation workflow.

---

## 8. Worked Meridian Health example

### 8.1 The registry contents

Meridian's model registry as of 2026-05-25:

| Model | Provider | Version | Status | Owner | Allowed features |
|---|---|---|---|---|---|
| claude-opus-4-7 | anthropic | 2026-04-12 | active | ai-platform-eng | care-coordinator-supervisor, care-coordinator-clinical-knowledge, analytics-copilot |
| claude-sonnet-4-6 | anthropic | 2025-08-15 | active | ai-platform-eng | care-coordinator-drafting, care-coordinator-summarization, analytics-copilot |
| claude-haiku-4-5 | anthropic | 2025-10-01 | active | ai-platform-eng | care-coordinator-classifier, care-coordinator-query-rewriter, patient-api-classification, patient-api-answer |
| claude-opus-4-6 | anthropic | 2025-12-01 | deprecated (retiring 2026-08-01) | ai-platform-eng | (migration in progress for 2 features) |
| text-embedding-3-large | openai | 2024-01-25 | active | ai-platform-eng | (used by retrieval-corpus-ingestion pipeline) |
| cohere-rerank-3.5 | cohere | 2025-09-01 | active | ai-platform-eng | care-coordinator-clinical-retrieval |
| gpt-5 | openai | 2026-01-10 | registered | ai-platform-eng | (canary only; not active for production) |

The registry tells the team at a glance: 5 models active, 1 deprecated and migrating, 1 in canary. Each model's BAA coverage, pricing, and allowed-features are recorded.

### 8.2 The registry implementation

`meridian.models.registry` is a Python package + a Postgres `models` table. The package is the typed interface; the table is the persistence.

The registry is loaded on application startup (or refreshed periodically) into in-memory data structures the gateway and other components consume.

### 8.3 The registration workflow in practice

When the team wants to add a new model (e.g., gpt-5 in 2026-Q1):

1. **PR opened.** YAML artifact for gpt-5. Justification: "Evaluating as primary-tier alternative to Opus for the analytics-copilot."
2. **Eval results attached.** gpt-5 on the analytics eval suite: 92% pass rate (vs Opus baseline 94%). Acceptable for the canary phase.
3. **Security review.** OpenAI's BAA coverage was confirmed; data-residency was verified; security-eng approved.
4. **Finops review.** Pricing was added; cost projection was reviewed.
5. **PR merged.** Status: `registered`. Allowed-features: empty (not yet enabled for any feature).
6. **Canary phase.** gpt-5 was used in canary against analytics-copilot for 4 weeks. Eval and operational metrics tracked.
7. **Activation decision.** The team decided not to activate gpt-5 for production (the small quality gap vs Opus was meaningful for the analytics workload). Status remains `registered`; the model is available for further evaluation but not for production routes.

This pattern — register, canary, then explicitly choose to activate or not — prevents the slow accretion of "we tried this model once and then it stayed in production for years."

### 8.4 The deprecation workflow in practice

In 2026-04-01, Anthropic announced Claude Opus 4.6 deprecation effective 2026-09-01. The team's response:

1. **Registry update.** Opus 4.6 status changed to `deprecated`; deprecation timeline set to 2026-08-01 (one month before provider deprecation, for safety margin).
2. **Migration plan.** Identified consuming features: care-coordinator-supervisor (already on Opus 4.7), care-coordinator-clinical-knowledge (already on Opus 4.7). Only legacy / preview features were on 4.6.
3. **Migration execution.** The legacy features were updated to pin 4.7. Eval-validated. Released. By 2026-05-01, no production code pinned 4.6.
4. **Retirement.** On 2026-08-01, Opus 4.6 status moved to `retired`. The artifact remains in the registry for audit.

The discipline worked. The deprecation was handled as a structured project rather than a fire drill.

### 8.5 The cost dashboard integration

Per-model cost dashboards consume from the registry (model identity) + the cost-aggregation pipeline (per-call cost from gateway). The dashboard:

- Per-model daily cost trend.
- Per-model per-feature cost attribution.
- Per-model usage volume.

When cost-spike alerts fire (per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)), the registry provides the metadata to correlate (which model? which features use it? who is the owner?).

### 8.6 The platform discipline

- `meridian.models.registry` is the only path to use a model. Lint rule against hardcoded model identifiers in calling code (calling code references models by registry name, not by raw provider+model string).
- Every PR adding or changing a model registration goes through the review per section 5.
- Quarterly registry review per section 4.2.
- Provider deprecation announcements are tracked in the team's calendar; team reviews quarterly for upcoming actions.

---

## 9. Anti-patterns

### 9.1 "Model aliases everywhere"

Application code references `claude-opus-latest`; the provider's alias resolution shifts the underlying model silently. The team only learns about the shift when behavior changes.

**Corrective.** Pin full version strings. The registry refuses alias registrations.

### 9.2 "Registry exists but bypassable"

The registry is in place, but application code can call the provider SDKs directly. The registry's enforcement is partial.

**Corrective.** Gateway is the only path to providers (per [ai-gateway-pattern.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md)); the gateway consults the registry on every call.

### 9.3 "No owner team per model"

Models are in the registry but ownership is "the platform." When a deprecation lands or a cost incident happens, no one is the responsible point of contact.

**Corrective.** Specific owner team per model. Recorded in the artifact. Lint check.

### 9.4 "Deprecation period is implicit"

Models are marked deprecated but the retirement date is not set. Deprecated models stay in production indefinitely.

**Corrective.** Deprecation has an explicit retirement date. Calendar-driven. Enforced by retirement automation.

### 9.5 "Registry is a wiki page"

The 'registry' is documentation. The deploy pipeline does not consult it; the gateway does not enforce it.

**Corrective.** Registry is code-readable (database or version-controlled file). Enforcement at deploy and at gateway.

### 9.6 "Frontier-tier proliferation"

The team has Opus 4.5, 4.6, 4.7, and 4.8 all active in the registry simultaneously. Each is used by a different feature. Migration coordination is impossible because the team is always somewhere in a migration.

**Corrective.** Limit active versions per provider; force migration coordination so older versions deprecate as new ones activate.

### 9.7 "Pricing stale"

Pricing in the registry was set at adoption time and never updated. Cost computations are based on stale prices.

**Corrective.** Pricing-update PRs on every provider announcement; monthly reconciliation against invoices catches drift.

### 9.8 "Eval validation skipped"

A model is registered and activated without the eval-validation step. Quality on production workloads is unverified.

**Corrective.** Eval pass is required for activation. Registry artifact includes the eval-validation status and the eval-suite reference.

---

## 10. Findings (sprint-assignable)

### ML-REG-001 — Severity: Critical
**Finding.** No central model registry exists; models in use are tracked in tribal knowledge.
**Recommendation.** Build the registry per section 2; populate with all production models.
**Owner.** ai-platform-eng, sprint N+1.

### ML-REG-002 — Severity: Critical
**Finding.** Model aliases (e.g., `claude-opus-latest`) are used in production; version drift is silent.
**Recommendation.** Pin full versions per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md); registry refuses aliases.
**Owner.** ai-platform-eng, sprint N+1.

### ML-REG-003 — Severity: Critical
**Finding.** Application code can call provider SDKs directly, bypassing any registry enforcement.
**Recommendation.** AI gateway (per [ai-gateway-pattern.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md)) as the only path; lint rule against direct SDK imports.
**Owner.** ai-platform-eng, sprint N+1.

### ML-REG-004 — Severity: High
**Finding.** Models do not have owner teams; lifecycle work has no responsible party.
**Recommendation.** Owner field in registry; recorded for every model; orphan detection quarterly.
**Owner.** ai-platform-eng team lead, sprint N+2.

### ML-REG-005 — Severity: High
**Finding.** BAA coverage status is not tracked per model; calls to non-BAA-covered models with PHI context have happened.
**Recommendation.** BAA status in registry artifact; gateway enforces; PHI-flagged calls route only to BAA-covered models.
**Owner.** ai-platform-eng + security-eng + compliance, sprint N+1.

### ML-REG-006 — Severity: High
**Finding.** Allowed-features list is not tracked per model; features can use any registered model without authorization.
**Recommendation.** Allowed-features in registry; gateway enforces; registration changes are reviewed.
**Owner.** ai-platform-eng, sprint N+2.

### ML-REG-007 — Severity: High
**Finding.** Deprecation tracking is informal; provider deprecation announcements get lost.
**Recommendation.** Deprecation timeline in registry artifact; tracked in team calendar; quarterly review.
**Owner.** ai-platform-eng team lead, sprint N+2.

### ML-REG-008 — Severity: High
**Finding.** Deploy gate does not validate model pins against the registry; deprecated or retired models can be deployed.
**Recommendation.** Deploy gate consults the registry per section 6.1; refuses non-active models in production releases.
**Owner.** ai-platform-eng + sre, sprint N+2.

### ML-REG-009 — Severity: High
**Finding.** Pricing is hardcoded; provider price changes require code deploys.
**Recommendation.** Pricing in registry; pricing-update PRs; monthly invoice reconciliation per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md).
**Owner.** ai-platform-eng + finops, sprint N+2.

### ML-REG-010 — Severity: High
**Finding.** Eval-validation is not recorded per model registration; models can be activated without quality verification.
**Recommendation.** Eval status in registry; activation requires passed eval; quarterly re-validation.
**Owner.** ai-platform-eng, sprint N+3.

### ML-REG-011 — Severity: Medium
**Finding.** Two-step registration (registered → active) is collapsed; models go straight to production.
**Recommendation.** Two-step per section 5; canary phase before activation.
**Owner.** ai-platform-eng, sprint N+3.

### ML-REG-012 — Severity: Medium
**Finding.** Multiple active versions of the same model proliferate; the team is always in some migration.
**Recommendation.** Limit active versions per provider (typically 1, sometimes 2 during a migration); coordinate migrations to keep this bounded.
**Owner.** ai-platform-eng team lead, sprint N+3.

### ML-REG-013 — Severity: Medium
**Finding.** Provider deprecation announcements are tracked manually; missed announcements delay migration planning.
**Recommendation.** Subscribe to provider deprecation announcements; track in registry; quarterly calibration.
**Owner.** ai-platform-eng, sprint N+3.

### ML-REG-014 — Severity: Medium
**Finding.** Cost dashboards consume from a separate source from the registry; the dashboard's model identity does not match the registry's.
**Recommendation.** Dashboards consume registry as the canonical model identity; cost data joins on it.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ML-REG-015 — Severity: Medium
**Finding.** Self-hosted open-weight models (if any) are tracked differently from API-hosted models; the registry pattern is inconsistent.
**Recommendation.** Single registry pattern for all models, regardless of hosting; deployment-specific fields are accommodated in the schema.
**Owner.** ai-platform-eng, sprint N+4.

### ML-REG-016 — Severity: Medium
**Finding.** Fine-tuned models (if any) do not have a clear ownership; lineage from base model is undocumented.
**Recommendation.** Fine-tuned models registered with base-model link; owned by the team that did the fine-tuning.
**Owner.** ai-platform-eng + domain teams, sprint N+4.

### ML-REG-017 — Severity: Low
**Finding.** Registry artifact does not include context-window limit; calls that exceed the limit fail at the provider rather than at the platform.
**Recommendation.** Context-window in registry; gateway pre-validates input size; clearer error for context-overflow.
**Owner.** ai-platform-eng, sprint N+5.

### ML-REG-018 — Severity: Low
**Finding.** Quarterly registry review is not scheduled.
**Recommendation.** Calendar quarterly review; ai-platform-eng team lead + finops + security-eng.
**Owner.** ai-platform-eng team lead, sprint N+4.

---

## 11. Adoption sequencing checklist

For a team without a model registry:

- [ ] **Sprint 0 — inventory.** Catalog every model in production use. For each: provider, version, what calls it.
- [ ] **Sprint 1 — registry scaffold.** Build the minimum viable registry (section 3.1 schema). Populate with the production models.
- [ ] **Sprint 1 — owner assignment.** Every model has a named owner.
- [ ] **Sprint 1 — version pinning.** Replace aliases with full version strings. Registry refuses aliases.
- [ ] **Sprint 2 — gateway enforcement.** AI gateway consults registry; unregistered models rejected.
- [ ] **Sprint 2 — deploy gate.** Release manifest consults registry; non-active models rejected.
- [ ] **Sprint 3 — BAA / regulatory coverage.** Track coverage status; enforce for regulated calls.
- [ ] **Sprint 3 — allowed-features.** Track per-model authorized features; enforce.
- [ ] **Sprint 4 — deprecation tracking.** Calendar-driven; provider announcement subscriptions; quarterly review.
- [ ] **Sprint 4 — two-step registration.** Registered → canary → active workflow.
- [ ] **Sprint 5 — dashboards.** Per-model cost / volume / quality dashboards consume from registry.
- [ ] **Ongoing — quarterly review.** Owner currency, deprecation timelines, pricing freshness, eval validation status.

A team that completes this sequence has the model-as-dependency discipline that turns "wait, which model is this" into a queryable fact and makes model lifecycle work tractable.

---

## 12. References

- MLflow Model Registry — the canonical pattern for model versioning and registry in the ML space; the AI registry is similar in spirit, different in details (LLM models are vendor-hosted artifacts, not team-trained models).
- Software dependency management practices (npm registry, pip / PyPI, Maven Central) — the broader pattern of versioned dependencies that the model registry is one shape of.
- Anthropic, OpenAI, Google provider documentation on model deprecation policies.
- HIPAA Business Associate Agreement requirements — the regulatory frame for BAA coverage tracking.
- This repo: [model-lifecycle/model-promotion.md](./) (coming) — the dev → staging → prod promotion path that the registry's two-step pattern supports.
- This repo: [model-lifecycle/model-deprecation-playbook.md](./) (coming) — the migration project pattern for provider-driven deprecations.
- This repo: [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — the release-side companion.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — the wrapper that records the registry-resolved model version on every call.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — consumes registry pricing for cost calculation.
- Sibling repo: [ai-architecture-reference-architecture/model-strategy/model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/) (coming) — the architecture-side framework for what to put in the registry.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/ai-gateway-pattern.md) — the gateway that enforces registry membership at call time.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — ARCH-CARE-004 is the cross-link finding.
