# Rollback Procedures

> **Audience.** Engineers on-call when a model deployment goes wrong. SREs whose rollback runbook for AI features doesn't exist. Anyone whose first rollback was improvised during an incident. **Scope.** The *engineering* practice of model rollback: detection (quality alert, cost alert, user complaints); decision (rollback vs hot-patch); execution (pinned-version rollback through the registry); post-incident review. The "no rollback path" anti-pattern that turns a quality issue into an extended outage. Not the broader incident response (see [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md)). Not the canary mechanics (see [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Rollback is the safety net. Without it:

- Bad deployment → extended outage.
- Quality regression → customer pain.
- Cost spike → financial damage.

With proper rollback:

- Issue detected.
- Rollback executed.
- Investigation follows.

The discipline:

- Rollback is pre-defined.
- Tested before incident.
- Fast (minutes).
- Reversible (can re-roll-forward).

This document is opinionated about four things:

1. **Every deployment has a rollback path.** If not: don't deploy.
2. **Rollback is tested in pre-production.** First real rollback shouldn't be the first test.
3. **Auto-rollback for critical metrics.** Don't rely on humans during incident.
4. **Post-rollback review is mandatory.** Process improvement.

Structure: (2) the rollback triggers; (3) detection; (4) decision (rollback vs hot-patch); (5) execution; (6) verification; (7) post-rollback review; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The rollback triggers

What prompts rollback.

### 2.1 Quality regression

Live-judge quality drops past threshold:

- Quality SLO breached.
- Sustained drop (not noise).

Cross-link to [reliability-engineering/fault-budgets-for-ai.md §3](../reliability-engineering/fault-budgets-for-ai.md).

### 2.2 Cost spike

Cost per call jumps:

- Burn rate exceeds threshold.
- Per-call cost up >2x.

Cross-link to [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md).

### 2.3 Latency degradation

P99 latency past SLO:

- Sustained spike.
- Beyond noise.

### 2.4 Error rate spike

HTTP errors, validation failures:

- Rate exceeds threshold.

### 2.5 User complaints

Spike in support tickets, thumbs-down:

- May lag other metrics.

### 2.6 The "we should have rolled back" hindsight

Common diagnosis:

- Issue detectable hours earlier.
- Manual rollback delayed.

Automate.

### 2.7 The trigger hierarchy

Per workload:

- Severity 1 (auto-rollback): quality drop >10%, cost >3x, errors >5x.
- Severity 2 (page on-call): quality drop 5-10%, cost 2-3x.
- Severity 3 (alert): quality drop 1-5%, cost 1.5-2x.

Per workload.

### 2.8 The "we'll rollback if there's a problem" vagueness

Without explicit triggers:

- Subjective; slow.

Pre-defined thresholds.

### 2.9 The detection-to-trigger latency

How fast: trigger to rollback initiated:

- Auto: seconds to minutes.
- Human: tens of minutes (typical).

For high-stakes: auto.

---

## 3. Detection

How issues surface.

### 3.1 The monitoring stack

- Quality metrics (live judge).
- Cost metrics (per-call attribution).
- Latency metrics.
- Error metrics.
- User feedback.

Per-metric pipeline.

Cross-link to [observability-and-telemetry/](../observability-and-telemetry/).

### 3.2 The alert routing

Per trigger:

- Auto-rollback fires; engineer notified.
- Page (for higher severity).
- Ticket (for lower).

Per alert design.

### 3.3 The dashboard

Real-time:

- All metrics.
- Recent canary status.
- Recent deployments.

For investigation.

### 3.4 The "we have logs but no metrics" gap

Without metrics:

- Detection by user complaints (lagging).
- Manual investigation.

Build metrics.

### 3.5 The false-positive vs false-negative

- False-positive: rollback when not needed; minor harm (deploy lost).
- False-negative: didn't rollback; quality / cost issue persists.

Tuning balances both.

### 3.6 The "we tuned alerts; got fewer pages but more issues" lesson

Over-aggressive tuning:

- Fewer false alarms.
- More missed real issues.

Calibrate.

### 3.7 The "we got a page; deployment was fine" investigation

