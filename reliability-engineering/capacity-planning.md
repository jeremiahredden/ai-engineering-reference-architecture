# Capacity Planning

> **Audience.** Engineers planning capacity for AI workloads. SREs whose first capacity surprise was the provider's account-level rate limit. Anyone whose autoscaling for self-hosted inference works for CPU services but doesn't work for GPU. **Scope.** The *engineering* practice of capacity planning for AI: identifying capacity dimensions (provider TPM/RPM, GPU pool, vector store, downstream); provider rate-limit headroom monitoring; multi-tenancy of provider quotas; burst vs sustained traffic; queue-and-shed-load design; auto-scaling for self-hosted inference; capacity forecasting. Not the architecture of multi-tenant fairness (see [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)). Not the backpressure topology (see [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md)). Not the multi-provider failover (see [multi-provider-failover.md](./multi-provider-failover.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI capacity planning differs from conventional capacity planning in two specific ways:

- **The binding constraint is usually a provider's account-level rate limit, not your infrastructure.** Adding more pods doesn't help if the provider's RPM cap is what's binding.
- **GPU economics is non-linear.** A self-hosted GPU pool's throughput doesn't scale linearly with GPU count; batching efficiency, memory constraints, and KV-cache dynamics create non-obvious behavior.

These differences mean conventional capacity-planning intuitions don't always apply. A team that's used to "we'll just add pods" approach discovers AI capacity is different the first time their AI traffic doubles.

The capacity dimensions for AI:

- Provider account-level RPM / TPM (typically the binding constraint).
- GPU pool for self-hosted (if any).
- Vector store QPS / size.
- Downstream APIs consuming AI output.
- Network egress for streaming.

Each dimension has its own planning approach.

Beyond identifying the dimensions, capacity planning produces:

- Forecasts of expected vs available capacity.
- Plans for what to do when capacity is constrained.
- Multi-tenant quota management.
- Negotiations with providers for increased rate limits.

This document covers the engineering: how to identify the binding capacity dimension; how to monitor headroom; how to plan for growth; how to design queue-and-shed-load when capacity is exceeded; how to auto-scale self-hosted; how to multi-tenant a single provider quota.

This document is opinionated about four things:

1. **Identify the binding dimension first.** Capacity planning that adds pods when the provider's TPM is the bottleneck is wasted effort. Diagnose.
2. **Provider rate-limit headroom is the primary capacity metric for hosted-model workloads.** Below 20% headroom, you're at risk; below 10%, you're effectively at capacity.
3. **Multi-tenant a single provider quota with discipline.** Multiple tenants on one account means one tenant's burst exhausts everyone. Per-tenant budgets and fair-share are non-optional.
4. **GPU auto-scaling is harder than CPU; plan accordingly.** Cold-start time for GPU pods is 5-15 minutes; pre-warm strategies are needed.

Structure: (2) identifying capacity dimensions; (3) provider rate-limit headroom; (4) multi-tenancy of provider quotas; (5) burst vs sustained traffic; (6) queue-and-shed-load design; (7) auto-scaling for self-hosted; (8) capacity forecasting; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. Identifying capacity dimensions

The starting point. Most teams miss at least one.

### 2.1 The dimension list

- **Provider RPM:** requests per minute at the account level.
- **Provider TPM:** tokens per minute (input + output combined or separate).
- **Provider concurrency:** simultaneous in-flight requests.
- **GPU pool capacity:** for self-hosted.
- **Vector store QPS:** queries per second.
- **Vector store size:** index capacity.
- **Downstream API capacity:** the API the AI's output flows into.
- **Network egress:** bandwidth for streaming.
- **Compute capacity:** your own pods / workers.

### 2.2 The binding-constraint identification

Which dimension is binding? Use per-stage latency breakdown:

```
Total request latency:
  Pre-call: 50ms
  Provider call: 4000ms        ← largest contributor
  Post-call: 100ms
  Downstream: 200ms
```

The largest contributor is often the binding capacity dimension; verify with capacity metrics.

### 2.3 The "which dimension is full?" check

Each dimension has a utilization metric:

- Provider RPM headroom: % remaining.
- Provider TPM headroom: % remaining.
- GPU utilization: % busy.
- Vector store QPS: vs published capacity.
- Pod CPU / memory utilization.

When one dimension's utilization > 80%, it's the binding constraint (or about to be).

### 2.4 The "constraint moves with workload" reality

The binding constraint changes based on workload mix:

- Short-context workload: RPM is binding (lots of small requests).
- Long-context workload: TPM is binding (fewer requests, large tokens).
- Agent workload: provider concurrency may bind (long-running, in-flight).

Per-workload analysis identifies the binding dimension.

### 2.5 The "no single binding constraint" case

For very high-volume systems, multiple dimensions may be near capacity:

- Provider RPM at 70%.
- Provider TPM at 80%.
- Vector store QPS at 60%.

The order of binding changes hour to hour. Plan for headroom on all dimensions.

### 2.6 The "binding constraint is operational, not capacity" case

Sometimes the bottleneck isn't capacity:

- Latency P99 too high → not a capacity issue; a latency issue.
- Cold starts producing slow responses → not capacity; deployment issue.

Diagnose carefully; capacity isn't always the answer.

---

## 3. Provider rate-limit headroom

The primary capacity metric for hosted-model workloads.

### 3.1 The headroom metric

```
headroom_pct = (limit - consumed) / limit × 100
```

Per provider × per model:

- 90%+ headroom: comfortable; plenty of room.
- 50-90%: normal; monitor.
- 20-50%: alert; growing usage.
- 10-20%: capacity risk; plan now.
- < 10%: effectively at capacity; expect 429s.

### 3.2 The signal sources

Provider rate-limit headers (cross-link to [retry-strategy.md §5.1](./retry-strategy.md)):

- `X-RateLimit-Limit-Requests`
- `X-RateLimit-Remaining-Requests`
- `X-RateLimit-Limit-Tokens`
- `X-RateLimit-Remaining-Tokens`
- `Retry-After`

Each call returns updated values; aggregate fleet-wide.

### 3.3 The fleet-wide aggregation

Multiple consumers share one provider account. Aggregate headroom signal:

```python
def update_provider_headroom(provider, model, remaining_rpm, remaining_tpm):
    redis.set(f"provider_rpm:{provider}:{model}", remaining_rpm, ex=60)
    redis.set(f"provider_tpm:{provider}:{model}", remaining_tpm, ex=60)

def get_provider_headroom(provider, model):
    return {
        "rpm": redis.get(f"provider_rpm:{provider}:{model}") or 0,
        "tpm": redis.get(f"provider_tpm:{provider}:{model}") or 0,
    }
```

All consumers consult; consistent picture.

### 3.4 The headroom dashboard

A panel per provider × model:

- Current headroom (RPM + TPM).
- Historical trend (last 24 hours).
- Alert thresholds visible.

When headroom drops, the dashboard reflects.

### 3.5 The headroom alerts

```yaml
alerts:
  provider_headroom_low:
    trigger: headroom < 20% for 5 minutes
    severity: warning
    
  provider_headroom_critical:
    trigger: headroom < 10% for 1 minute
    severity: page
    
  provider_headroom_zero:
    trigger: 429s firing
    severity: page
```

Multi-tier; early warning + actual exhaustion.

### 3.6 The "headroom dropping fast" prediction

Time-to-exhaustion forecast:

```
drain_rate = (headroom_5min_ago - current_headroom) / 5 minutes
predicted_minutes_to_zero = current_headroom / drain_rate
```

If predicted < 30 minutes, mitigation is needed soon.

### 3.7 The headroom-aware pre-call check

Pre-call check considers headroom (cross-link to [cost-and-finops/cost-aware-rate-limiting.md §6.3](../cost-and-finops/cost-aware-rate-limiting.md)):

```python
def should_call_provider(provider, model, estimated_tokens):
    headroom = get_provider_headroom(provider, model)
    if headroom["tpm"] < estimated_tokens * 2:
        return False  # not enough headroom; defer
    return True
```

Defers calls when headroom is insufficient; avoids unnecessary 429s.

### 3.8 The "request more capacity from provider" workflow

When headroom is consistently low:

- Document usage history.
- Request higher rate limits from provider.
- Negotiate based on commit (provider often offers higher limits with commit).
- Plan ahead; rate-limit increases take days to weeks.

---

## 4. Multi-tenancy of provider quotas

Multiple tenants share one provider account; ensure fairness.

### 4.1 The shared quota problem

Provider account: 5000 RPM total. Tenants:

- Tenant A: averages 200 RPM (4% of quota).
- Tenant B: averages 500 RPM (10% of quota).
- Tenant C: averages 50 RPM (1% of quota).

If Tenant A bursts to 4500 RPM, Tenants B and C get 429s. The architecture must prevent.

### 4.2 The per-tenant budget

Each tenant has a budget:

```yaml
tenant_quotas:
  tenant-A:
    rpm: 500
    tpm: 300_000
  tenant-B:
    rpm: 800
    tpm: 500_000
  tenant-C:
    rpm: 200
    tpm: 100_000
  # Sum: 1500 RPM, 900k TPM — under the 5000 RPM / 2M TPM account quota
```

Sum allows over-provisioning (tenants rarely use full budget); fair when contended.

### 4.3 The enforcement

Per-tenant token-bucket (cross-link to [cost-and-finops/per-tenant-cost-control.md §3](../cost-and-finops/per-tenant-cost-control.md)):

```python
def acquire_tenant_quota(tenant_id, estimated_tokens):
    bucket_key = f"tenant_tpm:{tenant_id}"
    return redis.eval(LUA_BUCKET_ACQUIRE, [bucket_key], [estimated_tokens, ...])
```

Tenants respect their own buckets; one tenant's burst doesn't exhaust others.

### 4.4 The platform reserve

A portion of the provider's quota is reserved for platform use (internal copilots, infrastructure):

```
Provider account: 5000 RPM
  - Tenant pool: 4000 RPM (allocated across tenants)
  - Platform reserve: 1000 RPM (for internal)
```

Platform reserve protects internal workloads from tenant pressure.

### 4.5 The premium tier

Premium tenants get dedicated capacity (cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md §4](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)):

- Premium tier: dedicated 30% of quota; not shared with standard / free.
- Premium tenants compete only among themselves (or have individual reservations).

### 4.6 The over-subscription strategy

Sum of tenant budgets > provider quota: over-subscription. Works when:

- Tenants don't all burst simultaneously.
- Fair-share scheduling handles contention.

When all tenants do burst simultaneously, some tenants experience 429s; per-tenant SLO degrades.

### 4.7 The "Tenant A wants their dedicated quota" case

Enterprise tenants may negotiate dedicated capacity:

- Account splits: Tenant A on its own provider account.
- Reserved capacity: % of shared account reserved for tenant.

Engineering work to support; commercial term that justifies it.

---

## 5. Burst vs sustained traffic

The traffic shape matters; capacity decisions follow.

### 5.1 The burst pattern

Short-duration spike: 30 minutes of 5x normal traffic.

- Cause: launch, viral content, promotion.
- Capacity needed: peak rate × duration.
- After burst: returns to normal.

### 5.2 The sustained pattern

Permanent or long-duration increase: 2x normal traffic indefinitely.

- Cause: customer growth, new product line.
- Capacity needed: continuously elevated.
- Replan capacity as the new normal.

### 5.3 The "predictable burst" case

Daily / weekly patterns:

- Monday morning peak (10x off-hours).
- Quarterly business cycle.
- Marketing campaign launches.

Predictable; capacity planned in advance.

### 5.4 The "unpredictable burst" case

Surprise spikes:

- Customer's referral campaign goes viral.
- Misconfigured customer agent (cross-link to [cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md)).
- Security event causing all customers to hit AI simultaneously.

Capacity to absorb; or shed-load if exceeded.

### 5.5 The capacity-for-burst calculation

For sustained: capacity = expected sustained × 1.2 (20% margin).

For burst: capacity = expected sustained × burst_factor (3-10x for short bursts).

Plan for the peak; not for the average.

### 5.6 The capacity-for-bust trade-off

Sustaining higher capacity costs:

- Provider commit at higher rate-limit tier.
- More self-hosted infrastructure.

Versus shed-load during bursts:

- Lower sustained cost.
- Some user-facing impact during burst.

Pick based on cost / customer-experience trade-off.

### 5.7 The "burst exhausts headroom" detection

Real-time signal:

- Headroom drops faster than baseline.
- Per-tenant burst metric spikes.

Triggers either capacity-mode or shed-load mode (§6).

---

## 6. Queue-and-shed-load design

When capacity is exceeded, how to handle.

### 6.1 The queue option

Accept the request; queue it; process when capacity is available.

**Pros.** No lost work; latency-tolerant workloads absorb.
**Cons.** Latency grows; queue depth must be bounded.

### 6.2 The shed-load option

Reject incoming requests; return 429.

**Pros.** Fast feedback to caller; bounded latency.
**Cons.** User-facing failures.

### 6.3 The mode-switching threshold

Below queue depth threshold: queue.
Above: shed-load.

```yaml
capacity_policy:
  queue_depth_max: 10000
  queue_latency_target: 5 minutes
  shed_threshold: queue depth > 80% capacity
```

### 6.4 The priority lane in shed

When shedding, shed by priority (cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md §5](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)):

