# Cost-Aware Rate Limiting

> **Audience.** Engineers whose rate-limit infrastructure measures RPS but doesn't know about cost. SREs whose first $400 spike-incident was a tenant respecting their request rate limit and exhausting the cost budget anyway. Anyone whose budget enforcement happens at the invoice stage instead of the call stage. **Scope.** The *engineering* practice of rate-limiting that targets cost rather than just request count: cost-rate dimensions (TPM, $/period, per-tenant + per-feature); priority lanes under contention; interaction with provider rate limits; the pre-call cost check architecture. Not the broader budget enforcement (see [per-tenant-cost-control.md](./per-tenant-cost-control.md), companion). Not the cost circuit-breaker on threshold (see [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md)). Not the queue-topology architecture (see [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Traditional rate limits are request-count based: "100 requests per second per user." That model has worked for web APIs for decades. It does not work for AI APIs.

The problem: AI calls have wildly variable cost. A single LLM call with 200 input tokens and 50 output tokens costs $0.0008. A single LLM call with 80,000 input tokens (a long-context analysis) and 4,000 output tokens costs $0.30. Both are "one request." A rate limit measured in RPS can't distinguish them.

A tenant respecting a 60 RPM limit can:
- Spend $0.05/minute on small calls.
- Spend $30/minute on long-context calls.
- Spend somewhere in between based on the input mix.

The 60 RPM limit is honored throughout, but cost varies by 600x.

The traditional response — "we'll add stricter RPM" — doesn't fix it. Lower RPM constrains the small-call workload unnecessarily while still allowing expensive workloads to overspend. The right response is cost-aware rate limiting: limits that target tokens-per-minute and $-per-period directly.

This document covers the engineering: what dimensions of cost-rate to track, how to enforce them, how priority lanes work under cost pressure, how to compose cost-rate limits with provider-side rate limits (which are themselves a constraint), and how the pre-call cost check actually works in the request path.

This document is opinionated about four things:

1. **RPS / RPM alone is insufficient for AI rate limiting.** A request-count limit doesn't catch cost overruns from long-context or expensive-output calls. TPM and $-per-period are non-optional.
2. **Cost rate limits must be per-tenant AND per-feature.** A tenant's budget can be exhausted by one feature; a feature's budget can be exhausted by one tenant. Both dimensions need protection.
3. **Pre-call cost estimation is what makes cost-rate limits work.** Without it, the system can only react after the spend; with it, the system rejects in advance.
4. **Priority lanes are a contract, not a label.** "Premium gets more budget under contention" must mean something measurable. Implement specific behavior; don't ship vague priority terms.

Structure: (2) the cost-rate dimensions; (3) cost-aware rate limiting per tenant; (4) cost-aware rate limiting per feature; (5) priority lanes under contention; (6) interaction with provider rate limits; (7) the pre-call cost check architecture; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption sequencing; (12) references.

---

## 2. The cost-rate dimensions

What to track and limit.

### 2.1 Tokens per minute (TPM)

Tokens are the unit the provider charges for; TPM is the natural cost-proxy rate.

```python
tpm_budget = 600_000  # tenant's TPM allocation
```

Enforced via token bucket: each call consumes (input_tokens + output_tokens) from the bucket; calls blocked when bucket is empty.

**Why it matters.** A small-call workload at 100 RPM consumes ~20k TPM (200 tokens/call avg); a long-context workload at 100 RPM consumes ~10M TPM (100k tokens/call avg). RPM-only limit can't distinguish.

**Granularity.** Per-minute is standard; per-second for high-volume workloads.

### 2.2 Dollars per period

The most direct cost-rate. Comes in flavors:

- **$/minute.** Catches very fast spikes.
- **$/hour.** Catches sustained anomalies.
- **$/day.** Standard per-tenant budget rollup.
- **$/month.** Long-term envelope.

```python
budgets = {
    "cost_per_minute_usd": 2,      # very fast anomaly cap
    "cost_per_hour_usd": 50,       # sustained anomaly
    "cost_per_day_usd": 200,       # daily envelope
    "cost_per_month_usd": 4500,    # monthly envelope
}
```

