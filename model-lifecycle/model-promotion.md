# Model Promotion

> **Audience.** Engineers whose model deployments are "engineer SSHes and updates a config." Tech leads whose new model version made it to prod without anyone running the eval suite. Anyone whose dev / staging / prod story for models is "they all use whatever model is in env vars." **Scope.** The *engineering* practice of model promotion: the dev → staging → prod path; the eval gate that blocks promotion on regression; per-environment configuration discipline; model-version-as-release-artifact pattern. Not the registry primitive (see [model-registry.md](./model-registry.md)). Not the migration playbook (see [model-deprecation-playbook.md](./model-deprecation-playbook.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Code has dev → staging → prod. Models don't, typically.

Pattern:

- Code is built; tests pass; promoted to staging; tested; promoted to prod.
- Model: someone changes the config; deploys.

The mismatch: code goes through gates; models don't. Quality regressions slip through.

The discipline:

- Models go through dev → staging → prod.
- Eval gate at each promotion.
- Per-environment configuration.
- Model is a release artifact.

This document covers the engineering practice.

This document is opinionated about four things:

1. **Models are first-class release artifacts.** Treat them like code; gate them like code.
2. **Eval gate blocks promotion on regression.** No "we'll fix it in prod"; pre-promotion verification.
3. **Per-environment configuration is explicit.** Dev/staging/prod each have their own model config.
4. **Promotion is a project, not an inline change.** Schedule; ownership; rollback.

Structure: (2) the promotion flow; (3) per-environment configuration; (4) the eval gate; (5) staging discipline; (6) production promotion; (7) release artifacts; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The promotion flow

The pipeline.

### 2.1 The standard flow

```
Dev (engineering iteration)
  ↓
Staging (full eval + integration)
  ↓
Production (canary → ramp → full)
```

Three environments.

### 2.2 Per-environment role

**Dev.** Engineering iteration. Quick changes; broken things expected.

**Staging.** Pre-production verification. Matches production architecture.

**Production.** Live traffic.

### 2.3 What "promotion" means

Promotion = moving from one environment to next:

- Dev → staging.
- Staging → production.

Each transition: a gate.

### 2.4 The gates

Per gate:

- **Eval gate.** Does the model meet quality SLO?
- **Cost gate.** Is the cost projection acceptable?
- **Latency gate.** Does it meet latency SLO?
- **Security review.** Compliance / safety checks.

Pass all → promote.

### 2.5 The auto vs manual promotion

- **Auto.** Each gate auto-runs; passes auto-promote.
- **Manual.** Gates run; humans approve.

Per organization. For high-stakes: manual review for prod.

### 2.6 The rollback path

Each promotion has rollback:

- Revert to previous version.
- Tested.
- Pre-defined criteria for triggering.

Cross-link to [rollback-procedures.md](./rollback-procedures.md).

### 2.7 The "we have no formal flow" reality

Many teams:

- Models updated ad-hoc.
- No environments.
- "We test in production."

Build the flow.

### 2.8 The flow cost

- Setting up environments: weeks.
- Eval suite for each: weeks.
- Promotion pipeline: weeks.
- Ongoing maintenance.

Substantial investment; high value.

### 2.9 The "we have a flow but skip steps" failure

When promotion is rushed:

- Skip staging.
- Direct to prod.

First production incident is the lesson.

---

## 3. Per-environment configuration

Each environment has its own model config.

### 3.1 The configuration shape

```yaml
care-coordinator-feature:
  dev:
    model: anthropic:claude-sonnet:4-6
    prompt_version: latest  # or specific
    dataset: care-coordinator-eval-v8.2.0
    rate_limits: relaxed
  
  staging:
    model: anthropic:claude-sonnet:4-6
    prompt_version: v2.3.0
    dataset: care-coordinator-eval-v8.2.0
    rate_limits: production-like
  
  production:
    model: anthropic:claude-sonnet:4-6
    prompt_version: v2.3.0
    dataset: care-coordinator-eval-v8.2.0
    rate_limits: production
```

Per environment.

### 3.2 The version pinning per environment

Each environment pins:

- Model version.
- Prompt version.
- Dataset version.

Pinned; explicit.

### 3.3 The "dev uses latest; staging uses specific" pattern

For iteration:

- Dev: tracks latest (or specific dev version).
- Staging: pinned to candidate for promotion.
- Production: pinned to current production version.

Each environment serves a purpose.

### 3.4 The config separation

Environments are separate:

- Separate config files (or namespaces).
- Separate deployments.
- Separate credentials.

Drift between environments is detectable.

### 3.5 The "we share dev API key with prod" risk

Sharing credentials:

- Dev tests can affect prod.
- Cost confusion.

Per-environment credentials.

### 3.6 The "we use prod model in dev" exception

For some workflows:

- Dev testing against prod model.
- Validates production behavior locally.

Acceptable for limited testing; not full dev workflow.

### 3.7 The config as code

Configuration in version control:

- Per-environment YAML.
- PR-reviewed.
- Versioned.

Reproducible.

### 3.8 The config drift

If environments drift:

- Dev has different model than staging.
- Bug "passes in dev" but fails in staging.

Drift detection.

### 3.9 The synchronization

When promoting:

- Staging config copied to production.
- Verified.
- Deployed.

Pipeline mechanic.

---

## 4. The eval gate

The pre-promotion verification.

### 4.1 The eval suite

Per workload:

- Curated eval set (cross-link to [data-engineering-for-ai/training-eval-split-discipline.md](../data-engineering-for-ai/training-eval-split-discipline.md)).
- Versioned.
- Test cases representative.

### 4.2 The pre-promotion eval

Before staging → prod:

- Run eval against candidate config.
- Compare to current production performance.
- Threshold: must meet or exceed.

### 4.3 The threshold determination

Per workload:

- Quality SLO (e.g., 95% pass rate).
- Per-category breakdown.
- No regression beyond N% on critical categories.

Pre-defined.

### 4.4 The gate decision

Eval results → decision:

- All pass thresholds → green-light promotion.
- Some fail → block; investigate.
- Borderline → manual review.

Decision documented.

### 4.5 The eval-result report

Per pre-promotion:

```yaml
promotion_eval:
  candidate: care-coordinator-release-v14.2.1
  current_production: v14.2.0
  
  eval_results:
    overall_pass_rate: 96.4% (vs 96.2% current; +0.2)
    
  per_category:
    clinical_workflows: 96.1% (vs 95.8%; +0.3)
    edge_cases: 92.0% (vs 93.5%; -1.5) ⚠️
  
  threshold_check:
    overall: pass (≥ 95% target)
    per_category: edge_cases below threshold; manual review required
  
  recommendation: HOLD; investigate edge_cases regression
```

Structured.

### 4.6 The "eval passed; deploy" workflow

If pass:

- Auto-promote (or manual approval).
- Documented.

### 4.7 The "eval failed; what now" workflow

If fail:

- Don't promote.
- Investigate.
- Iterate.
- Re-eval.

### 4.8 The eval-gate enforcement

In CI:

- Promotion blocked if gate fails.
- No bypass.

Engineering discipline.

### 4.9 The "we bypassed the gate" warning

If urgent deployment needed:

- Document exception.
- Track risk.
- Time-bound (e.g., 24h hot-fix; re-eval next sprint).

Exceptional; not routine.

### 4.10 The gate metrics

Per quarter:

- Promotions attempted.
- Gate passes / fails.
- Hot-fix bypasses.

Track.

---

## 5. Staging discipline

The pre-production environment.

### 5.1 The staging role

Staging:

- Pre-production validation.
- Final integration testing.
- Eval gate.

The last stop before production.

### 5.2 The staging-prod parity

Staging should match production:

- Same model versions.
- Same prompt versions.
- Same data sources (or staging mirrors).
- Same infrastructure (or close).

Differences cause "passes in staging; fails in prod."

### 5.3 The staging traffic

Sources:

- Synthetic traffic (replay of production).
- Internal test users.
- Some production traffic (shadow).

Per workload.

### 5.4 The staging quality verification

In staging:

- Run eval suite.
- Run integration tests.
- Sample-check responses.
- Manual review (for high-stakes).

Pre-production verification.

### 5.5 The "we don't have staging" reality

For some teams:

- Dev → prod (no staging).
- Higher risk.

Building staging is engineering investment.

### 5.6 The staging duration

How long candidate stays in staging:

- Minimal (hours): quick validation.
- Extended (days-week): observe behavior.

Per-workload.

### 5.7 The "staging passed; prod failed" diagnostic

If staging passed but production fails:

- Staging-prod parity insufficient.
- Traffic patterns differ.
- Re-investigate.

### 5.8 The staging cost

Staging infrastructure:

- ~30-50% of production cost (if mirror).
- ~10-20% (if scaled-down).

Per-workload.

### 5.9 The "we don't run eval in staging" gap

If staging doesn't include eval:

- Eval only happens in dev.
- Dev-staging differences could cause surprises.

Eval in staging.

---

## 6. Production promotion

The final step.

### 6.1 The production-promotion mechanics

```
Staging passes →
  Production deploy (canary 5%) →
    Monitor 24h →
      Ramp to 25% →
        Monitor →
          Ramp to 100% →
            Verify
```

Cross-link to [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md).

### 6.2 The canary monitoring

During canary:

- Quality metrics (online eval).
- Cost metrics.
- Latency metrics.
- Customer complaints.

Per-metric thresholds.

### 6.3 The auto-rollback

If canary metrics breach:

- Auto-rollback.
- Alert engineering.

Pre-defined rules.

### 6.4 The ramp schedule

```
Day 0: 5% canary
Day 1: 25%
Day 3: 50%
Day 5: 100%
```

Gradual.

### 6.5 The "we go from 5% to 100%" skip

Rushed:

- Skip intermediate ramps.
- More risk.

Standard ramp.

### 6.6 The production-monitoring duration

Post-100%:

- Monitor 1-2 weeks.
- Verify stable.

Final acceptance.

### 6.7 The "promotion succeeded" milestone

After production stable:

- Decommission previous version.
- Mark in catalog.

Cross-link to [model-registry.md](./model-registry.md).

### 6.8 The release-notes

Production release:

- Documented in release-notes.
- Customer-visible (if applicable).

Communication.

### 6.9 The post-promotion review

After promotion:

- Did it go smoothly?
- Lessons learned.

Process improvement.

---

## 7. Release artifacts

The model-version-as-release.

### 7.1 The release-artifact concept

A release pins:

- Model version.
- Prompt version.
- Dataset version.
- Eval suite version.
- Code version.

All artifacts; all change together.

Cross-link to [data-engineering-for-ai/dataset-versioning.md §6](../data-engineering-for-ai/dataset-versioning.md).

### 7.2 The release-manifest

Per release:

```yaml
release: care-coordinator-v14.2.1
date: 2026-05-15
artifacts:
  code: git_sha_abc123
  model: anthropic:claude-sonnet:4-6
  prompt: care-coordinator-v2.3.0
  dataset: care-coordinator-eval-v8.2.0
  eval_suite: care-coordinator-eval-suite-v1.8.0
```

Complete.

### 7.3 The release-immutability

Once released:

- Manifest immutable.
- Specific artifacts archived.
- Reproducible.

### 7.4 The cross-release diff

Compare releases:

- v14.2.0 → v14.2.1: what changed.
- Code changes; model changes; prompt changes.

Visibility into release-by-release evolution.

### 7.5 The release-history

Per workload:

- Catalog of all releases.
- When each was deployed.
- When each was deprecated.

Auditable.

### 7.6 The release-naming

Conventional:

- Major.Minor.Patch.
- Or date-based.

Per organization.

### 7.7 The "we just update the model in prod" anti-pattern

Without release artifacts:

- Hard to know what's running.
- Rollback unclear.
- Audit gaps.

**Corrective.** Release-as-artifact per §7.1.

### 7.8 The release coordination across features

Multiple features:

- Each has its own release cycle.
- Coordination meeting (monthly).
- Aligned schedules where dependencies exist.

### 7.9 The release-as-deployment-trigger

Production promotion:

- Triggered by release publish.
- Deployment pipeline executes.

Automation.

---

## 8. Worked Meridian example

Meridian's promotion practice.

### 8.1 The environments

```
Dev: 
  - Engineering iteration.
  - Models: latest or specific candidate.
  - Traffic: synthetic.

Staging:
  - Pre-production validation.
  - Models: candidate for promotion (pinned).
  - Traffic: synthetic + 1% shadow from production.
  - Same architecture as production.

Production:
  - Pinned versions.
  - Full customer traffic.
  - Multi-region.
```

Three environments.

### 8.2 The promotion gates

```yaml
gates:
  dev_to_staging:
    - eval suite pass (basic threshold)
    - no critical bugs
    
  staging_to_production:
    - eval suite pass (full threshold)
    - cost projection within budget
    - latency SLO met
    - manual sign-off (engineering lead)
    
  production_canary_to_full:
    - canary metrics within thresholds
    - no customer complaints
    - 24h+ at each canary level
```

Per gate.

### 8.3 The Q1 2026 promotion

A Care Coordinator prompt update:

- Day 1: developed in dev.
- Day 3: passed dev eval; promoted to staging.
- Day 4-7: staging validation; full eval pass.
- Day 8: 5% canary in production.
- Day 9: ramp to 25%.
- Day 11: ramp to 50%.
- Day 13: ramp to 100%.

Total: 13 days dev to full prod.

Smooth.

### 8.4 The Q1 prompt-bloat-incident (failed promotion)

(Cross-link to [cost-incident-runbook.md §9](../cost-and-finops/cost-incident-runbook.md).)

A prompt change passed dev:

- Dev eval passed.
- Staging eval passed (cost wasn't measured).
- Production canary: cost-per-call doubled.

Result:

- Rollback at canary.
- Investigation: cost gate missing in staging.

Lesson:

- Cost gate added to staging.
- Pre-deployment cost projection.

### 8.5 The release manifest

```yaml
care-coordinator-release-v14.2.1:
  date: 2026-05-15
  artifacts:
    code: git_sha_abc123
    model: anthropic:claude-sonnet:4-6
    prompt: care-coordinator-v2.3.0
    dataset: care-coordinator-eval-v8.2.0
    eval_suite: care-coordinator-eval-suite-v1.8.0
  promoted_via:
    - dev: 2026-05-01
    - staging: 2026-05-05
    - production_canary: 2026-05-12
    - production_full: 2026-05-15
```

Per release; auditable.

### 8.6 The "we have a release catalog" benefit

For investigations:

- "What was deployed on May 10?"
- Answer: look up release manifest.

Fast.

### 8.7 The deprecation process

Old releases:

- After 30 days at 100%: previous version deprecated.
- 60 days later: archived.
- 90 days later: removed.

Lifecycle.

### 8.8 The release coordination

Monthly:

- Release planning meeting.
- Each feature's upcoming releases.
- Dependencies coordinated.

### 8.9 The infrastructure cost

- Staging environment: ~30% of production cost.
- Engineering for promotion pipeline: 4 weeks initial; ongoing ~5%.

Total: ~$5k/month + initial.

### 8.10 The benefits

- 0 promotion regressions caught only in production in last 12 months.
- Multiple regressions caught in staging.
- Audit-ready.

### 8.11 The lessons

- Gates catch issues.
- Staging-prod parity is essential.
- Release-as-artifact enables auditing.
- Promotion is a discipline; not a process.

---

## 9. Anti-patterns

### 9.1 The "deploy directly to prod" rush

**Pattern.** No staging; no gate; direct to prod. Quality risk.

**Corrective.** Full flow per §2.

### 9.2 The "we bypass the gate when in a rush" pattern

**Pattern.** Eval gate skipped; quality regressions ship.

**Corrective.** Strict gate enforcement per §4.8.

### 9.3 The "staging is broken; we test in prod" reality

**Pattern.** Staging not maintained; teams skip.

**Corrective.** Staging parity per §5.2.

### 9.4 The unversioned release

**Pattern.** Production updates; no manifest; what's deployed unclear.

**Corrective.** Release manifest per §7.2.

### 9.5 The mixed environment configs

**Pattern.** Dev uses one model; prod another; surprise discrepancies.

**Corrective.** Explicit per-environment config per §3.

### 9.6 The "we don't measure cost in staging" gap

**Pattern.** Cost projection unverified before prod; surprises.

**Corrective.** Cost gate per §8.4.

### 9.7 The "all environments share credentials" laxity

**Pattern.** Dev affects prod cost; cost confusion.

**Corrective.** Per-environment credentials per §3.5.

### 9.8 The skip-canary rush

**Pattern.** 0% → 100% direct; high risk.

**Corrective.** Gradual ramp per §6.4.

### 9.9 The post-promotion review absent

**Pattern.** Promotion happens; no review; lessons lost.

**Corrective.** Per-promotion review per §6.9.

### 9.10 The release manifest in someone's head

**Pattern.** What's deployed: ask engineer.

**Corrective.** Manifest in source of truth per §7.5.

---

## 10. Findings (sprint-assignable)

### ML-MP-001 — Severity: Critical
**Finding.** No formal promotion flow.
**Recommendation.** Per §2.
**Owner.** AI platform + engineering management, sprint N+1.

### ML-MP-002 — Severity: Critical
**Finding.** Eval gate absent.
**Recommendation.** Per §4.
**Owner.** AI platform + eval, sprint N+1.

### ML-MP-003 — Severity: Critical
**Finding.** Staging environment absent.
**Recommendation.** Per §5.
**Owner.** AI platform, sprint N+1.

### ML-MP-004 — Severity: High
**Finding.** Per-environment configuration not explicit.
**Recommendation.** Per §3.
**Owner.** AI platform, sprint N+2.

### ML-MP-005 — Severity: High
**Finding.** Cost gate absent in staging.
**Recommendation.** Per §8.4.
**Owner.** AI platform + FinOps, sprint N+2.

### ML-MP-006 — Severity: High
**Finding.** Release manifest absent.
**Recommendation.** Per §7.2.
**Owner.** AI platform, sprint N+2.

### ML-MP-007 — Severity: High
**Finding.** Auto-rollback during canary absent.
**Recommendation.** Per §6.3.
**Owner.** AI platform, sprint N+2.

### ML-MP-008 — Severity: High
**Finding.** Canary ramp schedule ad-hoc.
**Recommendation.** Per §6.4.
**Owner.** AI platform, sprint N+2.

### ML-MP-009 — Severity: Medium
**Finding.** Staging-prod parity gap.
**Recommendation.** Per §5.2.
**Owner.** AI platform, sprint N+3.

### ML-MP-010 — Severity: Medium
**Finding.** Per-environment credentials missing.
**Recommendation.** Per §3.5.
**Owner.** AI platform + security, sprint N+3.

### ML-MP-011 — Severity: Medium
**Finding.** Promotion-pipeline bypass mechanism absent.
**Recommendation.** Per §4.9 (emergency exception process).
**Owner.** AI platform, sprint N+3.

### ML-MP-012 — Severity: Medium
**Finding.** Post-promotion review absent.
**Recommendation.** Per §6.9.
**Owner.** AI platform, sprint N+3.

### ML-MP-013 — Severity: Medium
**Finding.** Release-deprecation lifecycle not defined.
**Recommendation.** Per §8.7.
**Owner.** AI platform, sprint N+3.

### ML-MP-014 — Severity: Medium
**Finding.** Cross-team release coordination absent.
**Recommendation.** Per §8.8.
**Owner.** engineering management, sprint N+4.

### ML-MP-015 — Severity: Low
**Finding.** Promotion metrics not tracked.
**Recommendation.** Per §4.10.
**Owner.** SRE + AI platform, sprint N+5.

### ML-MP-016 — Severity: Low
**Finding.** Release-history catalog absent.
**Recommendation.** Per §7.5.
**Owner.** AI platform, sprint N+5.

### ML-MP-017 — Severity: Low
**Finding.** Synthetic traffic in staging absent.
**Recommendation.** Per §5.3.
**Owner.** AI platform, sprint N+6.

### ML-MP-018 — Severity: Low
**Finding.** Customer-facing release notes absent.
**Recommendation.** Per §6.8.
**Owner.** product, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Set up dev / staging / prod environments per §2.**
- [ ] **Per-environment config per §3.**
- [ ] **Define eval gate per §4.**
- [ ] **Build release-manifest mechanism per §7.2.**
- [ ] **Implement canary deployment per §6.**
- [ ] **Implement auto-rollback per §6.3.**
- [ ] **Define ramp schedule per §6.4.**
- [ ] **Per-environment credentials per §3.5.**
- [ ] **Post-promotion review per §6.9.**
- [ ] **Monthly release coordination per §8.8.**

---

## 12. References

**In this folder.**
- [model-registry.md](./model-registry.md) — catalog.
- [fine-tuning-operations.md](./fine-tuning-operations.md) — fine-tunes need promotion.
- [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) — rollout mechanics.
- [rollback-procedures.md](./rollback-procedures.md) — rollback.
- [model-deprecation-playbook.md](./model-deprecation-playbook.md) — deprecation.

**Elsewhere in this repo.**
- [data-engineering-for-ai/dataset-versioning.md](../data-engineering-for-ai/dataset-versioning.md) — dataset versioning.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — eval gate design.
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost gate.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — migration discipline.
- [ai-architecture-reference-architecture / context-and-prompt-architecture / prompt-as-api-discipline.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/context-and-prompt-architecture/prompt-as-api-discipline.md) — prompt versioning.

**External.**
- Continuous delivery literature (Humble, Farley).
- Google SRE Book — release engineering.
- Feature-flag platforms documentation.