If alert fired and rollback unjustified:

- Investigate the alert.
- Tune.
- Document.

### 3.8 The alert escalation

Per severity:

- Slack notification.
- Phone page.
- Manager / oncall escalation.

Per response time needed.

### 3.9 The cross-team detection

Different team owns metric:

- AI platform team owns quality.
- FinOps owns cost.
- SRE owns latency.

Coordination during incident.

---

## 4. Decision (rollback vs hot-patch)

What to do.

### 4.1 The rollback option

Revert to previous version:

- Pinned version pulled back.
- Time: minutes.
- Reversible.

Standard.

### 4.2 The hot-patch option

Fix in place:

- Patch the bug.
- Re-deploy.
- Time: hours.
- More risk.

When new fix is obvious.

### 4.3 The decision criteria

Rollback when:

- Issue unclear; investigating.
- Rollback fast.
- Previous version known good.

Hot-patch when:

- Issue identified.
- Fix simple.
- Rollback would lose meaningful improvements.

Per-incident.

### 4.4 The "we want to fix forward; not roll back" mindset

Avoid for high-stakes:

- "Fix forward" takes longer.
- During fix: issue persists.

Roll back; fix forward later.

### 4.5 The "rollback then investigate" pattern

Standard:

1. Detected issue.
2. Rollback.
3. Stable.
4. Investigate.
5. Develop fix.
6. Re-deploy via canary.

Time pressure off.

### 4.6 The "we don't know the cause; rollback" caution

If cause is unclear:

- Better to roll back.
- Investigate without pressure.

### 4.7 The "the previous version was also broken" complication

Sometimes:

- Newer version is bad.
- Previous version was also bad (different way).

Mitigations:

- Roll back to older.
- Or fix forward (forced).

Rare; investigate carefully.

### 4.8 The hot-patch eval

Before re-deploying hot-patch:

- Run eval suite.
- Quick verification.

Cross-link to [model-promotion.md §4](./model-promotion.md).

### 4.9 The decision-time constraint

Rollback decision:

- Within minutes (if auto-trigger).
- Within an hour (if manual).

Don't deliberate; act.

### 4.10 The post-decision audit

Document:

- Why rollback chosen.
- Or why hot-patch.
- Outcome.

For learning.

---

## 5. Execution

The mechanics of rollback.

### 5.1 The pinned-version pattern

Current production: pinned to version X.

Previous production: pinned to version X-1.

Rollback: change pin from X to X-1.

Cross-link to [model-registry.md](./model-registry.md).

### 5.2 The traffic-shift

Mechanism:

- Update feature flag.
- Or update routing.
- Or update config.

Per platform.

### 5.3 The rollback duration

From decision to verified:

- Auto: 1-5 minutes.
- Manual: 10-30 minutes.

Per platform.

### 5.4 The "we don't have the previous version available" failure

If previous decommissioned:

- Rollback impossible.

Retain previous versions per [model-promotion.md §6.6](./model-promotion.md).

### 5.5 The in-flight requests

During rollback:

- In-flight requests on new version: complete (or fail).
- New requests: routed to old version.

Per platform.

### 5.6 The data consistency

For database / side-effect operations:

- Rollback may leave inconsistent state.
- Compensation may be needed.

For pure inference: less concern.

### 5.7 The catalog update

Per rollback:

- Mark new version as "rolled-back".
- Mark previous as active.
- Audit log entry.

Cross-link to [model-registry.md](./model-registry.md).

### 5.8 The multi-region rollback

For multi-region:

- Rollback per region (sequential? parallel?).
- Coordination.

Per platform.

### 5.9 The customer-facing notification

If rollback affects users:

- Notification (banner; email).
- Status page update.

Communication.

### 5.10 The "rollback failed" recovery

If rollback itself fails:

- Disaster scenario.
- Investigate.
- Escalate.

Have backup rollback paths.

---

## 6. Verification

After rollback.

### 6.1 The post-rollback metrics

Verify:

- Quality returned to baseline.
- Cost returned to baseline.
- Latency returned to baseline.
- Errors returned to baseline.

Per metric.

### 6.2 The verification window

After rollback:

- 30-60 min observation.
- Verify stable.

Confirmation.

### 6.3 The "rollback partial; some traffic still on new" check

For multi-region or sticky-session:

- Some traffic might still be on new.
- Verify 100% on old.

### 6.4 The post-rollback investigation

Why did the issue happen:

- What was different in new version?
- Why didn't staging catch it?
- Investigation per incident-response.

Cross-link to [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md).

### 6.5 The "we rolled back; old version is also bad" surprise

Sometimes:

- New is bad; old was also bad (different way).

Investigate.

### 6.6 The user-impact assessment

Per rollback:

- How many users affected.
- How long.
- Customer support implications.

Quantify.

### 6.7 The post-rollback communication

To stakeholders:

- "Rollback completed."
- "Investigation ongoing."
- Resolution timeline.

Honest.

### 6.8 The "we'll re-deploy when fix is ready" plan

For rolled-back release:

- Fix developed.
- Re-evaluation.
- Re-canary.
- Re-deploy.

Standard.

### 6.9 The catalog-of-rolled-back-releases

Per workload:

- Catalog of rolled-back versions.
- Lessons learned.

Audit.

### 6.10 The rollback metrics

Track:

- Rollback rate (rollbacks per N deployments).
- Rollback time (decision to stable).
- Rollback success rate.

For improvement.

---

## 7. Post-rollback review

The retrospective.

### 7.1 The review template

```
Rollback Incident: [name]
Date: [...]
Duration: detection → stable: [time]

Summary:
What happened.

Detection:
What alert fired; how quickly.

Decision:
Rollback vs hot-patch; rationale.

Execution:
Rollback steps; time.

Verification:
Post-rollback metrics.

Root cause:
What caused the issue.

Why didn't staging catch:
Gap analysis.

Prevention plan:
- What changes prevent recurrence.
- Owner.
- Due date.
```

Per rollback.

### 7.2 The cross-team participation

- AI platform.
- Feature team.
- SRE.
- Product.

Inclusive.

### 7.3 The "rollback was fast; great" complacency

If rollback was effective:

- Still: why did the issue occur?
- Process improvement opportunity.

### 7.4 The "we'll do better next time" vagueness

Without specific changes:

- Same incident recurs.

Specific actions.

### 7.5 The action items

Per review:

- Specific changes.
- Owners.
- Due dates.

Tracked.

### 7.6 The pattern across rollbacks

If multiple rollbacks:

- Same cause?
- Process gap.
- Bigger fix.

Pattern recognition.

### 7.7 The cross-incident learning

Lessons applied broadly:

- Other workloads benefit.
- Platform improvement.

### 7.8 The post-mortem published

For significant rollbacks:

- Internal post-mortem.
- Blameless.
- Documented.

Cross-link to [reliability-engineering/incident-response-for-ai.md §8](../reliability-engineering/incident-response-for-ai.md).

### 7.9 The "we want to never have to rollback" aspiration

Healthy goal:

- Better staging.
- More thorough eval.
- Better detection.

But: have rollback ready regardless.

### 7.10 The audit-log retention

Per incident:

- Review documented.
- Retained per compliance.

---

## 8. Worked Meridian example

Meridian's rollback practice.

### 8.1 The infrastructure

```
Feature flags per workload.
Auto-rollback rules.
Pre-defined thresholds.
Catalog of rolled-back versions.
```

Standard.

### 8.2 The Q1 2026 quality rollback

Cross-link to [cost-incident-runbook.md §9](../cost-and-finops/cost-incident-runbook.md).

A Care Coordinator prompt change:

- Deployed via canary.
- Quality dropped (live judge).
- Auto-rollback fired within 10 minutes.

Detection → stable: 12 minutes.

### 8.3 The Q1 2026 cost rollback

A Patient API chat new prompt:

- Cost per call doubled.
- Auto-rollback at canary.

Detection → stable: 8 minutes.

Cost impact: ~$200 (caught early).

### 8.4 The Q2 2026 manual rollback

A Care Coordinator agent update:

- Latency degraded; below auto-threshold.
- Engineer paged for review.
- Decision: rollback (uncertain cause).

Detection → stable: 35 minutes.

