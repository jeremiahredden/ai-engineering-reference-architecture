# Prompt Versioning

> **Audience.** Engineers who have completed the prompts-as-code refactor (or are doing it now) and need the versioning discipline to ride on top. Tech leads tired of "which version was running when this incident happened." **Scope.** The *engineering* practice of semantic versioning for prompts — version-bump semantics, pinning in releases, backwards-compatible evolution, deprecation lifecycle, rollback. Pair with [prompts-as-code-discipline.md](./prompts-as-code-discipline.md) for the artifact pattern this builds on. Not the prompt-writing technique (vendor docs cover that). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Once prompts are in a versioned store (per [prompts-as-code-discipline.md](./prompts-as-code-discipline.md)), the next discipline is *what counts as a version, when to bump, how to deprecate*. The patterns are familiar from software versioning, with some adaptations for prompts:

- Multiple versions of the same prompt routinely coexist (a canary rollout means 5% of traffic on v2.4.2 and 95% on v2.4.1).
- The "API surface" of a prompt is its output shape — consumers (parsers, downstream extractors, eval suites) depend on it.
- The cost of a "breaking change" is meaningful but not catastrophic — a prompt that breaks downstream parsers can be rolled back in minutes, unlike an HTTP API that breaks a thousand customer integrations.

So this document adapts the semver discipline to prompts: major version bumps for breaking changes, minor for additive, patches for non-behavioral. It documents the pinning pattern (every release pins prompt versions), the deprecation lifecycle (give consumers time before removal), and the rollback pattern (always available).

This document is opinionated about three things:

1. **Semver applies to prompts.** Major / minor / patch with adapted semantics. The discipline is the same as for any other versioned artifact.
2. **Releases pin prompt versions.** Every code release pins the prompt versions it expects; mismatch fails deployment; rollback restores the pinned set.
3. **Old versions stick around.** Removing an old prompt version is a deliberate operation, not garbage collection. Rollback paths must remain available.