- Free-tier shed first.
- Batch workloads shed before real-time.
- Internal copilots shed before customer-facing.

### 6.5 The "queue is invisible to user" pattern

For async workloads (batch, agents), user submits and doesn't wait. Queue is the user-facing async path; not an emergency capacity response.

### 6.6 The "shed-load with structured error" pattern

When sheddding, the response:

```json
{
  "status": "service_at_capacity",
  "retry_after_seconds": 30,
  "queue_position_estimate": null,
  "alternative_actions": ["Retry in a moment", "Try again later"]
}
```

Tells caller what to do.

### 6.7 The "queue depth as health signal" interpretation

Growing queue depth is a capacity signal:

- Steady-state: queue depth stays low.
- Growing: capacity at limit; consider increasing.

Alert on growing queue depth.

---

## 7. Auto-scaling for self-hosted

GPU pools scale differently from CPU services.

### 7.1 The cold-start problem

GPU pod cold start: 5-15 minutes (download model weights, warm-up).

CPU pod cold start: seconds.

GPU auto-scaling that's reactive (scale up when load arrives) is too slow.

### 7.2 The pre-warm strategy

Maintain a small pool of pre-warmed GPU pods:

```yaml
gpu_pool:
  min_replicas: 4 (always warm)
  max_replicas: 20
  scale_up_threshold: queue depth > 5 per replica
  scale_down_threshold: queue depth = 0 for 20 minutes
  scale_up_rate: 2 replicas per minute (limited by GPU availability)
```