Multi-window: each is enforced independently; exceeding any one blocks the call.

### 2.3 Request rate (RPM/RPS) — still needed

Despite the limits of RPS, it's still useful:

- Catches simple bursts (someone retrying in a tight loop).
- Provider-side rate limits are typically RPM-based; respect them.
- Some operational concerns (concurrency, queue depth) scale with RPS.

Don't drop RPS limits; combine them with cost-rate limits.

### 2.4 Per-model rates

Different models have different costs. A "TPM budget" expressed in tokens doesn't capture the per-token cost difference between Opus ($15/1M output) and Haiku ($1.25/1M output).

For multi-model workloads, prefer $-based budgets. Or: per-model TPM budgets that account for the model's rate.

### 2.5 Per-endpoint rates

Real-time and batch endpoints have different rates (batch is 50%). Cost-rate enforcement must use the appropriate rate per call.

Cross-link to [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md).

### 2.6 Cost-bearing resources beyond LLM

Vector store QPS, downstream API calls — also cost-bearing. Rate-limit them per cost:

- Vector store: ~$0.0001 per query (varies by provider).
- Embedding generation: ~$0.0001 per call.
- Per-tenant downstream API costs.

Aggregate cost across all cost-bearing resources for a complete picture.

### 2.7 The dimension matrix

```
For tenant T, current period:
  Calls used: 8,234 of 30k/day               [RPM-based, classic]
  Tokens used: 4.2M of 8M TPM-allocation     [TPM-based, cost-proxy]
  Dollars used: $87 of $200/day              [$-based, direct]
  Provider RPM headroom: 23% remaining       [Provider constraint]
```

A call must pass all four checks to be authorized.

---

## 3. Cost-aware rate limiting per tenant

The per-tenant layer.

### 3.1 The per-tenant token bucket for $

```python
def acquire_cost_budget(tenant_id: str, estimated_cost_usd: float, periods: list[str]) -> bool:
    """
    Acquire cost budget across multiple period windows.
    All-or-nothing: if any window doesn't have headroom, none consumed.
    """
    pipeline = redis.pipeline()
    for period in periods:
        key = f"cost:{tenant_id}:{period}"
        budget = get_budget(tenant_id, period)
        # Use atomic Lua to check-and-decrement across all periods
        pipeline.eval(LUA_CHECK_AND_DECREMENT, [key], [estimated_cost_usd, budget])

    results = pipeline.execute()
    return all(r == 1 for r in results)
```

Multi-window enforcement; rollback on partial failure.

### 3.2 The pre-call estimate

Pre-call cost is estimated; actual is reconciled post-call.

```python
def pre_call_estimate(model: str, prompt: str, max_tokens: int) -> float:
    input_tokens = count_tokens(prompt, model)
    output_tokens_estimate = min(max_tokens, average_output_for_model(model))
    rate = get_rate(model)
    return (input_tokens * rate.input + output_tokens_estimate * rate.output) * 1.5  # safety margin
```

Cross-link to [per-tenant-cost-control.md](./per-tenant-cost-control.md) §4 for pre-flight estimation discipline.

### 3.3 The post-call reconciliation

```python
def post_call_reconcile(tenant_id: str, request_id: str, actual_cost_usd: float, estimated_cost_usd: float):
    delta = actual_cost_usd - estimated_cost_usd
    if delta > 0:
        # Actual exceeded estimate; debit the difference
        consume_extra(tenant_id, delta)
    elif delta < 0:
        # Actual was less; refund
        refund(tenant_id, abs(delta))
```

The estimate is approximate; reconciliation corrects.

### 3.4 The estimate-accuracy SLI

Track:

```python
estimate_actual_ratio = actual_cost / estimated_cost
```

Target close to 1.0; consistently >1.0 means safety margin is too low; <1.0 means margin is too high.

### 3.5 The "long-context call would exhaust budget" check

For a single call:

```python
def is_call_authorized(tenant_id, estimated_cost):
    remaining = get_remaining_budget(tenant_id, "day")
    if estimated_cost > remaining:
        return False  # single call exceeds remaining; refuse
    if estimated_cost > remaining * 0.5:
        # single call would consume half of remaining; warn but allow
        log_warning(tenant_id, "high-cost-call-consumes-half-remaining")
        return True
    return True
```