Structure: (2) what a prompt version is; (3) semver semantics for prompts; (4) pinning prompts in releases; (5) backwards-compatible evolution; (6) the deprecation lifecycle; (7) the rollback path; (8) multi-version coexistence (canary, A/B, rollback); (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. What a prompt version is

A prompt version is the unique identifier for a specific configuration of a prompt artifact at a point in time. It includes:

- **The content** (the prompt text).
- **The metadata** (variables, depends_on, eval_suite_refs from §4.1 of the prompts-as-code document).
- **The semver string** (e.g., `2.4.1`).

Two prompts with the same name but different versions are distinct artifacts. The application can reference either by name + version; routing logic decides which version is served per request.

### 2.1 The version-string format

`{major}.{minor}.{patch}` is the default. Some teams add a pre-release suffix for in-development versions (`2.5.0-rc1`). Avoid date-based versioning (`2026-05-25`) — date strings hide the breaking-change shape.

### 2.2 The version-string scope

The version is per-prompt-name. Two prompts (`care_coordinator_supervisor` and `care_coordinator_classifier`) have independent version sequences. The supervisor at `2.4.1` is unrelated to the classifier at `2.4.1`.

The prompt-store-wide schema version (the version of the *artifact format* itself) is separate. Schema changes (a new field added to the artifact structure) are managed independently.

### 2.3 The version-string lifecycle

A version exists in the prompt store. Its status moves through:

- `proposed`: in PR review, not yet merged.
- `staged`: merged, awaiting deploy.
- `active`: deployed, can be referenced in production calls.
- `deprecated`: still callable but marked for removal; new references should not be added.
- `retired`: removed; calls referencing it fail.

The status is metadata on the artifact, queryable.

---

## 3. Semver semantics for prompts

The standard semver rules adapted to prompts:

### 3.1 Major version bump (X.0.0)

A breaking change. Consumers that depend on the prompt's output shape will break without coordination.

Examples of major changes:
- The output format changes (used to return Markdown bullets; now returns a JSON object).
- A previously-required variable is removed.
- A previously-supported behavior is removed (the prompt used to handle a certain class of input; now it refuses).
- The output's structured shape changes (used to include a `confidence` field; now does not).

Major bumps require:
- Explicit deprecation notice (§6).
- Migration path for downstream consumers.
- Coordinated release with downstream changes.

### 3.2 Minor version bump (X.Y.0)

Additive change. Backward-compatible. Existing consumers keep working without modification.

Examples of minor changes:
- A new optional variable is added (existing callers continue to omit it).
- A new behavior is added without changing existing behavior (the prompt now handles a previously-unhandled class of input).
- A new output field is added (existing parsers ignore unknown fields).
- An optional reasoning step is added that does not change the final output shape.

Minor bumps require:
- Eval-gate pass.
- Reviewer approval.
- Optional: canary rollout.

### 3.3 Patch version bump (X.Y.Z)

Non-behavioral change. The prompt's behavior is meant to be unchanged; the change is about cost, latency, clarity, robustness.

Examples of patch changes:
- Rewording for clarity that does not change behavior.
- Adding a chain-of-thought instruction that does not change the final answer's content.
- Restructuring the prompt's internal organization without changing what it does.
- Fixing a typo.

Patch bumps require:
- Eval-gate pass (to confirm behavior is in fact unchanged).
- Reviewer approval.

### 3.4 The "no behavior change" assertion

Patch bumps assert no behavior change. The eval gate is the test — if a patch bump fails eval (because behavior did change), the change is actually a minor or major bump, and the bump should be reclassified.

This is the discipline that keeps the semver meaningful. Without it, every change becomes a patch bump and consumers lose the signal.

### 3.5 The minor-vs-major edge cases

Some changes are ambiguous. Patterns:

- **Adding refusal behavior on previously-handled inputs.** This is a major bump — consumers who depended on the prompt handling those inputs will see the new refusal.
- **Removing refusal behavior (the prompt now handles inputs it used to refuse).** Minor bump — additive capability.
- **Tightening the prompt's tone or formatting.** Often minor; can be major if a downstream parser depended on the old formatting.
- **Changing the prompt's intended model** (designed for Opus-class, now designed for Sonnet-class). Major bump — the prompt may behave differently on the new tier.

When in doubt, bump major. The cost of an unnecessary major bump (extra coordination) is much less than the cost of a missed major bump (consumers break in production).

---

## 4. Pinning prompts in releases

Every code release pins the prompt versions it expects. This is the discipline that makes "the release artifact is reproducible" a true claim.

### 4.1 The release manifest

A release artifact includes a manifest:

```yaml
release:
  version: 2026.05.25-r3
  code_sha: 9c2a1f8b...
  prompts:
    care_coordinator_supervisor: 2.4.1
    care_coordinator_classifier: 1.2.0
    care_coordinator_clinical_knowledge: 1.8.0
    care_coordinator_drafting: 3.0.2
    care_coordinator_query_rewriter: 1.1.0
  models:
    supervisor_tier: claude-opus-4-7@2026-04-12
    clinical_tier: claude-opus-4-7@2026-04-12
    drafting_tier: claude-sonnet-4-6@2025-08-15
    classifier_tier: claude-haiku-4-5@2025-10-01
    rewriter_tier: claude-haiku-4-5@2025-10-01
  datasets:
    clinical_guidelines_corpus: 2026-Q2-1
    drug_interaction_graph: 2026-05-01
    eval_clinical_golden_set: 2026-05-15
```

The release manifest is the reproducibility contract. Given this manifest, the team can re-create the exact production behavior of this release.

### 4.2 The deployment gate

The deploy pipeline checks the manifest against the prompt store:

- Every pinned prompt version must exist in the store.
- Every pinned model version must be registered in the model registry (per [model-registry.md](../model-lifecycle/model-registry.md)).
- Every pinned dataset version must exist.

If any pin fails, the deploy fails. The fail-fast pattern prevents partial-state deployments where a release expects a prompt version that does not exist.

### 4.3 The pinning discipline at the application level

Application code references prompts by name only at design time; the version comes from the release manifest at runtime:

```python
# At deploy time, MANIFEST is loaded and made available
prompt_artifact = prompt_store.get(
    name="care_coordinator_supervisor",
    version=MANIFEST.prompts["care_coordinator_supervisor"],
)
```

The application does not hardcode `version="2.4.1"`; the release manifest does. This means a new release that updates the manifest also updates which version the application reads.

### 4.4 The rollback contract

Because the release manifest is the pinning contract, a rollback restores the entire pinned set. Rolling back code without rolling back the prompt version (or vice versa) produces a hybrid state that the team did not test.

The rollback runbook: restore the previous release manifest; the deployment system reads the prior set of pins; production reverts to the pre-deploy state.

---

## 5. Backwards-compatible evolution

The discipline that makes the most prompt changes minor bumps rather than major: design prompt evolution to be backwards-compatible where practical.

### 5.1 Additive output fields

When adding a new field to a structured output, add it as optional. Downstream parsers that do not know about the new field ignore it; parsers that do can use it. The change is minor.

The anti-pattern: adding a field that downstream parsers will fail to parse (strict schema validators that error on unknown fields). The corrective: design parsers to ignore unknown fields.

### 5.2 Additive variables

When adding a new variable, add it as optional with a sensible default. Callers that do not provide it get the default behavior; callers that do can specify it. The change is minor.

### 5.3 Behavior toggles

When changing a behavior, add a toggle that defaults to the old behavior; new callers opt in to the new behavior. The toggle becomes the default in a later major bump after consumers have migrated.

This is the same feature-flag pattern used for code changes, applied to prompt behaviors.

### 5.4 Deprecation-then-removal

When removing a behavior or a variable, first deprecate (the behavior still works, but the artifact marks it deprecated and downstream consumers see a warning). After the deprecation period, remove with a major bump.

The deprecation period gives consumers time to migrate without coordination on a single release.

### 5.5 The non-design-for-back-compat case

Some changes cannot reasonably be made backwards-compatible. A complete restructuring of the prompt's output format. A change in intended use case. A change in clinical reasoning that produces materially different recommendations.

For these: major bump, coordinated migration, explicit deprecation of the prior version.

---

## 6. The deprecation lifecycle

Prompt versions deprecate; the lifecycle is structured.

### 6.1 Deprecation triggers

- A new major version of the prompt has stabilized.
- The prompt's underlying assumption has changed (a referenced tool removed, a referenced retriever deprecated).
- The prompt has not been used in production for >60 days.
- The prompt's intended model has been deprecated.
- A security or compliance issue has been identified in the prompt.

### 6.2 The lifecycle

```
active → deprecated → retired
```

**Active → deprecated.**
- The artifact's `status` field is updated to `deprecated`.
- The `deprecation` field is populated with: deprecation date, planned retirement date, recommended replacement version.
- The prompt is still callable; calls produce a warning in the trace.
- Downstream consumers are notified (typically: an automated email + a tracking ticket).

**Deprecated → retired.**
- After the deprecation period (Meridian default: 90 days; high-stakes prompts: 180 days).
- The artifact's `status` field is updated to `retired`.
- Calls to the retired version fail with a structured error.
- The artifact remains in the store (for historical / audit purposes); it is not deletable.

### 6.3 Deprecation period

The deprecation period is calibrated to give consumers time to migrate. Factors:

- **Number of consumers.** More consumers = longer period.
- **Type of change.** Major refactors need more migration time than additive deprecations.
- **Regulatory requirements.** Some changes need longer windows for change management.
- **Operational pace of consumers.** Teams that ship weekly can migrate faster than teams that ship quarterly.

The default 90 days fits most cases. The minimum is 30 days (rarely justifiable). The maximum is 365 days (for highly stable prompts with many consumers).

### 6.4 Emergency retirement

For security or compliance issues that require immediate removal:

- The version is retired immediately (calls fail).
- Affected consumers are notified out-of-band (paged, not just emailed).
- An immediate-replacement version is shipped.
- The incident is documented.

Emergency retirement is the exception. Frequent emergency retirements suggest the deprecation discipline is failing somewhere upstream.

---

## 7. The rollback path

Rollback is a routine operation, not an emergency one. The discipline:

### 7.1 Always-available rollback target

The previous prompt version (the one before the current production version) is always available in the store with `active` status. Rolling back is changing the manifest pin from the current version to the previous version.

The previous-previous version is also kept active for a defined window (Meridian default: 14 days after the current version becomes default). This provides a two-step rollback if the immediate previous is also problematic.

### 7.2 Rollback runbook

When a production prompt issue is identified:

1. Confirm: which prompt version is implicated? Which version is the rollback target?
2. The rollback target's eval suite is verified green (it was, when that version was current).
3. The release manifest is updated to pin the prior version.
4. Deploy the updated manifest. Production reads the new pin within the cache-refresh window (Meridian: 60 seconds).
5. Verify production behavior recovered.
6. Document the incident and the rollback.

The rollback is typically completed within 5 minutes of identification.

### 7.3 Rollback-by-routing (canary backout)

For canary rollouts where the new version is at 5% traffic and shows a regression, the rollback is even simpler — the routing layer's traffic split is updated from 5%/95% to 0%/100%. No new deploy; no manifest change. The next routing-cache refresh restores the previous version at 100%.

This pattern is the operational backbone of safe prompt rollouts.

### 7.4 The "no rollback path" anti-pattern

A team that allows old versions to be deleted, or that does not retain previous-version status, has no rollback target. Production issues become extended outages.

The discipline: old versions persist. Storage cost is negligible; rollback option is invaluable.

---

## 8. Multi-version coexistence

A mature prompt-versioning practice routinely has multiple versions of the same prompt active in production at once. Three patterns:

### 8.1 Canary rollout

The new version is deployed alongside the previous version. Routing layer sends a small percentage of traffic to the new version. After a verification window, traffic ramps up or rolls back.

The Meridian Care Coordinator's standard canary: 5% for 4 hours, then 25% for 4 hours, then 100% if quality SLIs (judge-pass-rate on the canary subset) remain green.

### 8.2 A/B testing

Two versions of a prompt are tested against each other to determine which is better on a specific metric (quality, cost, latency, user feedback). Traffic is split 50/50; statistical analysis determines a winner over a defined window.

A/B tests are less common than canary rollouts because most prompt changes have a clear better direction (the new version was designed to improve something specific). A/B is right when the trade-off is genuinely uncertain.

### 8.3 Long-tail support

For prompts with downstream consumers on different update cadences, the latest version coexists with older versions for an extended period. Consumers reference the version they have tested against; they migrate on their own schedule (until the version is deprecated and retired).

This is more common in multi-team platforms where prompts are consumed by other teams' code. For single-team platforms, the canary pattern is usually sufficient.

### 8.4 The routing layer

Multi-version coexistence requires a routing layer that can decide which version a given request goes to. The routing decision can be based on:

- Random sampling (canary, A/B).
- Tenant ID (some tenants on the new version first).
- Feature flag (admin-controlled).
- Request metadata (some request classes on the new version).

The routing layer's state is configuration, not code. Routing changes do not require deploys.

---

## 9. Worked Meridian Health example

### 9.1 The version landscape

Meridian's prompt store as of 2026-05-25:

| Prompt | Active versions | Deprecated versions | Retired versions |
|---|---|---|---|
| `care_coordinator_supervisor` | 2.4.1 (current), 2.4.0 (rollback target) | 2.3.0 (deprecated 2026-04-01, retiring 2026-07-01) | 2.2.x (retired 2026-03-15) |
| `care_coordinator_clinical_knowledge` | 1.8.0 (current), 1.7.2 (rollback target) | 1.7.0 (deprecated 2026-04-15) | 1.6.x |
| `care_coordinator_drafting` | 3.0.2 (current, recent major bump), 2.9.5 (still active, awaiting tenant migration) | 2.8.0 (deprecated) | |

The drafting prompt's 3.0.2 is a recent major bump (output format changed); 2.9.5 is still active because some downstream parsers have not migrated yet — the deprecation period is in effect.

### 9.2 A canary rollout in detail

Supervisor 2.4.2 was being prepared in 2026-05-20. The canary:

- **Day 1, 09:00.** 2.4.2 deployed. Routing layer set to 0% → 5% traffic on 2.4.2.
- **Day 1, 13:00.** First 4-hour window completes. Canary judge-pass-rate: 94.2% (baseline: 94.0%). Within tolerance. Ramp to 25%.
- **Day 1, 17:00.** Second 4-hour window completes. Canary judge-pass-rate: 93.8%. Within tolerance. Ramp to 100%.
- **Day 1, 17:00 onward.** 2.4.2 is the default. 2.4.1 status: active (rollback target). 2.4.0 status: active (secondary rollback target).
- **Day 14.** 2.4.0 status moves to deprecated (deprecation 2026-06-03, retiring 2026-09-01). 2.4.1 stays as active rollback target.

The discipline kept the previous version available for rollback throughout. No prompt version was deleted; the prior versions exist in the store.

### 9.3 An actual rollback

In 2026-04-08, a supervisor change (`2.3.5 → 2.3.6`) introduced the cost regression described in the cost circuit-breaker doc. The team identified the regression at ~17:30; rollback was complete by 17:35:

- 17:30: cost-spike alert fires.
- 17:31: on-call investigates; identifies the prompt version 2.3.6 as the cause (correlated with deploy time).
- 17:32: release manifest is updated to pin 2.3.5 again.
- 17:33: deploy of the updated manifest.
- 17:34: cache refresh; production calls use 2.3.5.
- 17:35: cost normalizes.

Total rollback time: 5 minutes from alert to recovery. The previous version had been kept available; the manifest update was the only change.

### 9.4 A deprecation

The drafting 3.0 major bump (in 2026-Q1) changed the output format from a JSON object with `body` and `disclaimer` fields to a structured object with `body`, `disclaimer`, `recommended_action`, `escalation_flag`. Downstream parsers needed to be updated.

- **2026-02-01.** Drafting 3.0.0 released. Drafting 2.9.x marked deprecated with deprecation_date=2026-02-01, retirement_date=2026-08-01 (6-month period for the major bump).
- **2026-02-01 onward.** Downstream parsers notified (automated email + ticket per consumer). Migration tracking dashboard set up.
- **2026-02 through 2026-07.** Consumers migrate at their own pace. The dashboard shows migration progress.
- **2026-07-15.** Two consumers still on 2.9.x. Reminders sent. Engineering escalation.
- **2026-07-31.** One consumer still on 2.9.x. Direct ticket. Hard deadline.
- **2026-08-01.** 2.9.x retired. Calls to 2.9.x fail. The last consumer had completed migration on 2026-07-30.

The deprecation period gave consumers time. The discipline of the retirement date being firm forced migration.

### 9.5 The release manifest at work

Every Meridian Care Coordinator release publishes the manifest. The most recent (release 2026.05.25-r3) is in section 4.1 above.

The deploy gate verifies every pinned prompt version exists; releases that reference a version that has been retired fail. This caught one near-miss in 2025-Q4: a release was prepared with a prompt version that had been retired three weeks prior; deploy refused; the release was corrected.

### 9.6 The platform discipline

- `meridian-prompts` store has retention forever (no garbage collection of old versions).
- Every release manifest is committed and queryable.
- Deprecation tracking dashboard surfaces upcoming retirements.
- Rollback runbook is rehearsed quarterly.
- Quarterly review of deprecation period calibration.

---

## 10. Anti-patterns

### 10.1 "Patch bumps for everything"

Every change is called a patch bump because "we didn't intend behavior change." Consumers lose the semver signal; major changes ship without coordination.

**Corrective.** The eval gate is the test for "no behavior change." If eval moves, the bump is at least minor. If the output shape moves, the bump is major.

### 10.2 "No deprecation period — versions removed when no longer current"

A new version becomes current; the old version is deleted. Consumers who had not yet adopted the new version break. Rollback is impossible.

**Corrective.** Old versions persist for at least the deprecation period (90 days default). Deletion is replaced with retirement; the artifact stays in the store.

### 10.3 "Release manifest doesn't pin prompts"

Releases pin code SHAs but not prompt versions. Prompt deploys are independent of code deploys; rollback of a code release does not rollback the prompts; reproducibility is partial.

**Corrective.** Release manifest pins everything: code SHA, prompt versions, model versions, dataset versions. Rollback restores the entire pinned set.

### 10.4 "Hardcoded version strings in application code"

Application code references `version="2.4.1"` directly. Updating to a new version requires a code change.

**Corrective.** Version comes from the release manifest at runtime; application code references prompts by name only.

### 10.5 "Canary without rollback automation"

Canary rollouts go out at 5%, but rolling back requires a manual code change. Regressions identified during canary take hours to revert.

**Corrective.** Routing-layer-based canary; rollback is a configuration change (traffic split → 0%/100%); cache refresh restores within seconds.

### 10.6 "Deprecation period is theatrical"

A version is marked deprecated but the retirement date is never enforced; deprecated versions stay deprecated forever; the next deprecation has no credibility.

**Corrective.** Deprecation periods have firm retirement dates. Retirement is automated where possible. Calendar-driven.

### 10.7 "No previous-version rollback target"

The team's prompt store has only the current version; the previous version was deleted after the new one stabilized. When the new version has a regression, there is no rollback target.

**Corrective.** The previous version persists with active status. Rollback is always available.

### 10.8 "Major bumps without consumer notification"

A major-bumped prompt is released; downstream consumers find out by their code breaking in production.

**Corrective.** Major bumps trigger automated notification to documented consumers (depends_on metadata). Migration tracking. Deprecation period.

---

## 11. Findings (sprint-assignable)

### PROMPT-VER-001 — Severity: High
**Finding.** Prompts have no explicit version strings; "the current prompt" is the only version reference.
**Recommendation.** Adopt semver per section 3; pin in artifact metadata; CI checks for valid version on every PR.
**Owner.** ai-platform-eng, sprint N+1.

### PROMPT-VER-002 — Severity: High
**Finding.** Release manifest does not pin prompt versions; prompt deploys are decoupled from code deploys without explicit pinning contract.
**Recommendation.** Release manifest pins prompt versions per section 4; deploy gate enforces existence.
**Owner.** ai-platform-eng + sre, sprint N+1.

### PROMPT-VER-003 — Severity: High
**Finding.** Rollback target is not available — previous versions are deleted when superseded.
**Recommendation.** Previous versions stay active per section 7.1; rollback is always available.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-VER-004 — Severity: High
**Finding.** Deprecation lifecycle is undefined; old versions either stay active forever or are deleted abruptly.
**Recommendation.** Adopt the active → deprecated → retired lifecycle per section 6; default deprecation period 90 days.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-VER-005 — Severity: High
**Finding.** Application code hardcodes prompt versions; new versions require code changes.
**Recommendation.** Version comes from release manifest; application code references prompts by name only.
**Owner.** ai-platform-eng, sprint N+2.

### PROMPT-VER-006 — Severity: High
**Finding.** Major bumps are not distinguished from minor bumps; consumers cannot tell which prompt changes are breaking.
**Recommendation.** Semver discipline per section 3; PR checklist requires bump justification.
**Owner.** ai-platform-eng + prompt-engineering, sprint N+2.

### PROMPT-VER-007 — Severity: High
**Finding.** Canary rollouts require code deploys to roll back; backout latency is hours.
**Recommendation.** Routing-layer-based canary per section 8.1; rollback via traffic-split configuration; <5 minute backout.
**Owner.** ai-platform-eng + sre, sprint N+3.

### PROMPT-VER-008 — Severity: Medium
**Finding.** Multi-version coexistence has no consumer-tracking; deprecation notices go to a generic distribution.
**Recommendation.** Per-prompt depends_on metadata documents consumers; deprecation notices target them.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-VER-009 — Severity: Medium
**Finding.** Deprecation tracking has no dashboard; engineering team does not know which deprecations are approaching retirement.
**Recommendation.** Deprecation dashboard per section 9.6; surface upcoming retirements 30 days out.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### PROMPT-VER-010 — Severity: Medium
**Finding.** Rollback runbook is undocumented; on-call response to prompt-version incidents is ad-hoc.
**Recommendation.** Rollback runbook per section 7.2; rehearse quarterly.
**Owner.** ai-platform-eng + sre, sprint N+3.

### PROMPT-VER-011 — Severity: Medium
**Finding.** Semver discipline is inconsistent; some teams bump major freely while others avoid major bumps and accumulate change.
**Recommendation.** Platform-wide semver conventions documented; PR review enforces.
**Owner.** ai-platform-eng team lead, sprint N+3.

### PROMPT-VER-012 — Severity: Medium
**Finding.** Backwards-compatible evolution patterns (optional fields, behavior toggles) are not in the team's standard practice.
**Recommendation.** Document patterns per section 5; train the team; the discipline reduces major-bump frequency.
**Owner.** ai-platform-eng, sprint N+3.

### PROMPT-VER-013 — Severity: Medium
**Finding.** Emergency retirement procedure (for security / compliance issues) is undefined.
**Recommendation.** Document per section 6.4; integrate with the security incident response process.
**Owner.** ai-platform-eng + security-eng, sprint N+4.

### PROMPT-VER-014 — Severity: Medium
**Finding.** Old versions accumulate in the store without status updates; deprecated versions are confused with active ones.
**Recommendation.** Status field on every artifact; automated lifecycle transitions where possible; quarterly cleanup audit.
**Owner.** ai-platform-eng, sprint N+4.

### PROMPT-VER-015 — Severity: Low
**Finding.** Release manifests are not surfaced to downstream consumers; consumers do not know which prompt versions are in production right now.
**Recommendation.** Manifest publication endpoint; consumers can query the current pinned set per prompt.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-VER-016 — Severity: Low
**Finding.** Version-string format is inconsistent across prompts; some use semver, some use date-based, some use sequential integers.
**Recommendation.** Standardize on semver; migrate non-conforming prompts.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-VER-017 — Severity: Low
**Finding.** Multi-tenant prompt variants (per-tenant overlays) do not have a coherent versioning story.
**Recommendation.** Per-tenant overlays version independently; the base prompt's version is part of the overlay's dependency.
**Owner.** ai-platform-eng, sprint N+5.

### PROMPT-VER-018 — Severity: Low
**Finding.** Prompt-version dashboards do not show per-version traffic share during canary rollouts.
**Recommendation.** Add per-version traffic visualization to the prompt operations dashboard.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team with prompts in a store (per [prompts-as-code-discipline.md](./prompts-as-code-discipline.md)) but without version discipline:

- [ ] **Sprint 0 — convention.** Adopt semver per section 3. Document the bump semantics.
- [ ] **Sprint 1 — version metadata.** Add explicit version strings to all artifacts. CI validates.
- [ ] **Sprint 1 — release manifest.** Include prompts in the release manifest; deploy gate enforces existence.
- [ ] **Sprint 2 — runtime resolution.** Application code reads version from manifest, not hardcoded.
- [ ] **Sprint 2 — rollback path.** Previous versions persist; rollback is documented.
- [ ] **Sprint 3 — deprecation lifecycle.** active / deprecated / retired status; default 90-day period; tracking dashboard.
- [ ] **Sprint 3 — canary infrastructure.** Routing-layer-based canary; backout via configuration.
- [ ] **Sprint 4 — consumer notification.** Depends_on metadata; automated deprecation notices to known consumers.
- [ ] **Sprint 4 — runbook.** Rollback procedure; quarterly rehearsal.
- [ ] **Sprint 5 — A/B and long-tail support.** Multi-version coexistence patterns for the advanced cases.
- [ ] **Ongoing — discipline.** Quarterly review of deprecation calibration; orphan-prompt scans; manifest audit.

A team that completes this sequence has the prompt-versioning discipline that turns "which version was running" into a queryable fact and makes rollback a routine operation rather than an emergency.

---

## 13. References

- Semantic Versioning 2.0 (semver.org) — the source convention adapted here.
- Schema migration patterns (Liquibase, Flyway) — analogous discipline for database schemas.
- API versioning practices (REST, GraphQL) — analogous discipline for service APIs.
- This repo: [prompt-engineering/prompts-as-code-discipline.md](./prompts-as-code-discipline.md) — the prerequisite artifact pattern.
- This repo: [cicd-and-eval-gates/prompt-version-pinning.md](../cicd-and-eval-gates/) (coming) — the deploy-side pinning practice.
- This repo: [cicd-and-eval-gates/release-artifacts-for-ai.md](../cicd-and-eval-gates/) (coming) — the broader release-artifact pattern.
- This repo: [model-lifecycle/model-version-pinning.md](../model-lifecycle/) (coming-but-this-batch: [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md)) — the analogous pattern for model versions.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — the eval-gate that makes version semantics meaningful.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — the wrapper that records prompt version on every call.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — ARCH-CARE-005 is the cross-link finding.