### 7.3 The scaling signal

Standard:
- CPU / GPU utilization (less useful for GPU; binary).
- Queue depth in inference server (vLLM exposes).
- Latency (P99 trending up).

Queue depth is usually the primary signal.

### 7.4 The "GPU shortage" reality

In some regions, GPU instances may be unavailable when needed. Mitigations:

- Multi-region GPU pool.
- Provider commit for reserved GPUs.
- Fallback to hosted provider when self-hosted exhausted.

### 7.5 The model-loading optimization

Loading model weights takes minutes. Optimizations:

- Persistent storage volume per pod (model weights pre-loaded; no download).
- Pre-baked image with model weights.
- Shared model storage (NFS or similar) accessed by multiple pods.

### 7.6 The continuous-batching benefit

Inference servers (vLLM) use continuous batching: multiple requests batched on the GPU simultaneously. Effective throughput is higher than naive batch.

Plan capacity based on effective throughput, not theoretical max.

### 7.7 The cost of pre-warm vs cold-start

Pre-warm: idle GPU costs even when not used.

Cold-start: cheaper at low traffic; higher latency on first request after scale-up.

Find the right balance per workload.

---

## 8. Capacity forecasting

Looking ahead.

### 8.1 The forecast inputs

- Historical traffic (last 6-12 months).
- Known upcoming changes (feature launches, customer onboardings).
- Seasonal patterns.
- Growth trends.