Single calls that disproportionately consume budget are flagged.

### 3.6 The per-tenant TPM bucket

Same token-bucket pattern, with tokens as units:

```python
def acquire_tpm_tokens(tenant_id: str, estimated_tokens: int) -> bool:
    return redis.eval(LUA_BUCKET_ACQUIRE,
                      [f"tpm:{tenant_id}"],
                      [estimated_tokens, tpm_budget_for(tenant_id), refill_rate(tenant_id)])
```

TPM bucket refills at the TPM budget rate (e.g., 600,000 / 60 = 10,000 tokens per second).

### 3.7 The hierarchical bucket

Per-tenant + per-feature: the call must pass both:

```python
def authorize_call(tenant_id, feature, estimated_cost, estimated_tokens):
    tenant_ok = (
        acquire_cost_budget(tenant_id, estimated_cost, ["day", "month"])
        and acquire_tpm_tokens(tenant_id, estimated_tokens)
    )
    feature_ok = (
        acquire_feature_budget(feature, estimated_cost)
        and acquire_feature_tpm(feature, estimated_tokens)
    )
    if not (tenant_ok and feature_ok):
        # Rollback if one passed but other didn't
        rollback_partial(tenant_id, feature, estimated_cost, estimated_tokens)
        return False
    return True
```

Both layers; both enforced.

---

## 4. Cost-aware rate limiting per feature

The per-feature layer prevents one feature from monopolizing capacity.

### 4.1 The per-feature budget

```yaml
features:
  care-coordinator:
    cost_per_day_usd: 1500
    cost_per_month_usd: 35000
    tpm: 3_000_000
    priority: high

  patient-api-chat:
    cost_per_day_usd: 400
    cost_per_month_usd: 10000
    tpm: 800_000
    priority: high

  document-ingestion:
    cost_per_day_usd: 2000
    cost_per_month_usd: 50000
    tpm: 8_000_000
    priority: medium

  analytics-warehouse-copilot:
    cost_per_day_usd: 800
    cost_per_month_usd: 18000
    tpm: 1_500_000
    priority: low

  internal-copilot:
    cost_per_day_usd: 100
    cost_per_month_usd: 2000
    tpm: 200_000
    priority: low
```

Each feature has its own budget; consumption is tracked separately.

### 4.2 The "feature budget exhausted" handling

Same per-feature policy as per-tenant (cross-link to [per-tenant-cost-control.md](./per-tenant-cost-control.md) §6):

- fail-open with degraded mode.
- fail-closed with structured error.
- queue.
- escalate.

Choice is feature-specific.

### 4.3 The aggregation discipline

The sum of feature budgets does not equal a platform-wide budget. Features can overlap; aggregate platform budget is set separately.

The platform-wide budget is:
- The provider rate limit (account-level).
- The financial limit (whatever the engineering team has agreed).

Sum of feature budgets is allowed to exceed (over-provisioned for normal case); enforcement when contention arises.

### 4.4 The "new feature on launch" budget

When launching a new feature, assign it a starting budget; review after a month of data; adjust.

Without an assigned budget, the feature is "unbudgeted" — alert if it appears in spend dashboards without a budget entry.

### 4.5 The feature → owner mapping

Each feature budget has an owner team:

```yaml
features:
  care-coordinator:
    cost_per_day_usd: 1500
    owner_team: clinical-ai-team
```

When the feature's budget is approaching, the owner team is notified (cross-link to [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md)).

---

## 5. Priority lanes under contention

When aggregate capacity is constrained, priority determines who gets what.

### 5.1 The priority dimensions

For each call, priority is determined by:

- Tenant tier (premium > standard > free).
- Feature priority (high > medium > low).
- Call type (interactive > batch).
- Time-sensitivity (clinical > recreational).

The architecture computes an effective priority per call.

### 5.2 The contention scenarios

Contention happens when:

- Provider rate limit binds (account at TPM or RPM cap).
- Self-hosted GPU at full utilization.
- Aggregate platform budget hit.
- Multiple features contending for shared capacity.

