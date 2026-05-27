# Model Deprecation Playbook

> **Audience.** Engineers facing a vendor's model deprecation announcement. Tech leads whose roadmap has "migrate off Claude 3" in Q3. Anyone whose first vendor sunset notice just arrived. **Scope.** The *engineering* playbook for migrating off a vendor-sunset model: typical 6-month timeline; parallel-running with candidate replacement; eval cross-check; prompt-port discipline; rollout sequence; rollback criteria. Includes worked Meridian example of migrating Care Coordinator. Not the broader model strategy (see [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md)). Not the canary mechanics (see [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

When a vendor announces sunset of a model:

- Typical lead time: 6-12 months.
- Migration is forced (model becomes unavailable).
- Work falls on consumer team.

Without a playbook:

- Migration improvised.
- Quality regressions.
- Late panic.

With a playbook:

- Structured timeline.
- Phases of work.
- Risk managed.

This document is opinionated about four things:

1. **Treat as a project, not an inline change.** Schedule; phases; deliverables.
2. **Parallel running is essential.** Validate replacement before forced switch.
3. **Test against eval; not just hope.** Real measurement.
4. **Don't wait until 30 days before deadline.** Start within weeks of announcement.

Structure: (2) the timeline; (3) phase 1: assessment; (4) phase 2: candidate selection; (5) phase 3: parallel running; (6) phase 4: rollout; (7) phase 5: deprecation; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The timeline

The phases.

### 2.1 The typical timeline

```
Vendor announces sunset: T=0 (180 days notice)

Phase 1: Assessment (0-2 weeks)
Phase 2: Candidate selection (2-6 weeks)
Phase 3: Parallel running (6-16 weeks)
Phase 4: Rollout (16-22 weeks)
Phase 5: Deprecation (22-26 weeks)

Old model sunset: T+26 weeks (or vendor-announced date).
```

6 months total.

### 2.2 The "we have less time" reality

If vendor announces 30-day notice:

- Compressed timeline.
- Higher risk.

Push back; or accept compressed work.

### 2.3 The buffer

Aim to complete migration 4+ weeks before vendor sunset:

- Buffer for unexpected.
- Customer communication.
- Rollback if needed.

### 2.4 The "we have longer" comfortable

12-month notice:

- More time.
- Don't waste it.

Start anyway.

### 2.5 The "deferred until the deadline" trap

Common:

- 6 months feels long.
- Project deferred.
- 30 days remaining; panic.

Don't defer.

### 2.6 The dependency on vendor's communication

Vendor's announcement quality varies:

- Some: clear timeline + reasons.
- Some: terse "will be removed."

Follow up; clarify.

### 2.7 The cross-team impact

Migration affects:

- AI platform.
- Product.
- Customer success.
- Compliance.

Coordinate.

### 2.8 The "we'll re-evaluate vendor choice" opportunity

Forced migration:

- Re-evaluate.
- Could move to different vendor.
- Or stay with same vendor's new model.

Strategic decision.

### 2.9 The migration cost

For one migration:

- Engineering: 4-8 weeks.
- Eval: weeks.
- Operations: ongoing.

Substantial.

---

## 3. Phase 1: Assessment

The first weeks.

### 3.1 The inventory

- Which workloads use the deprecated model?
- Per workload: volume, criticality.

Per consumer:

- Code references.
- Configurations.
- Catalog entries.

Cross-link to [model-registry.md](./model-registry.md).

### 3.2 The impact assessment

Per workload:

- Volume.
- Customer-facing or internal.
- Cost.
- Quality requirements.

Risk ranking.

### 3.3 The vendor-replacement clarity

What does vendor recommend:

- Newer version of same model family.
- Different model.

Read vendor's migration guide.

### 3.4 The "vendor recommends X but X may not fit our workload" pushback

Vendor's recommendation:

- Not always right for the workload.
- Eval per workload.

Don't blindly accept.

### 3.5 The cross-team stakeholder identification

Per affected workload:

- Engineering lead.
- Product owner.
- Customer success.
- SME (clinical, legal, etc.).

Coordinate.

### 3.6 The communication plan

For affected customers:

- When to inform.
- What to say.
- Timeline.

Cross-link to [§7.8](#78-the-customer-facing-communication).

### 3.7 The output

Phase 1 deliverable:

- Migration project plan.
- Affected workload list.
- Replacement candidate(s).
- Timeline.
- Owners.

Per project management.

### 3.8 The "we have N workloads to migrate; not one" complexity

For platforms with many features:

- Prioritize per criticality.
- Migrate sequentially or in parallel.

### 3.9 The "we don't know which workloads use it" gap

Without registry:

- Audit code.
- Audit configurations.

Discover.

---

## 4. Phase 2: Candidate selection

Choosing the replacement.

### 4.1 The candidate options

Typical:

- Newer version of same model family (most common).
- Different model from same vendor.
- Different vendor.

Per workload.

### 4.2 The evaluation

Per candidate:

- Quality eval (vs deprecated model).
- Cost analysis.
- Latency.
- Compliance.

Multi-dimensional.

### 4.3 The eval methodology

For each workload:

- Run eval suite against deprecated.
- Run eval suite against candidate.
- Compare per category.

Cross-link to [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md).

### 4.4 The "candidate matches deprecated" baseline

If candidate ≥ deprecated:

- Acceptable.
- Proceed.

If candidate < deprecated:

- Investigate.
- Maybe different candidate.
- Or accept degradation.

### 4.5 The "we need to port the prompt for the candidate" consideration

New model may behave differently:

- Prompt may need tweaking.
- Cross-link to [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md §6](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md).

Port; not literal copy.

### 4.6 The cost comparison

Per call:

- Deprecated cost.
- Candidate cost.

Different may favor migration; some may not.

### 4.7 The compliance check

For regulated workloads:

- Candidate BAA / DPA / FedRAMP status.
- Match deprecated's or better.

Per workload.

### 4.8 The "deprecated had unique capability" concern

If deprecated had specific strength:

- Verify candidate has it.

Per workload.

### 4.9 The candidate decision

Per workload:

- Document choice.
- Rationale.

Per project plan.

### 4.10 The output

Phase 2 deliverable:

- Candidate per workload.
- Eval comparison.
- Cost analysis.
- Prompt-port plan.

---

## 5. Phase 3: Parallel running

The validation.

### 5.1 The parallel-run design

```
Production: deprecated model (current).
Parallel: candidate model.

Method 1: shadow traffic (cross-link to canary-and-shadow-rollout §3).
Method 2: A/B (some users on candidate; cross-link).

Both produce comparison data.
```

### 5.2 The shadow-first

Common:

- Shadow 1-2 weeks.
- Offline comparison.

Then if good: canary.

### 5.3 The comparison-metrics

- Quality (eval; live judge).
- Cost.
- Latency.
- User feedback.

Comparison report.

### 5.4 The "candidate is worse" finding

If candidate inferior:

- Investigate (prompt-port issue? Different fundamentally?).
- Iterate.
- Or: choose different candidate.

### 5.5 The "candidate is better" excitement

If candidate superior:

- Migration is opportunity.
- Proceed confidently.

### 5.6 The "candidate is mostly equal; some regressions" complexity

Most common:

- Overall similar.
- Per-category regressions.

Decide per regression severity.

### 5.7 The customer-pilot

For some workloads:

- Pilot tenant (or selected users).
- Migrated to candidate first.
- Feedback collected.

Before broader rollout.

### 5.8 The "we'll wait and see longer" caution

Don't over-extend parallel:

- Costs money (shadow).
- Delays migration.

Determine clearly; move.

### 5.9 The "we found candidate has different output style" surprise

New model may have stylistic differences:

- Slightly more verbose.
- Different phrasings.

Investigate; decide; document.

### 5.10 The output

Phase 3 deliverable:

- Comparison report.
- Recommendation: proceed / iterate / choose alternative.

---

## 6. Phase 4: Rollout

The actual migration.

### 6.1 The canary rollout

Per [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md):

- 5% canary.
- Ramp to 25%, 50%, 100%.

Standard.

### 6.2 The per-workload sequence

Multiple workloads:

- Order matters.
- Less critical first.
- Critical last.

Per priority.

### 6.3 The customer-tier sequence

Per tier:

- Free tier first.
- Standard.
- Premium last (highest stakes).

Tiered rollout.

### 6.4 The geographic sequence

Per region:

- One region first.
- Then others.

### 6.5 The rollback option

During rollout:

- Rollback to deprecated.
- But: deprecated is itself sunset.

So:

- Rollback feasible only while deprecated still available.
- After vendor sunset: no rollback.

Window.

### 6.6 The "we kept the candidate-LRU pattern" tooling

Maintain both versions:

- Catalog entries for both.
- Pinning per release.

Until sunset complete.

### 6.7 The "we have multiple migrations going" coordination

Multiple workloads migrating:

- Track each.
- Avoid simultaneous high-risk.

### 6.8 The eval-during-rollout

Per ramp step:

- Re-eval.
- Compare to baseline.
- Decision: continue ramp or hold.

Cross-link to [model-promotion.md](./model-promotion.md).

### 6.9 The output

Phase 4 deliverable:

- 100% on candidate model.
- Verified.

---

## 7. Phase 5: Deprecation

The wind-down.

### 7.1 The deprecated-version retirement

After 100% on candidate:

- Deprecated catalog entry: deprecated state.
- Pin removed from production.
- Catalog kept (for history).

### 7.2 The "vendor's deadline" trigger

Vendor's sunset date:

- Service no longer available.
- Hopefully you're already off.

If not: emergency.

### 7.3 The rollback-window expiration

If issues found after vendor sunset:

- Can't rollback to deprecated (unavailable).
- Hot-patch or different fallback.

Risk of failed migration.

### 7.4 The "we're not 100% migrated by sunset" emergency

If migration not done:

- Try to accelerate.
- Or accept service interruption.

Push back on vendor (rarely successful).

### 7.5 The customer-communication

Post-migration:

- Inform customers (where applicable).
- Reassurance.
- Document.

### 7.6 The post-migration eval

Confirm:

- Production performing on candidate.
- Quality / cost / latency stable.

Pre-vendor-sunset.

### 7.7 The lessons-learned documentation

Per migration:

- What went well.
- What didn't.
- Patterns for next migration.

Cross-link to [§7.8](#78-the-customer-facing-communication).

### 7.8 The customer-facing communication

For customer-visible:

- Status page entries.
- Email notifications.
- Release notes.

Honest; informative.

### 7.9 The output

Phase 5 deliverable:

- Migration complete.
- Deprecated retired.
- Lessons documented.

---

## 8. Worked Meridian example

Meridian's migration practice.

### 8.1 The Q1 2026 Sonnet 4.5 → 4.6 (proactive migration)

Anthropic announced Sonnet 4.6 (not a deprecation; an upgrade):

```
T=0: 4.6 announced.
T+1 week: assessment.
T+3 weeks: eval done; 4.6 better in most categories.
T+5 weeks: parallel running started.
T+7 weeks: canary 5%.
T+9 weeks: 25%.
T+11 weeks: 100%.
T+13 weeks: complete.
```

13 weeks; smooth.

### 8.2 The hypothetical Sonnet 3 sunset migration

(Hypothetical; for illustration.)

If Sonnet 3 deprecated:

```
T=0: announcement (180 days notice).
T+2 weeks: assessment done.
T+6 weeks: candidate selected (Sonnet 4.5 or 4.6).
T+16 weeks: parallel running done; eval complete.
T+22 weeks: 100% on candidate.
T+26 weeks: deprecation final.
```

Within timeline.

### 8.3 The cross-team coordination

For the hypothetical migration:

- AI platform: migration leadership.
- Product: feature impact assessment.
- Customer success: communication.
- Compliance: BAA verification.

Coordinated.

### 8.4 The eval-driven decision

Candidate evaluation:

- Sonnet 4.5: 95% pass rate.
- Sonnet 4.6: 96% pass rate.
- Sonnet 4 (assuming exists): 92% pass rate.

Sonnet 4.6 chosen.

### 8.5 The parallel-run cost

For Care Coordinator:

- 6-week parallel run.
- Shadow traffic: 100k calls.
- Cost: ~$2k.

Worth it for confidence.

### 8.6 The customer notification

For external customers:

```
Subject: Model Update Coming

To: [customers]
Our AI features will be migrating to a newer model version
between [date1] and [date2]. Expected user experience: 
similar to today; possibly slightly improved.

If you observe unexpected changes, please report to support.
```

Standard.

### 8.7 The post-migration eval

After 100%:

- 4 weeks observation.
- Quality stable.
- Cost stable.
- Customer feedback positive.

Migration successful.

### 8.8 The "no rollback after sunset" risk

For hypothetical migration:

- Sonnet 3 unavailable after T+26 weeks.
- After: no rollback to old model.

Accept; or have fallback (e.g., Sonnet 4.5).

### 8.9 The infrastructure cost

- Engineering: ~12 weeks total across team for the migration.
- Operational: 2 weeks for shadow + canary infrastructure.

### 8.10 The lessons

- 6-month timeline accommodates standard work.
- Parallel running gives confidence.
- Cross-team coordination essential.
- Customer communication matters.

---

## 9. Anti-patterns

### 9.1 The "we'll handle it when the deadline approaches" deferral

**Pattern.** 6-month notice; deferred 5 months; 30 days to migrate.

**Corrective.** Start within weeks per §2.5.

### 9.2 The "vendor recommends X; we use X" thoughtlessness

**Pattern.** Vendor's recommendation accepted without eval.

**Corrective.** Per-workload eval per §4.3.

### 9.3 The skipped parallel-running

**Pattern.** Direct switch; quality regression discovered in production.

**Corrective.** Parallel run per §5.

### 9.4 The "deprecated had unique capability we missed" surprise

**Pattern.** Deprecated had subtle strength; candidate lacks it; production worse.

**Corrective.** Per §4.8.

### 9.5 The no-rollback-after-sunset realization

**Pattern.** Production switched; vendor sunset; issues emerge; can't roll back.

**Corrective.** Resolve before sunset per §7.3.

### 9.6 The premature customer-promise

**Pattern.** Customer told migration date before validation complete; deadline slips.

**Corrective.** Promise only after validation.

### 9.7 The "we don't know which workloads use it" gap

**Pattern.** Workload audit not done; missed workloads break later.

**Corrective.** Inventory per §3.1.

### 9.8 The cross-team-coordination-failure

**Pattern.** AI team migrating; product team unaware; customers surprised.

**Corrective.** Cross-team per §3.5.

### 9.9 The "we just port the prompt verbatim" laxity

**Pattern.** Prompt not adapted to new model; behavior differs.

**Corrective.** Prompt port per §4.5.

### 9.10 The compressed-timeline panic

**Pattern.** Started late; everything compressed; risk high.

**Corrective.** Plan per §2.1.

---

## 10. Findings (sprint-assignable)

### ML-MDP-001 — Severity: Critical
**Finding.** No migration project plan.
**Recommendation.** Per §3.7.
**Owner.** AI platform + engineering management, sprint N+1.

### ML-MDP-002 — Severity: Critical
**Finding.** Workload inventory absent.
**Recommendation.** Per §3.1.
**Owner.** AI platform, sprint N+1.

### ML-MDP-003 — Severity: Critical
**Finding.** Candidate evaluation absent.
**Recommendation.** Per §4.3.
**Owner.** AI platform + eval, sprint N+1.

### ML-MDP-004 — Severity: High
**Finding.** Parallel running absent.
**Recommendation.** Per §5.
**Owner.** AI platform, sprint N+2.

### ML-MDP-005 — Severity: High
**Finding.** Cross-team coordination absent.
**Recommendation.** Per §3.5.
**Owner.** engineering management, sprint N+2.

### ML-MDP-006 — Severity: High
**Finding.** Customer-communication plan absent.
**Recommendation.** Per §3.6 and §7.8.
**Owner.** product + customer success, sprint N+2.

### ML-MDP-007 — Severity: High
**Finding.** Prompt-port plan absent.
**Recommendation.** Per §4.5.
**Owner.** AI platform, sprint N+2.

### ML-MDP-008 — Severity: Medium
**Finding.** Multi-workload sequencing not planned.
**Recommendation.** Per §6.2.
**Owner.** AI platform, sprint N+3.

### ML-MDP-009 — Severity: Medium
**Finding.** Rollback-window expiration not considered.
**Recommendation.** Per §7.3.
**Owner.** AI platform, sprint N+3.

### ML-MDP-010 — Severity: Medium
**Finding.** Buffer time absent.
**Recommendation.** Per §2.3.
**Owner.** AI platform, sprint N+3.

### ML-MDP-011 — Severity: Medium
**Finding.** Compliance verification of candidate absent.
**Recommendation.** Per §4.7.
**Owner.** AI platform + compliance, sprint N+3.

### ML-MDP-012 — Severity: Medium
**Finding.** Per-workload sequencing not prioritized.
**Recommendation.** Per §3.2.
**Owner.** AI platform + product, sprint N+3.

### ML-MDP-013 — Severity: Medium
**Finding.** Customer-tier sequencing absent.
**Recommendation.** Per §6.3.
**Owner.** AI platform + product, sprint N+4.

### ML-MDP-014 — Severity: Medium
**Finding.** Post-migration eval absent.
**Recommendation.** Per §7.6.
**Owner.** AI platform + eval, sprint N+4.

### ML-MDP-015 — Severity: Low
**Finding.** Lessons-learned documentation absent.
**Recommendation.** Per §7.7.
**Owner.** engineering management, sprint N+5.

### ML-MDP-016 — Severity: Low
**Finding.** Migration project tracking ad-hoc.
**Recommendation.** Project management infrastructure.
**Owner.** engineering management, sprint N+5.

### ML-MDP-017 — Severity: Low
**Finding.** Migration cost not budgeted.
**Recommendation.** Per §2.9.
**Owner.** engineering management + FinOps, sprint N+6.

### ML-MDP-018 — Severity: Low
**Finding.** Vendor-relationship management for sunsets absent.
**Recommendation.** Subscribe to release notes per [model-registry.md](./model-registry.md).
**Owner.** AI platform, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Inventory workloads per §3.1.**
- [ ] **Subscribe to vendor release notes per §8.10.**
- [ ] **Define migration project template per §3.7.**
- [ ] **Implement candidate eval per §4.3.**
- [ ] **Implement parallel-run infrastructure per §5.**
- [ ] **Plan customer communication template per §3.6.**
- [ ] **Per-workload priority sequencing per §6.2.**
- [ ] **Customer-tier rollout per §6.3.**
- [ ] **Post-migration eval per §7.6.**
- [ ] **Lessons-learned process per §7.7.**

---

## 12. References

**In this folder.**
- [model-registry.md](./model-registry.md) — catalog tracks deprecation dates.
- [model-promotion.md](./model-promotion.md) — promotion of replacement.
- [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) — rollout mechanics.
- [rollback-procedures.md](./rollback-procedures.md) — rollback.
- [fine-tuning-operations.md](./fine-tuning-operations.md) — fine-tunes need re-training on new base.

**Elsewhere in this repo.**
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — eval discipline.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — model deprecation surprise class.
- [cost-and-finops/finops-process.md](../cost-and-finops/finops-process.md) — vendor management cadence.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — architectural migration playbook.
- [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md) — catalog with deprecation tracking.

**External.**
- Anthropic, OpenAI, Google deprecation announcement processes.
- Industry-wide model deprecation literature.
- Project management methodologies for software migrations.