### 8.2 The forecast horizon

- Short (next quarter): operational planning.
- Medium (next 6 months): provider negotiation.
- Long (next year): infrastructure investment.

### 8.3 The capacity-vs-forecast comparison

```yaml
Q3 2026 forecast:
  Care Coordinator: 1.5M tokens/min projected (vs 1.0M today)
  Patient API: 800k tokens/min projected (vs 500k today)
  Document ingestion: 5M tokens/min projected (vs 3M today)
  
  Total: 7.3M tokens/min projected
  Provider quota: 8M tokens/min
  Headroom: 9% (low; renegotiate before Q3)
```

The forecast surfaces capacity gaps.

### 8.4 The provider-negotiation timeline

Provider rate-limit increases take time:

- Standard increase: days.
- Major increase (10x+): weeks (commitment terms negotiated).
- New provider account: weeks.

Plan accordingly.

### 8.5 The "build capacity ahead" pattern

For sustained growth, build capacity ahead:

- Buffer of 20-30% headroom maintained.
- Scale out triggers added headroom.

Versus reactive: capacity is built in response to incidents.

### 8.6 The capacity-budget alignment

Capacity costs money:

- Provider commit tier: higher rate limits at higher commit.
- Self-hosted GPUs: capital + operational.

Capacity-budget integration (cross-link to [cost-and-finops/finops-process.md](../cost-and-finops/finops-process.md)).