Under contention, lower-priority calls are blocked or queued; higher-priority calls proceed.

### 5.3 The weighted-share allocation

Under contention, capacity is divided by priority:

```
Total available capacity (when contended): X TPM
Priority weights:
  high:   weight 4
  medium: weight 2
  low:    weight 1
  
high:    capacity × 4/7 ≈ 57%
medium:  capacity × 2/7 ≈ 29%
low:     capacity × 1/7 ≈ 14%
```

Weighted-fair queuing across priority bands; each band's share is reserved.

### 5.4 The premium dedicated reserve

Premium tenants may have reserved capacity that's not shared:

```yaml
premium_tenant_X:
  dedicated_tpm: 500_000  # this much is reserved; nobody else can use
  burst_tpm: 1_500_000     # peak burst budget
```

Even under contention, premium has the reserved capacity. Cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 5.5 The "priority is contract" enforcement

If the contract says "premium gets priority," it must mean something measurable. Examples:

- Reserved 20% of capacity.
- Weight 4x in priority queue.
- Skip to front of queue under contention.
- Maintained latency SLO under contention.

Vague priority terms degrade to "we tried" in the first incident.

### 5.6 The starvation prevention

Strict priority can starve low-priority indefinitely. Avoid:

- **Weighted instead of strict.** Every priority band gets some allocation; high just gets more.
- **Aging.** Calls waiting too long get priority bump.
- **Floor capacity.** Reserve minimum capacity for each priority band; low-priority always gets at least N TPM.

Starvation prevention isn't usually litigated, but the incident where free-tier customers all simultaneously timeout is worth avoiding.

---

## 6. Interaction with provider rate limits

The provider's own rate limits are another constraint to manage.

### 6.1 The provider rate-limit signal

Provider rate-limit headers (Anthropic, OpenAI, etc.):

- `X-RateLimit-Limit-Requests`
- `X-RateLimit-Remaining-Requests`
- `X-RateLimit-Limit-Tokens`
- `X-RateLimit-Remaining-Tokens`
- `Retry-After` (when rate-limited)

Read these to know where the provider's headroom is.

### 6.2 The provider rate-limit budget

Maintain fleet-wide visibility of provider headroom (Redis-backed):

```python
def update_provider_headroom(provider, model, remaining_rpm, remaining_tpm):
    redis.set(f"provider_rpm:{provider}:{model}", remaining_rpm, ex=60)
    redis.set(f"provider_tpm:{provider}:{model}", remaining_tpm, ex=60)
```

Each call updates from the response headers; fleet-wide view aggregates.

### 6.3 The check-against-provider-headroom

Before submitting a call:

```python
def can_call_now(provider, model, estimated_tokens):
    provider_tpm_headroom = redis.get(f"provider_tpm:{provider}:{model}") or 0
    return estimated_tokens < float(provider_tpm_headroom) * 0.9  # leave 10% headroom
```

If headroom is low, defer / queue. Avoids hitting 429s.

### 6.4 The 429 backoff

When a 429 fires from the provider:

- Read `Retry-After`.
- Back off the fleet (not just this consumer): pause all consumers via shared signal.
- Resume after Retry-After elapses.

```python
def on_provider_429(provider, retry_after_seconds):
    redis.set(f"provider_pause:{provider}", "1", ex=retry_after_seconds)

def is_paused(provider):
    return redis.get(f"provider_pause:{provider}") == "1"
```

Cross-link to [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md §4.4](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md).

### 6.5 The provider-tier multiplexing

Different model tiers have different provider rate limits. Routing calls across tiers (cross-link to [tier-routing-for-cost.md](./tier-routing-for-cost.md)) effectively multiplexes the provider's per-tier rate limits.

### 6.6 The multi-region / multi-account multiplexing

For very high volume, splitting across provider regions or provider accounts increases total rate limit:

- Anthropic account A: 5000 RPM, 2M TPM.
- Anthropic account B (separate billing): another 5000 RPM, 2M TPM.

Combined: 10000 RPM, 4M TPM.

Some teams maintain multiple accounts; the architecture's job is to route calls across them.

---

## 7. The pre-call cost check architecture

How the check actually fits in the request path.

### 7.1 The order of operations

