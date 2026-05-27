# Canary and Shadow Rollout

> **Audience.** Engineers whose model deployments are "deploy to 100% on Tuesday." Tech leads whose new-model rollout strategy is "wait and see." Anyone whose model-rollout has scared them recently. **Scope.** The *engineering* practice of canary and shadow rollout for model deployments: canary (small percentage of real traffic; monitor quality / cost / latency; ramp); shadow traffic (full traffic to new model in parallel; compare outputs; no user impact); cost-of-shadow trade-off; rollback criteria. Not the full migration playbook (see [model-deprecation-playbook.md](./model-deprecation-playbook.md), companion). Not the rollback procedures themselves (see [rollback-procedures.md](./rollback-procedures.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

When deploying a new model version, the deployment strategy matters:

- All-at-once: if there's a problem, all users affected.
- Canary: small subset; observe; ramp.
- Shadow: full traffic; new model in parallel; no user impact; compare.

Each has trade-offs.

The discipline:

- Most production deployments use canary.
- For high-stakes: combine canary + shadow.
- Quick rollback when canary metrics breach.

This document covers the mechanics.

This document is opinionated about four things:

1. **Canary is the default.** Not "deploy at 100% and hope."
2. **Rollback criteria are pre-defined.** Not subjective at incident time.
3. **Shadow is the validator before canary, not a replacement.** For high-stakes workloads.
4. **Automation is essential.** Manual canary management doesn't scale.

Structure: (2) the canary pattern; (3) the shadow pattern; (4) canary metrics; (5) ramp design; (6) rollback criteria; (7) automated decisions; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist; (12) references.

---

## 2. The canary pattern

Small subset; observe; ramp.

### 2.1 The mechanic

```
Production traffic →
  Router decides: which model.
  Canary fraction (e.g., 5%) → new model.
  Rest (95%) → existing model.
  
Monitor canary metrics.
If green: ramp.
If red: rollback.
```

Per request, routing.

### 2.2 The ramp schedule

```
T=0: 5%
T=24h: 25%
T=72h: 50%
T=120h: 100%
```

Gradual.

### 2.3 The "we go from 5% to 100% in 1 day" rush

Compressed:

- Less observation time.
- Issues may not surface.

Standard ramp: 5-7 days.

### 2.4 The "we deploy at 100%" risk

Direct:

- All users at risk.
- No safety net.

Don't.

### 2.5 The canary-fraction tuning

Per workload:

- Lower stakes: 10-20% initial canary.
- Higher stakes: 1-5% initial.

Per workload's tolerance.

### 2.6 The "we have low volume; canary is unreliable" issue

If canary at 5%:

- Very few requests in canary.
- Metrics noisy.

Either:

- Higher canary percentage.
- Longer observation window.

### 2.7 The traffic-splitting infrastructure

Mechanisms:

- Feature flags (Statsig, LaunchDarkly).
- Custom routing.
- Provider's traffic splitting (some).

Per platform.

### 2.8 The sticky session

For some workloads:

- A user's session sticks to one variant.
- Avoids switching mid-conversation.

Cross-link to [model-strategy/multi-provider-failover.md §6.5](https://github.com/jeremiahredden/ai-engineering-reference-architecture/blob/main/reliability-engineering/multi-provider-failover.md).

### 2.9 The "we don't have flag infrastructure" reality

If no flag system:

- Alternative: deployment slots.
- Less flexible.

Long-term: build flag infrastructure.

---

## 3. The shadow pattern

Full traffic; parallel new model; no user impact.

### 3.1 The mechanic

```
Production traffic →
  Existing model: response returned to user.
  
  Simultaneously:
  New model called with same input → response logged (not returned).

Compare outputs offline.
```

User sees existing; engineer sees both.

### 3.2 The cost-of-shadow

Doubled inference cost:

- 100% to existing.
- 100% to shadow.

Cost = 2x normal.

### 3.3 The shadow duration

For meaningful comparison:

- 1-2 weeks typical.
- Sufficient sample.

### 3.4 The shadow comparison

For each pair (existing-output, new-output):

- Semantic similarity.
- Schema validity (if structured).
- Quality scores.

Aggregate.

### 3.5 The "shadow output is different; was it better" question

Existing and new differ:

- Better: how to know?
- Worse: how to know?

Difficult.

Mitigations:

- Sample human review.
- Eval suite as proxy.
- A/B in canary (where possible).

### 3.6 The "shadow doesn't tell us about real impact" limitation

Shadow data:

- Tells you outputs differ.
- Doesn't tell you which users prefer / which would prefer.

For that: canary (real users).

### 3.7 The shadow-as-pre-canary

Common workflow:

- Shadow first (1-2 weeks; offline comparison).
- Then canary (gradual user exposure).

Higher confidence; higher cost.

### 3.8 The "we just canary; skip shadow" justification

For lower-stakes:

- Canary alone sufficient.
- Shadow's 2x cost not justified.

Trade-off.

### 3.9 The shadow for migration

When migrating models (cross-link to [model-deprecation-playbook.md](./model-deprecation-playbook.md)):

- Shadow new model during deprecation period.
- Catch differences pre-migration.

### 3.10 The shadow infrastructure

Per request:

- Fire new-model call (async).
- Log result.
- Don't block user.

Infrastructure for parallel calls.

---

## 4. Canary metrics

What to monitor.

### 4.1 Quality metrics

Per workload:

- Live judge (eval).
- User feedback.
- Schema validation rate.

Quality SLO compliance.

Cross-link to [reliability-engineering/fault-budgets-for-ai.md §3](../reliability-engineering/fault-budgets-for-ai.md).

### 4.2 Latency metrics

P50, P95, P99.

Compare to existing.

### 4.3 Cost metrics

- Cost per call.
- Aggregate cost.

Compare to existing.

### 4.4 Error rate

- HTTP errors.
- Schema validation failures.
- Refusals.

Higher errors → red flag.

### 4.5 User feedback

- Thumbs-up rate.
- Support tickets.
- User complaints.

Lagging indicators.

### 4.6 The metric thresholds

Per metric:

- "Green" range (canary continues).
- "Yellow" range (alert; humans review).
- "Red" range (auto-rollback).

Pre-defined.

### 4.7 The "we don't have all these metrics" gap

Some teams:

- Only latency / error rate.
- Quality + cost not tracked.

Build these per [fault-budgets-for-ai.md](../reliability-engineering/fault-budgets-for-ai.md).

### 4.8 The metric-aggregation window

Per metric:

- Aggregate over window (e.g., last hour).
- Smooths noise.

Per-metric appropriate.

### 4.9 The comparison-baseline

Per metric:

- Compare to existing model in parallel.
- Or: compare to previous canary period.

Apples-to-apples.

### 4.10 The canary-dashboard

Per canary:

```
Canary status:
  Traffic share: 25%
  Duration: 36h
  
  Quality: 96.1% (vs 96.3% existing; -0.2)
  Latency P99: 4.8s (vs 5.2s existing; -8%)
  Cost: $0.018 (vs $0.022 existing; -18%)
  Error rate: 0.2% (vs 0.3% existing; -33%)
  
Status: GREEN; ramp to 50% scheduled.
```

Real-time.

---

## 5. Ramp design

The progressive rollout.

### 5.1 The standard ramp

```
T=0: 5%
T=24h: 25% (if green)
T=72h: 50%
T=120h: 100%
```

Per workload tolerance.

### 5.2 The accelerated ramp

For confident teams:

```
T=0: 10%
T=12h: 50%
T=24h: 100%
```

Faster; more risk.

### 5.3 The conservative ramp

For high-stakes:

```
T=0: 1%
T=24h: 5%
T=48h: 10%
T=72h: 25%
T=120h: 50%
T=168h: 100%
```

7+ days.

### 5.4 The per-workload ramp

Per workload's risk profile.

### 5.5 The "ramp paused; investigating" hold

If metrics show yellow:

- Hold at current.
- Investigate.
- Resume or rollback.

Yellow ≠ rollback; just hold.

### 5.6 The "we ramp same model across all features" simplification

If multiple features use same model:

- Each feature's canary independent.
- Or: shared canary (cheaper; coordination).

Per platform.

### 5.7 The "model release; multiple features" coordination

Major model upgrades:

- Coordinate features.
- Stage timing.

### 5.8 The ramp-completion criterion

Full deployment:

- All metrics green.
- No regressions.
- 24h+ stable at 100%.

Then: declared complete.

### 5.9 The "we have low volume" patient ramp

For low-volume features:

- Longer at each step.
- More observation time.

Patience.

---

## 6. Rollback criteria

When to abort.

### 6.1 The pre-defined criteria

```yaml
auto-rollback-triggers:
  quality: pass rate drops > 5% from baseline
  latency: P99 increases > 30% from baseline
  cost: cost per call > 2x baseline
  errors: error rate > 5x baseline
  user_complaints: > 10 in 1 hour

manual-review-triggers (yellow):
  quality: drop 1-5%
  latency: increase 10-30%
  cost: 1.2-2x
  errors: 2-5x
  user_complaints: 3-10 in 1 hour
```

Per workload.

### 6.2 The auto-rollback

For red triggers:

- Automatic.
- Within minutes.
- No human delay.

Critical for cost / safety.

### 6.3 The manual-review for yellow

For yellow:

- Alert engineers.
- Human investigates.
- Decision.

Some delays acceptable.

### 6.4 The rollback time

Auto: minutes (router shifts traffic back).

Manual: depends on investigation.

### 6.5 The rollback verification

After rollback:

- Verify previous model serving.
- Verify metrics return to baseline.

Confirmation.

### 6.6 The "we rolled back; what now" recovery

Investigation:

- Why did canary metrics breach?
- Was new model bad? Or external?

Iterate.

### 6.7 The "we never roll back" hubris

Teams that never roll back:

- Either lucky.
- Or aggressive (eventually bad).

Have rollback ready.

### 6.8 The "we rolled back twice; same issue" pattern

If rollback doesn't help:

- Issue may not be the model.
- External cause.

Diagnose deeper.

### 6.9 The post-rollback review

After rollback:

- Why did we miss this in staging?
- What metric should have caught it?
- Improve.

Process.

---

## 7. Automated decisions

Letting the system decide.

### 7.1 The full-auto canary

For mature workloads:

- Canary starts automatically.
- Ramps based on metrics.
- Rollback on red.
- Engineer notified.

Minimal human intervention.

### 7.2 The threshold automation

```python
def evaluate_canary():
    metrics = get_canary_metrics(window='1h')
    
    if metrics.quality_drop > 5 or metrics.cost > 2x or metrics.errors > 5x:
        rollback()
        return "ROLLED_BACK"
    
    if all_metrics_green and time_at_current_share > 24h:
        ramp_up()
        return "RAMPED"
    
    return "HOLDING"
```

Programmatic.

### 7.3 The "we'll review canary manually each time" overhead

Manual reviews:

- Don't scale.
- Slow.

Automate.

### 7.4 The human-override

Always:

- Engineer can override automation.
- Halt; advance; rollback.

Backup.

### 7.5 The "automation has a bug; canary went wrong" risk

Mitigations:

- Human approval for major thresholds.
- Daily review.

### 7.6 The escalation

If auto-canary detects unusual:

- Page engineer.
- Even if auto-rollback handled.

For learning.

### 7.7 The "we don't have automation; doing it manually" reality

For early-stage:

- Manual.
- Build automation as scale demands.

### 7.8 The audit log

Per canary:

- Decisions logged.
- Auto-actions recorded.

For post-incident.

### 7.9 The cost-of-automation

- Engineering: 2-4 weeks to build.
- Ongoing: minimal.

Worth it at scale.

---

## 8. Worked Meridian example

Meridian's canary practice.

### 8.1 The standard ramp

```
Care Coordinator new model deployment:
  T=0: 5% canary
  T=24h: 25% (auto-ramp if green)
  T=72h: 50%
  T=120h: 100%

Patient API chat:
  Similar.

Document classification:
  Faster (8h, 25h, 49h) — less risk; smaller blast.
```

Per workload.

### 8.2 The metrics + thresholds

```yaml
care-coordinator-canary:
  quality_threshold_red: pass rate < 91% (vs 96% baseline)
  latency_threshold_red: P99 > 12s (vs 8s baseline)
  cost_threshold_red: cost-per-call > $0.06 (vs $0.03 baseline)
  errors_threshold_red: rate > 1%
  
patient-api-chat-canary:
  quality_threshold_red: pass rate < 88% (vs 92% baseline)
  latency_threshold_red: P99 TTFT > 3s
  cost_threshold_red: per-conversation > $0.08
```

Per workload.

### 8.3 The Q1 2026 canary rollback

A Care Coordinator new-prompt canary:

- T=0: 5%.
- T=2h: quality dropped from 96% to 87% (red).
- Auto-rollback fired.
- Investigation: prompt change introduced inconsistency.

Recovery within 10 minutes.

### 8.4 The Q2 2026 canary success

Sonnet 4.5 → 4.6 migration:

- Shadow phase: 2 weeks; outputs compared.
- Canary 5%: metrics green.
- Ramp to 25%, 50%, 100% over 5 days.
- Successful.

Smooth.

### 8.5 The shadow trial

For Sonnet 4.5 → 4.6:

- Shadow Sonnet 4.6 in parallel.
- 100k requests in shadow over 2 weeks.
- Quality comparison: 4.6 superior on 60% of cases; equal on 35%; worse on 5%.
- Decision: deploy with confidence.

Shadow's value: pre-canary validation.

### 8.6 The auto-rollback prevented a quality incident

The Q1 canary auto-rollback:

- Without it: 24h to detect at lower percentage.
- Cost in lost quality: significant.

The 10-minute rollback prevented incidents.

### 8.7 The infrastructure

- Feature flag system (LaunchDarkly or similar).
- Canary metrics pipeline.
- Auto-decision logic.
- Dashboard.

~6 weeks initial; ongoing minimal.

### 8.8 The canary-completion rate

Per quarter:

- ~12 canary rollouts.
- ~10 succeed without intervention.
- ~2 require human intervention.
- ~0-1 rollback.

Acceptable.

### 8.9 The "we share canary infrastructure across features" benefit

Single canary system serves all features:

- One pipeline.
- One dashboard.
- One discipline.

Efficient.

### 8.10 The lessons

- Canary is non-optional.
- Auto-rollback is essential.
- Shadow useful for high-stakes; expensive routine.
- Per-workload tuning.

---

## 9. Anti-patterns

### 9.1 The 100%-deploy

**Pattern.** Skip canary; deploy to all. Disaster on issues.

**Corrective.** Canary per §2.

### 9.2 The no-rollback-criteria

**Pattern.** Canary deployed; "we'll see how it goes." Subjective in stress.

**Corrective.** Pre-defined criteria per §6.1.

### 9.3 The no-auto-rollback

**Pattern.** Rollback requires human action; canary issues persist.

**Corrective.** Auto-rollback per §6.2.

### 9.4 The "shadow without comparison" gap

**Pattern.** Shadow traffic enabled; outputs not compared.

**Corrective.** Comparison per §3.4.

### 9.5 The accelerated ramp without justification

**Pattern.** 0% → 100% in hours; high-stakes workload.

**Corrective.** Standard ramp per §5.1.

### 9.6 The "we don't measure quality in canary" miss

**Pattern.** Latency / errors monitored; quality not.

**Corrective.** Quality metric per §4.1.

### 9.7 The "we canary same model for all features at once" coordination gap

**Pattern.** Multiple features migrating simultaneously; cascade issues.

**Corrective.** Stagger per §5.7.

### 9.8 The "we never tested rollback" failure

**Pattern.** Rollback path never exercised.

**Corrective.** Pre-test in staging per [rollback-procedures.md](./rollback-procedures.md).

### 9.9 The "user-facing canary indicator" leakage

**Pattern.** Canary users get a "you're in canary" notification; UX cost.

**Corrective.** Canary should be invisible to user.

### 9.10 The "we ramp slowly; investigation hard" patience

**Pattern.** Issue at 25%; ramp held; takes weeks to investigate. Long-running canary.

**Corrective.** Decisive action per §5.5.

---

## 10. Findings (sprint-assignable)

### ML-CSR-001 — Severity: Critical
**Finding.** No canary; direct deployment.
**Recommendation.** Per §2.
**Owner.** AI platform, sprint N+1.

### ML-CSR-002 — Severity: Critical
**Finding.** Rollback criteria undefined.
**Recommendation.** Per §6.1.
**Owner.** AI platform + product, sprint N+1.

### ML-CSR-003 — Severity: Critical
**Finding.** Auto-rollback not implemented.
**Recommendation.** Per §6.2.
**Owner.** AI platform, sprint N+1.

### ML-CSR-004 — Severity: High
**Finding.** Quality not monitored in canary.
**Recommendation.** Per §4.1.
**Owner.** AI platform + eval, sprint N+2.

### ML-CSR-005 — Severity: High
**Finding.** Cost not monitored in canary.
**Recommendation.** Per §4.3.
**Owner.** FinOps + AI platform, sprint N+2.

### ML-CSR-006 — Severity: High
**Finding.** Ramp ad-hoc.
**Recommendation.** Per §5.
**Owner.** AI platform, sprint N+2.

### ML-CSR-007 — Severity: High
**Finding.** Shadow not used for high-stakes workloads.
**Recommendation.** Per §3.
**Owner.** AI platform, sprint N+2.

### ML-CSR-008 — Severity: High
**Finding.** Feature flag infrastructure absent.
**Recommendation.** Per §2.7.
**Owner.** AI platform + product, sprint N+2.

### ML-CSR-009 — Severity: Medium
**Finding.** Canary dashboard absent.
**Recommendation.** Per §4.10.
**Owner.** observability-eng, sprint N+3.

### ML-CSR-010 — Severity: Medium
**Finding.** Per-workload ramp not differentiated.
**Recommendation.** Per §5.4.
**Owner.** AI platform, sprint N+3.

### ML-CSR-011 — Severity: Medium
**Finding.** Shadow infrastructure absent.
**Recommendation.** Per §3.10.
**Owner.** AI platform, sprint N+3.

### ML-CSR-012 — Severity: Medium
**Finding.** Canary-completion review absent.
**Recommendation.** Per §6.9.
**Owner.** AI platform, sprint N+3.

### ML-CSR-013 — Severity: Medium
**Finding.** Sticky-session pattern absent.
**Recommendation.** Per §2.8.
**Owner.** AI platform, sprint N+3.

### ML-CSR-014 — Severity: Medium
**Finding.** Cross-feature coordination absent.
**Recommendation.** Per §5.7.
**Owner.** engineering management + AI platform, sprint N+4.

### ML-CSR-015 — Severity: Low
**Finding.** Canary metrics not retained for analysis.
**Recommendation.** Audit log per §7.8.
**Owner.** observability-eng, sprint N+5.

### ML-CSR-016 — Severity: Low
**Finding.** Automation human-override absent.
**Recommendation.** Per §7.4.
**Owner.** AI platform, sprint N+5.

### ML-CSR-017 — Severity: Low
**Finding.** Cost-of-shadow not tracked.
**Recommendation.** Per §3.2.
**Owner.** FinOps, sprint N+6.

### ML-CSR-018 — Severity: Low
**Finding.** Pre-canary deployment validation absent.
**Recommendation.** Per §8.5.
**Owner.** AI platform, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Build canary infrastructure (flags) per §2.7.**
- [ ] **Pre-defined rollback criteria per §6.1.**
- [ ] **Auto-rollback per §6.2.**
- [ ] **Quality + cost + latency monitoring per §4.**
- [ ] **Per-workload ramp design per §5.**
- [ ] **Build canary dashboard per §4.10.**
- [ ] **Shadow infrastructure for high-stakes per §3.10.**
- [ ] **Cross-feature coordination per §5.7.**
- [ ] **Per-canary review per §6.9.**
- [ ] **Audit logging per §7.8.**

---

## 12. References

**In this folder.**
- [model-registry.md](./model-registry.md) — version-pin for canary.
- [model-promotion.md](./model-promotion.md) — promotion flow.
- [rollback-procedures.md](./rollback-procedures.md) — rollback.
- [model-deprecation-playbook.md](./model-deprecation-playbook.md) — migration.
- [fine-tuning-operations.md](./fine-tuning-operations.md) — fine-tune deployments.

**Elsewhere in this repo.**
- [reliability-engineering/fault-budgets-for-ai.md](../reliability-engineering/fault-budgets-for-ai.md) — SLOs that trigger.
- [reliability-engineering/degraded-mode-design.md](../reliability-engineering/degraded-mode-design.md) — degraded mode during rollback.
- [observability-and-telemetry/](../observability-and-telemetry/) — observability.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / model-migration-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-migration-playbook.md) — migration as project.

**External.**
- Feature-flag platform documentation (LaunchDarkly, Statsig, Optimizely).
- Continuous delivery / canary deployment literature.
- Site Reliability Engineering — chapter on launches.