### 8.7 The annual capacity review

Annual:

- Forecast vs actual reconciliation.
- Provider commit renewal / renegotiation.
- Infrastructure investment for next year.

---

## 9. Worked Meridian example

Meridian's capacity planning evolved from "wait until we hit limits" to systematic forecasting.

### 9.1 The capacity catalog

```
Anthropic account (US):
  RPM: 5,000
  TPM: 2,000,000
  
Anthropic account (Premium reserved):
  RPM: 1,500 (30% of total)
  TPM: 600,000 (30%)
  
Cohere account (CA):
  RPM: 500
  TPM: 200,000
  
Self-hosted Llama cluster (HIPAA):
  GPUs: 8 × A100 80GB
  Effective throughput: ~150k tokens/min
  Auto-scaling: min 4, max 12

Pinecone vector store:
  QPS: 500
  Index size: 50M vectors
```

### 9.2 The headroom signals

Real-time:

```
Anthropic US: 2,000 RPM (40% headroom), 800k TPM (60% headroom)
Anthropic Premium: 800 RPM (47%), 250k TPM (58%)
Cohere CA: 200 RPM (60%), 80k TPM (60%)
Self-hosted: 35% GPU utilization (avg over last hour)
Pinecone: 250 QPS (50% headroom)
```

Reasonable headroom on all dimensions in normal operation.

### 9.3 The per-tenant quota structure

12 SaaS tenants share the Anthropic US account quota (the standard tier sub-pool of ~2000 RPM):

- 4 free tier: 100 RPM each (400 total).
- 5 standard: 200 RPM each (1000 total).
- 2 premium: 600 RPM each (1200 total; on the Premium-reserved sub-pool).
- 1 enterprise: on its own provider account.

Sum (standard+free): 1400 RPM under the 2000 sub-pool. Over-subscribed by ~30% (acceptable; bursts rare).

### 9.4 The Q1 2026 capacity event

A new customer onboarded; their projected daily traffic was 200k tokens.

Actual first-week traffic: 800k tokens/day (4x projected; misconfigured agent on their side).

Effect:
- Their per-tenant TPM bucket hit limits.
- They got 429s; called support.
- Mitigation: tightened their TPM limit until they fixed their agent.

Customer-facing impact: limited to that customer. Other tenants unaffected. The architecture isolated the issue.

### 9.5 The Q2 2026 forecast review

Forecast for Q3:

- Care Coordinator: growth 10% per quarter.
- Patient API chat: growth 15% per quarter (new feature launch).
- Document ingestion: growth 8%.

Projected aggregate: 1.4M TPM peak (vs 1.0M current).

Provider account quota: 2M TPM.

Headroom margin: comfortable. No provider negotiation needed.

But projected for Q4 (with new enterprise customer): 1.8M TPM. Closer to the limit. Provider negotiation initiated; rate limit increased to 3M TPM for Q4 launch.

### 9.6 The self-hosted scaling

Self-hosted Llama cluster:

- Steady state: 4 GPUs at ~60% utilization.
- Peak (overnight ingestion): 8 GPUs at 80% utilization.
- Auto-scaling: up at queue depth > 5/replica; down after 20 min idle.

Pre-warm strategy: always 4 pods warm; weights pre-loaded; scale-up adds ~2 min per pod.

Cold-start incident (early 2026): unable to scale up fast enough during a spike. Resolved by:
- Increased min pods to 4.
- Faster pre-load (NFS shared model storage).
- Cap on scale-up rate to avoid GPU shortage.

### 9.7 The capacity dashboard

A "capacity headroom" dashboard:

- Per provider, per dimension headroom.
- GPU utilization.
- Vector store QPS / utilization.
- Per-tenant consumption vs budget.

Reviewed weekly; deeper review in monthly cost / SLO meetings.

### 9.8 What the planning produces