```
Request arrives
    ↓
Authentication / authorization
    ↓
Tenant identification
    ↓
Per-tenant policy lookup (budget, tier, etc.)
    ↓
Estimate cost (input tokens × rate + estimated output tokens × rate)
    ↓
Pre-call check: budget? TPM? RPM? Feature budget? Provider headroom?
    ↓
If denied: return 429 with structured error.
If approved: reserve estimated cost.
    ↓
Provider call
    ↓
Post-call: reconcile actual vs estimate; release / debit.
    ↓
Return response.
```

Each check is fast (Redis ops; sub-millisecond). Total overhead: ~5-10ms for the rate-limit layer.

### 7.2 The fast-path optimization

For tenants well under their budget, skip expensive checks:

```python
def pre_call_check(tenant_id, feature, estimated_cost):
    if is_well_under_budget(tenant_id):
        # Fast path: log only; no Redis check
        return ApproveResult(authorized=True)
    return full_check(tenant_id, feature, estimated_cost)
```

"Well under" might be: < 50% of daily budget AND < 50% of TPM. Saves Redis ops for non-contentious tenants.

### 7.3 The local cache for budget state

Repeated reads of "what's tenant X's budget" benefit from local cache:

```python
@cached(ttl=10)  # 10s TTL
def get_tenant_budget(tenant_id):
    return redis.get(f"budget:{tenant_id}")
```

Budget rarely changes; 10s staleness is acceptable. Reduces Redis load.

### 7.4 The "soft refusal" pattern

For some workloads, refusal isn't acceptable. Pattern:

- If budget exhausted, try fallback model (cheaper).
- Fallback model fits within remaining budget.
- Customer gets degraded but functional response.

```python
def call_with_fallback(tenant_id, feature, primary_model, fallback_model, prompt):
    if pre_call_check(tenant_id, feature, estimate_cost(primary_model, prompt)):
        return call(primary_model, prompt)
    
    if pre_call_check(tenant_id, feature, estimate_cost(fallback_model, prompt)):
        return call(fallback_model, prompt)
    
    raise BudgetExhausted("No model fits remaining budget")
```

Cross-link to [tier-routing-for-cost.md](./tier-routing-for-cost.md).

### 7.5 The structured-error response

When cost-rate limit fires, the response is structured:

```json
{
  "status": "rate_limited",
  "limit_type": "tpm_per_tenant",
  "limit_value": 600000,
  "consumed_value": 600000,
  "resets_at": "2026-05-27T14:00:00Z",
  "retry_after_ms": 47000,
  "alternative_actions": [
    "Retry after the suggested time",
    "Reduce request size",
    "Contact account manager to discuss limit increase"
  ]
}
```

Customer can act based on the structured fields.

### 7.6 The audit log

Every pre-call check is logged (at minimum on denial):

```json
{
  "timestamp": "...",
  "tenant_id": "...",
  "feature": "...",
  "request_id": "...",
  "decision": "denied",
  "denial_reason": "tpm_per_tenant_limit",
  "estimated_cost_usd": 0.45,
  "current_consumption": {
    "cost_today_usd": 198.20,
    "cost_today_limit_usd": 200,
    "tpm_consumed": 600000,
    "tpm_limit": 600000
  }
}
```

Useful for tenant dispute investigation; useful for tuning.

---

## 8. Worked Meridian example

Meridian's cost-aware rate limiting prevents per-tenant runaway and per-feature overruns.

### 8.1 The dimensions tracked

For each tenant:

- API RPM (gateway-enforced).
- LLM RPM (per the consumer-side bucket).
- LLM TPM.
- $/day.
- $/month.
- Per-feature TPM and $/day.

For each feature:

- TPM platform-wide.
- $/day platform-wide.
- $/month platform-wide.

For the platform:

- Provider RPM (Anthropic account total).
- Provider TPM (Anthropic account total).

### 8.2 The pre-call check overhead

Median pre-call check latency: ~3ms.
P99: ~12ms.

The check operates against Redis (10-node cluster); each call does ~5 atomic Lua operations.

Total request latency overhead: ~5ms; absorbed in the budget for the LLM call (which is seconds).