### 8.5 The catalog of rolled-back versions

```
Q1 2026:
  Care Coordinator v14.1.3 (rolled back; quality regression).
  Patient API v3.4.2 (rolled back; cost spike).
  Care Coordinator v14.1.4 (deployed; not rolled back).

Q2 2026:
  Care Coordinator v14.2.5 (rolled back; latency).
  Document classification v12.1.0 (deployed; not rolled back).

Q3 2026: ...
```

Pattern: ~2-3 rollbacks per quarter; absorbed by the system.

### 8.6 The pre-rollback testing

Quarterly drill:

- Synthetic issue triggered in staging.
- Rollback procedure executed.
- Verified working.

Catches infrastructure issues before real incident.

### 8.7 The post-rollback reviews

Per significant rollback:

- 30-minute review.
- Cross-team.
- Action items.

Tracked.

### 8.8 The "we caught it before customer impact" results

Most rollbacks:

- Caught in canary (5-25% traffic).
- Few customers affected.
- Trust maintained.

### 8.9 The detection metric

Average detection time:

- Quality issue: 5-15 minutes (live judge).
- Cost issue: 2-10 minutes (real-time cost metric).
- Latency: 3-10 minutes.
- Errors: < 5 minutes.

Fast enough.

### 8.10 The rollback success rate

```
Q1 2026: 4 rollbacks; 4 successful (100%).
Q2 2026: 2 rollbacks; 2 successful.
```

Infrastructure works.

### 8.11 The infrastructure cost

- Feature flag platform: ~$500/month (LaunchDarkly).
- Metrics pipeline: shared with other observability.
- Engineering: ~5% of platform team.

### 8.12 The lessons

- Auto-rollback is essential.
- Pre-defined thresholds matter.
- Drills catch infrastructure issues.
- Post-rollback reviews build the platform.

---

## 9. Anti-patterns

### 9.1 The no-rollback-path

**Pattern.** New version deployed; previous version decommissioned. Rollback impossible.

**Corrective.** Retain previous per §5.4.

### 9.2 The "we'll figure out rollback during incident" improvisation

**Pattern.** First rollback is during incident. Slow; error-prone.

**Corrective.** Pre-tested per §8.6.

### 9.3 The no-auto-rollback

**Pattern.** Rollback requires human action; incidents drag.

**Corrective.** Auto-rollback for critical per §2.7.

### 9.4 The "rollback was hot-patch in disguise" muddle

**Pattern.** Hot-patch labeled rollback. Confusion.

**Corrective.** Per §4.2.

### 9.5 The "we didn't verify post-rollback" gap

**Pattern.** Rollback assumed working; sometimes isn't.

**Corrective.** Verification per §6.

### 9.6 The "we'll review later" deferral

**Pattern.** Rollback happens; review never. Patterns invisible.

**Corrective.** Review per §7.

### 9.7 The skipped catalog-update

**Pattern.** Rolled-back version not marked; future engineer redeploys it.

**Corrective.** Per §5.7.

### 9.8 The "multi-region rollback we forgot one" gap

**Pattern.** Rollback in primary region; secondary still on bad version.

**Corrective.** Per §5.8.

### 9.9 The customer-not-told

**Pattern.** Rollback impacts customers; no communication.

**Corrective.** Per §5.9.

### 9.10 The "rollback rate is high; we're fine" complacency

**Pattern.** Many rollbacks; team says "we have it under control."

**Corrective.** Pattern recognition per §7.6.

---

## 10. Findings (sprint-assignable)

### ML-RB-001 — Severity: Critical
**Finding.** No rollback path.
**Recommendation.** Per §5.
**Owner.** AI platform, sprint N+1.

### ML-RB-002 — Severity: Critical
**Finding.** Rollback not tested.
**Recommendation.** Per §8.6.
**Owner.** SRE + AI platform, sprint N+1.

### ML-RB-003 — Severity: Critical
**Finding.** No auto-rollback for critical metrics.
**Recommendation.** Per §2.7.
**Owner.** AI platform, sprint N+1.