- Provider account quotas adequate for forecast (with negotiated increases).
- Self-hosted GPU pool right-sized (not over-provisioned; not under).
- Per-tenant fairness maintained.
- No capacity-driven incidents in last 12 months despite multiple onboardings.

### 9.9 The cost of the discipline

- 0.25 FTE for capacity-planning + tracking + provider negotiation.
- Capacity-review meetings: ~2 hours / month + 8 hours quarterly review.
- Infrastructure: shared with other observability.

### 9.10 The lessons

- Provider rate limits are the primary constraint for hosted workloads.
- Per-tenant quotas are non-optional.
- GPU pool requires pre-warm; reactive scaling is too slow.
- Annual provider negotiation is part of the cadence.

---

## 10. Anti-patterns

### 10.1 The "we'll scale when we hit limits" reflex

**Pattern.** No capacity planning. First time provider 429s the entire fleet → emergency.

**Corrective.** Proactive headroom monitoring per §3.

### 10.2 The single-account multi-tenant without quotas

**Pattern.** All tenants on one provider account; no per-tenant budgets. First tenant's burst exhausts all.

**Corrective.** Per-tenant quotas per §4.

### 10.3 The "more pods solves it" assumption

**Pattern.** Add more pods. Provider RPM is still binding; pods sit idle waiting on provider.

**Corrective.** Diagnose binding constraint per §2.

### 10.4 The reactive GPU scaling

**Pattern.** GPU pool scales up on demand. Cold-start is 10 minutes; first 10 minutes of spike has no capacity.

**Corrective.** Pre-warm pool per §7.2.

### 10.5 The forecast that's just "grow at 10%"

**Pattern.** Forecast doesn't account for upcoming changes (feature launches, new customers).

**Corrective.** Inputs per §8.1; account for known changes.

### 10.6 The "we have headroom" without monitoring

**Pattern.** Belief that capacity is fine; no actual monitoring. First incident reveals belief was wrong.

**Corrective.** Headroom dashboard per §3.4; weekly review.

### 10.7 The over-subscription without fair-share

**Pattern.** Sum of tenant budgets > provider quota; no fair-share enforcement. When tenants burst together, some win, others lose.

**Corrective.** Weighted-fair scheduling per §4.

### 10.8 The "shed-load policy is unwritten"

**Pattern.** Capacity exceeded; ad-hoc decisions about who to shed. Inconsistent; political.

**Corrective.** Priority lane shed policy per §6.4.

### 10.9 The capacity that's never used

**Pattern.** Self-hosted pool provisioned for 5x current load. Idle GPUs cost money.

**Corrective.** Right-size to actual + buffer per §8.5.

### 10.10 The "provider quota is private" lack of visibility

**Pattern.** Quota information not surfaced to engineering teams; they don't know the limit.

**Corrective.** Headroom dashboard per §3.4; visible to engineering management.

---

## 11. Findings (sprint-assignable)

### REL-CAP-001 — Severity: Critical
**Finding.** No provider rate-limit headroom monitoring.
**Recommendation.** Headroom signal + dashboard per §3.
**Owner.** SRE + AI platform, sprint N+1.

### REL-CAP-002 — Severity: Critical
**Finding.** No per-tenant quotas; one tenant exhausts shared provider account.
**Recommendation.** Per-tenant quota enforcement per §4.
**Owner.** AI platform, sprint N+1.

### REL-CAP-003 — Severity: Critical
**Finding.** Reactive GPU scaling; cold-start gap.
**Recommendation.** Pre-warm pool per §7.2.
**Owner.** AI platform, sprint N+1.

### REL-CAP-004 — Severity: High
**Finding.** No headroom alerts.
**Recommendation.** Multi-tier alerts per §3.5.
**Owner.** SRE, sprint N+2.

### REL-CAP-005 — Severity: High
**Finding.** Capacity forecast absent.
**Recommendation.** Quarterly forecasting per §8.
**Owner.** SRE + AI platform, sprint N+2.

### REL-CAP-006 — Severity: High
**Finding.** Provider negotiation reactive, not proactive.
**Recommendation.** Negotiate ahead of forecast per §3.8.
**Owner.** engineering management + procurement, sprint N+2.

### REL-CAP-007 — Severity: High
**Finding.** Queue / shed-load policy undocumented.
**Recommendation.** Policy per §6.3, §6.4.
**Owner.** AI platform, sprint N+2.