### 8.3 The per-feature budget catalog (excerpt)

```yaml
features:
  care-coordinator:
    cost_per_day_usd: 1500
    cost_per_month_usd: 35000
    tpm: 3_000_000
    priority: high
    on_exhaust: escalate  # business-hours engineer

  patient-api-chat-us:
    cost_per_day_usd: 400
    cost_per_month_usd: 10000
    tpm: 800_000
    priority: high
    on_exhaust: fail_open_degraded  # smaller model fallback

  patient-api-chat-canada:
    cost_per_day_usd: 100
    cost_per_month_usd: 2500
    tpm: 200_000
    priority: high
    on_exhaust: fail_open_degraded

  document-ingestion:
    cost_per_day_usd: 2000
    cost_per_month_usd: 50000
    tpm: 8_000_000  # batch path; higher allocation
    priority: medium
    on_exhaust: queue

  document-embedding:
    cost_per_day_usd: 300  # self-hosted; small marginal cost
    tpm: 5_000_000
    priority: medium
    on_exhaust: queue

  analytics-warehouse-copilot:
    cost_per_day_usd: 800
    cost_per_month_usd: 18000
    tpm: 1_500_000
    priority: low
    on_exhaust: queue
```

### 8.4 The Q1 2026 incident: long-context burst

A standard-tier tenant deployed a workflow that sent unusually long-context queries (each ~30k tokens). The tenant's API-RPM budget (500 RPM) was respected. The tenant's LLM-RPM budget (200 RPM) was respected.

But the TPM budget (300k/min for standard tier) was being consumed at 6M TPM (200 calls × 30k tokens). The TPM bucket emptied in seconds; subsequent calls were 429'd by the consumer-side check.

Without TPM enforcement, the tenant would have exhausted the provider account's TPM (2M TPM) within seconds — affecting all tenants.

The TPM enforcement at the per-tenant level isolated the impact. Customer-success contacted the tenant; the workflow was redesigned to chunk the long-context queries.

### 8.5 The Q2 2026 incident: provider rate-limit spike

Anthropic's RPM rate-limit briefly dropped during a provider-side incident. The fleet's provider-headroom signal updated within seconds. Consumers paused submissions where headroom was insufficient.

During the 8-minute incident:

- Care Coordinator: continued at reduced rate; priority preserved.
- Patient API chat: fallback to cached responses for low-priority queries.
- Document ingestion: queued; processed after incident.
- Analytics workloads: queued; processed after incident.

The incident was customer-invisible. Cross-link to [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md).

### 8.6 The priority lane in action

During a Friday-afternoon load spike (clinical hours peak), aggregate platform capacity approached limits:

- Care Coordinator and Patient API (high priority): served at full rate.
- Document ingestion (medium priority): queued; latency increased from 30 min to 90 min.
- Analytics workloads (low priority): queued; latency increased from 30 min to 4 hours.

Tenants in lower-priority workloads saw expected degradation; high-priority workloads were unaffected.

### 8.7 The infrastructure cost

- Redis cluster (10 nodes): $4k/month.
- Engineering: ~3 weeks to build out the multi-dimensional enforcement.
- Ongoing: ~5% of platform team's time.

### 8.8 The impact

- Zero incidents where one tenant's cost overrun affected another tenant in 18 months.
- Zero incidents where one feature's runaway exhausted platform capacity.
- Provider rate-limit incidents handled gracefully; customer-invisible.
- Per-tenant 429 events: ~5-10 per month (all from tenant-specific misconfigurations; expected).

---

## 9. Anti-patterns

### 9.1 The RPM-only rate limit

**Pattern.** Rate limits are RPS / RPM only. Long-context workload consumes 100x the cost per request; RPM doesn't catch it.

**Corrective.** Add TPM and $-based budgets per §2.

### 9.2 The "we'll add cost limits later" deferral

**Pattern.** RPM limit at launch; cost limit "for v2." First cost-spike incident is the trigger; v2 is rushed.

**Corrective.** Cost-rate limits from day one. Defaults from tier; refinement over time.

### 9.3 The pre-call estimate that's static

**Pattern.** Cost estimate ignores actual input tokens; uses a constant. Long-context calls bypass effective enforcement.