### ML-RB-004 — Severity: High
**Finding.** Rollback triggers vague.
**Recommendation.** Per §2.7.
**Owner.** AI platform + product, sprint N+2.

### ML-RB-005 — Severity: High
**Finding.** Rollback execution slow.
**Recommendation.** Per §5.3.
**Owner.** AI platform, sprint N+2.

### ML-RB-006 — Severity: High
**Finding.** Post-rollback verification absent.
**Recommendation.** Per §6.
**Owner.** AI platform, sprint N+2.

### ML-RB-007 — Severity: High
**Finding.** Previous versions not retained.
**Recommendation.** Per §5.4.
**Owner.** AI platform, sprint N+2.

### ML-RB-008 — Severity: Medium
**Finding.** Post-rollback review absent.
**Recommendation.** Per §7.
**Owner.** SRE + AI platform, sprint N+3.

### ML-RB-009 — Severity: Medium
**Finding.** Catalog-update on rollback missing.
**Recommendation.** Per §5.7.
**Owner.** AI platform, sprint N+3.

### ML-RB-010 — Severity: Medium
**Finding.** Multi-region rollback coordination absent.
**Recommendation.** Per §5.8.
**Owner.** AI platform, sprint N+3.

### ML-RB-011 — Severity: Medium
**Finding.** Customer-communication template absent.
**Recommendation.** Per §5.9.
**Owner.** product + customer success, sprint N+3.

### ML-RB-012 — Severity: Medium
**Finding.** Rollback metrics not tracked.
**Recommendation.** Per §6.10.
**Owner.** observability + SRE, sprint N+3.

### ML-RB-013 — Severity: Medium
**Finding.** Data-consistency after rollback unclear.
**Recommendation.** Per §5.6.
**Owner.** AI platform, sprint N+4.

### ML-RB-014 — Severity: Medium
**Finding.** Cross-team detection coordination unclear.
**Recommendation.** Per §3.9.
**Owner.** SRE + AI platform, sprint N+4.

### ML-RB-015 — Severity: Low
**Finding.** Drills not on schedule.
**Recommendation.** Quarterly per §8.6.
**Owner.** SRE, sprint N+5.

### ML-RB-016 — Severity: Low
**Finding.** Pattern recognition across rollbacks absent.
**Recommendation.** Per §7.6.
**Owner.** SRE + AI platform, sprint N+5.

### ML-RB-017 — Severity: Low
**Finding.** Post-mortems not published.
**Recommendation.** Per §7.8.
**Owner.** SRE, sprint N+6.

### ML-RB-018 — Severity: Low
**Finding.** Rollback-rate trending up; not investigated.
**Recommendation.** Per §9.10.
**Owner.** engineering management, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Retain previous versions per §5.4.**
- [ ] **Define rollback triggers per §2.7.**
- [ ] **Implement auto-rollback per §2.7.**
- [ ] **Test rollback procedure pre-incident per §8.6.**
- [ ] **Build post-rollback verification per §6.**
- [ ] **Post-rollback review process per §7.**
- [ ] **Catalog-update on rollback per §5.7.**
- [ ] **Cross-team coordination per §3.9.**
- [ ] **Track rollback metrics per §6.10.**
- [ ] **Quarterly drills.**

---

## 12. References

**In this folder.**
- [model-registry.md](./model-registry.md) — version pinning.
- [model-promotion.md](./model-promotion.md) — promotion produces rollback paths.
- [canary-and-shadow-rollout.md](./canary-and-shadow-rollout.md) — canary may trigger rollback.
- [model-deprecation-playbook.md](./model-deprecation-playbook.md) — migrations may require rollback.
- [fine-tuning-operations.md](./fine-tuning-operations.md) — fine-tune deployments rollback.

**Elsewhere in this repo.**
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — broader incident response.
- [reliability-engineering/fault-budgets-for-ai.md](../reliability-engineering/fault-budgets-for-ai.md) — SLO breach triggers rollback.
- [observability-and-telemetry/](../observability-and-telemetry/) — observability for detection.
- [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md) — cost incident response.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — migration as project.

**External.**
- Feature-flag platform rollback documentation.
- Continuous delivery rollback patterns.
- Google SRE Book — release engineering.