### REL-CAP-008 — Severity: High
**Finding.** Pre-call headroom check not enforced.
**Recommendation.** Per §3.7.
**Owner.** AI platform, sprint N+3.

### REL-CAP-009 — Severity: Medium
**Finding.** Vector store QPS / capacity not monitored.
**Recommendation.** Per §2.1.
**Owner.** AI platform + observability, sprint N+3.

### REL-CAP-010 — Severity: Medium
**Finding.** Burst-aware capacity planning absent.
**Recommendation.** Per §5.5.
**Owner.** SRE + AI platform, sprint N+3.

### REL-CAP-011 — Severity: Medium
**Finding.** Premium tier dedicated capacity not implemented.
**Recommendation.** Per §4.5.
**Owner.** AI platform + product, sprint N+3.

### REL-CAP-012 — Severity: Medium
**Finding.** GPU auto-scaling without model-loading optimization.
**Recommendation.** Per §7.5.
**Owner.** AI platform, sprint N+3.

### REL-CAP-013 — Severity: Medium
**Finding.** Capacity-budget alignment absent.
**Recommendation.** Per §8.6.
**Owner.** FinOps + AI platform, sprint N+4.

### REL-CAP-014 — Severity: Medium
**Finding.** Annual capacity review not scheduled.
**Recommendation.** Per §8.7.
**Owner.** engineering management, sprint N+4.

### REL-CAP-015 — Severity: Low
**Finding.** Continuous-batching benefit not captured in capacity model.
**Recommendation.** Per §7.6.
**Owner.** AI platform, sprint N+5.

### REL-CAP-016 — Severity: Low
**Finding.** GPU shortage contingency absent.
**Recommendation.** Per §7.4; multi-region or fallback to hosted.
**Owner.** AI platform, sprint N+5.

### REL-CAP-017 — Severity: Low
**Finding.** Forecast reconciliation not performed.
**Recommendation.** Forecast vs actual review per §8.7.
**Owner.** SRE, sprint N+5.

### REL-CAP-018 — Severity: Low
**Finding.** Provider quota information not visible to engineering teams.
**Recommendation.** Per §3.4; engineering management visibility.
**Owner.** SRE + engineering management, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Identify capacity dimensions for the system (§2).**
- [ ] **Build provider rate-limit headroom monitoring (§3).**
- [ ] **Implement per-tenant quotas (§4).**
- [ ] **Configure GPU pre-warm pool (§7.2).**
- [ ] **Implement headroom alerts (§3.5).**
- [ ] **Build capacity-headroom dashboard.**
- [ ] **Build per-tenant consumption dashboard.**
- [ ] **Document queue / shed-load policy (§6).**
- [ ] **Build quarterly capacity forecast (§8).**
- [ ] **Implement pre-call headroom check (§3.7).**
- [ ] **Annual provider negotiation cadence (§8.4).**
- [ ] **Premium tier reserved capacity (§4.5).**
- [ ] **GPU shortage contingency plan (§7.4).**
- [ ] **Annual capacity review (§8.7).**

---

## 13. References

**In this folder.**
- [timeout-strategy.md](./timeout-strategy.md) — timeouts under capacity pressure.
- [retry-strategy.md](./retry-strategy.md) — retry that interacts with provider rate limits.
- [circuit-breakers.md](./circuit-breakers.md) — breakers that engage under capacity exhaustion.
- [fallback-patterns.md](./fallback-patterns.md) — fallback under capacity pressure.
- [degraded-mode-design.md](./degraded-mode-design.md) — degraded mode when capacity is constrained.
- [fault-budgets-for-ai.md](./fault-budgets-for-ai.md) — capacity affects availability and latency SLOs.
- [multi-provider-failover.md](./multi-provider-failover.md) *(companion)* — failover when one provider's capacity is exhausted.

**Elsewhere in this repo.**
- [cost-and-finops/cost-aware-rate-limiting.md](../cost-and-finops/cost-aware-rate-limiting.md) — rate-limit infrastructure.
- [cost-and-finops/per-tenant-cost-control.md](../cost-and-finops/per-tenant-cost-control.md) — per-tenant budgets.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — backpressure topology.
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — multi-tenant capacity allocation.

**External.**
- Anthropic / OpenAI rate-limit documentation.
- vLLM continuous batching documentation.
- Kubernetes HPA / VPA (limited usefulness for GPU; consider KEDA).
- AWS capacity planning best practices.