**Corrective.** Tokenize the prompt; estimate based on actual size per §3.2.

### 9.4 The post-call reconciliation that's missing

**Pattern.** Pre-call estimate is used; actual cost is never reconciled. Over-estimates leave headroom unused; under-estimates leave overruns uncaught.

**Corrective.** Post-call reconciliation per §3.3.

### 9.5 The single-window budget

**Pattern.** Daily budget only. A 10-minute spike that consumes the daily budget rapidly goes uncaught until the cap is hit.

**Corrective.** Multi-window per §2.2 (per-minute, per-hour, per-day, per-month).

### 9.6 The provider rate-limit ignored

**Pattern.** Consumer doesn't read provider rate-limit headers. 429 storms during provider pressure.

**Corrective.** Provider-headroom signal per §6.2 and §6.3.

### 9.7 The priority that's just a label

**Pattern.** Premium tier is documented as "priority" but produces no measurable difference in routing or capacity allocation.

**Corrective.** Priority must change behavior measurably per §5.5.

### 9.8 The starvation of low-priority

**Pattern.** Strict priority queue; low-priority calls never complete during sustained high-priority load.

**Corrective.** Weighted-fair queueing + floor capacity per §5.6.

### 9.9 The unstructured 429 response

**Pattern.** "Rate limited" with no detail. Caller can't decide what to do.

**Corrective.** Structured error per §7.5.

### 9.10 The cost check that's slow

**Pattern.** Pre-call check adds 50ms to every call. P99 latency degraded; cost-check overhead is itself a problem.

**Corrective.** Redis-backed atomic ops + local caching + fast-path per §7.

---

## 10. Findings (sprint-assignable)

### COST-RATE-001 — Severity: Critical
**Finding.** Only RPM rate limits enforced; cost overruns possible at allowed RPM.
**Recommendation.** Add TPM and $-based limits per §2; multi-dimensional enforcement.
**Owner.** AI platform, sprint N+1.

### COST-RATE-002 — Severity: Critical
**Finding.** No pre-call cost estimate.
**Recommendation.** Pre-call estimate per §3.2 with safety margin; required for cost enforcement.
**Owner.** AI platform, sprint N+1.

### COST-RATE-003 — Severity: Critical
**Finding.** Provider rate-limit headers not consumed; 429s when headroom was visible in advance.
**Recommendation.** Provider headroom signal per §6.2; pre-call check considers it per §6.3.
**Owner.** AI platform + SRE, sprint N+1.

### COST-RATE-004 — Severity: Critical
**Finding.** No per-feature budget.
**Recommendation.** Per-feature budget catalog per §4; enforcement at pre-call.
**Owner.** AI platform + product, sprint N+1.

### COST-RATE-005 — Severity: High
**Finding.** No post-call reconciliation.
**Recommendation.** Post-call reconciliation per §3.3; estimate-actual ratio SLI.
**Owner.** AI platform, sprint N+2.

### COST-RATE-006 — Severity: High
**Finding.** Single-window budget; spikes within window go uncaught.
**Recommendation.** Multi-window per §2.2 (minute, hour, day, month).
**Owner.** AI platform, sprint N+2.

### COST-RATE-007 — Severity: High
**Finding.** Premium tier has no enforceable priority.
**Recommendation.** Priority lanes per §5; reserved capacity for premium per §5.4.
**Owner.** AI platform + product, sprint N+2.

### COST-RATE-008 — Severity: High
**Finding.** Strict priority configuration starves low-priority.
**Recommendation.** Weighted-fair queueing + floor capacity per §5.6.
**Owner.** AI platform, sprint N+2.

### COST-RATE-009 — Severity: High
**Finding.** 429 responses are unstructured; callers can't decide action.
**Recommendation.** Structured error response per §7.5.
**Owner.** AI platform + API team, sprint N+2.

### COST-RATE-010 — Severity: High
**Finding.** Pre-call check adds significant latency.
**Recommendation.** Redis-backed atomic ops + local caching + fast-path per §7.
**Owner.** AI platform, sprint N+3.

### COST-RATE-011 — Severity: Medium
**Finding.** Audit log of cost-rate decisions absent.
**Recommendation.** Log denials and approvals per §7.6; queryable for tenant disputes.
**Owner.** AI platform + observability, sprint N+3.

### COST-RATE-012 — Severity: Medium
**Finding.** "Soft refusal" fallback to cheaper model not implemented.
**Recommendation.** Fallback pattern per §7.4; cross-link to tier-routing.
**Owner.** AI platform, sprint N+3.

### COST-RATE-013 — Severity: Medium
**Finding.** Per-call cost ignored for non-LLM resources (vector store, embedding, downstream).
**Recommendation.** Aggregate cost per §2.6; rate-limit all cost-bearing resources.
**Owner.** AI platform, sprint N+3.

### COST-RATE-014 — Severity: Medium
**Finding.** Provider 429 doesn't pause the fleet.
**Recommendation.** Fleet-wide pause signal per §6.4.
**Owner.** AI platform, sprint N+4.

### COST-RATE-015 — Severity: Medium
**Finding.** Multi-account / multi-region provider rate-limit multiplexing not implemented.
**Recommendation.** Route across accounts / regions for high-volume workloads per §6.6.
**Owner.** AI platform, sprint N+4.

### COST-RATE-016 — Severity: Low
**Finding.** Cost-estimate accuracy SLI absent.
**Recommendation.** Track estimate-actual ratio per §3.4; alert on drift.
**Owner.** AI platform + observability, sprint N+5.

### COST-RATE-017 — Severity: Low
**Finding.** Fast-path optimization not deployed.
**Recommendation.** Fast-path for well-under-budget tenants per §7.2.
**Owner.** AI platform, sprint N+5.

### COST-RATE-018 — Severity: Low
**Finding.** Aging / priority bump not implemented.
**Recommendation.** Aging per §5.6 to prevent indefinite starvation.
**Owner.** AI platform, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Implement per-tenant TPM bucket (§3.6).** Redis-backed; atomic.
- [ ] **Implement per-tenant $-based budget (§3.1).** Daily + monthly; atomic check-and-decrement.
- [ ] **Implement pre-call cost estimate (§3.2).** Token-count input; estimate output.
- [ ] **Implement post-call reconciliation (§3.3).** Actual vs estimate; refund / debit delta.
- [ ] **Implement per-feature budget (§4).** Cost + TPM per feature; aggregate enforcement.
- [ ] **Read provider rate-limit headers (§6.1, §6.2).** Maintain fleet-wide headroom signal.
- [ ] **Pre-call check provider headroom (§6.3).** Defer if headroom low.
- [ ] **Implement priority lanes (§5).** Weighted-fair; floor capacity; aging.
- [ ] **Define structured 429 response (§7.5).**
- [ ] **Audit log cost decisions (§7.6).**
- [ ] **Fast-path optimization (§7.2).** Reduce Redis load.
- [ ] **Local cache for budget state (§7.3).** 10s TTL.
- [ ] **Per-tenant 429 → fleet provider 429 distinction.** Different recovery paths.
- [ ] **Pre-production test:** synthetic burst from one tenant; verify isolation.
- [ ] **Quarterly review:** estimate-actual ratio; tune safety margin.

---

## 12. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — per-call attribution that feeds reconciliation.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) — companion; budget enforcement covers cost-rate.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — circuit-breaker on budget exhaustion.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) — alerts on rate-limit denials.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — soft-refusal fallback pattern.
- [caching-for-cost.md](./caching-for-cost.md) — caching reduces rate-limit pressure.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) — batch endpoints have separate rate limits.

**Elsewhere in this repo.**
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alerting design including rate-limit metrics.
- [agent-engineering/agent-cost-control.md](../agent-engineering/agent-cost-control.md) — agent-side cost control.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — backpressure architecture; cost-rate limiting is a flavor of backpressure.
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant isolation including cost-rate dimension.

**External.**
- Redis Lua scripting documentation — atomic enforcement.
- Anthropic / OpenAI rate-limit header documentation.
- Token-bucket algorithm literature.
- Weighted-fair queueing networking literature (Demers, Keshav, Shenker).
- IETF Rate-Limit Headers draft.
