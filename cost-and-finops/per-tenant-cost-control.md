# Per-Tenant Cost Control

> **Audience.** Engineers building per-tenant budget and rate-limit infrastructure for multi-tenant AI platforms. SREs whose first noisy-tenant incident is in next week's calendar. Finance partners who want chargeback that actually works. **Scope.** The *engineering* practice of per-tenant cost control: the budget data model; enforcement mechanisms (token-bucket, pre-flight estimation, real-time tracking, circuit-breaker integration); premium-tier configuration; budget-exceeded handling (fail-open / fail-closed / human-escalate); chargeback integration; customer-facing budget API. Not the architectural design of multi-tenant fairness (see [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)). Not the circuit-breaker primitive (see [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md)). Not the broader cost attribution (see [cost-attribution.md](./cost-attribution.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Per-tenant cost control is the financial isolation primitive for multi-tenant AI. Without it, one tenant's runaway agent shows up on your AWS bill, not theirs; one tenant's burst exhausts shared capacity; one tenant's misconfigured automation generates thousands of dollars before anyone notices. The first noisy-tenant incident at every multi-tenant AI platform makes the case for per-tenant cost control; the engineering question is how to build it before the incident, not after.

The companion architecture document ([noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)) covers the design. This document covers the implementation: what the budget data model looks like, how enforcement actually works, what happens when a budget is exceeded, and how the cost rolls up to invoices.

The engineering challenges:

- **Real-time enforcement.** Each LLM call must check budget headroom before committing. Pre-flight estimation has to be accurate; post-call reconciliation has to be reliable.
- **Multiple budget dimensions.** A tenant has a daily $ budget *and* a monthly $ budget *and* a TPM rate *and* an RPM rate. Each must be enforced; exceeding any one blocks the call.
- **Tier-based defaults with per-tenant overrides.** Most tenants inherit tier defaults; some have negotiated specific budgets. The data model handles both cleanly.
- **Graceful degradation vs hard fail.** Some features should fail-open with a degraded response when budget is exceeded; some should fail-closed with a clear error; some should escalate to a human. The choice is per-feature.
- **Chargeback that doesn't lie.** Per-tenant cost numbers must reconcile with provider invoices. Customers will dispute; the engineering side must produce defensible numbers.
- **Customer self-service.** Tenants want to query their own budgets, set their own internal sub-budgets (per-user, per-feature), and configure alert thresholds. The API matters.

This document is opinionated about four things:

1. **Multi-dimensional budgets are non-negotiable.** A tenant respecting RPM can still exhaust TPM; a tenant within TPM can still exhaust $/day. Single-dimension enforcement leaves the other dimensions unprotected.
2. **Pre-flight cost estimation is required for high-cost workloads.** Post-call attribution catches the spend after it happened; pre-flight catches it before. For agent loops and long-context calls, pre-flight prevents catastrophic single-call overruns.
3. **The budget-exceeded handling is a per-feature decision, not a platform default.** Customer-facing chat should degrade gracefully; clinical workflows should fail-closed with explicit error; analytics should queue for retry. Don't pick one default; pick per-feature.
4. **Per-tenant cost reconciliation against vendor invoices must be < 2% drift.** Anything more and chargeback disputes consume support capacity. The engineering investment to get under 2% is justified by the ongoing support cost it avoids.

Structure: (2) the budget data model; (3) the enforcement mechanisms; (4) pre-flight cost estimation; (5) tier configuration; (6) budget-exceeded handling; (7) chargeback integration; (8) the customer-facing budget API; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The budget data model

What the per-tenant budget catalogue looks like at the data layer.

### 2.1 The tenant budget record

```python
@dataclass
class TenantBudget:
    tenant_id: str
    tier: str  # "free", "standard", "premium", "enterprise"

    # Cost budgets
    cost_daily_usd: float
    cost_monthly_usd: float

    # Rate budgets
    api_rpm: int
    api_rpm_burst: int  # short-term burst limit
    llm_rpm: int
    llm_tpm: int

    # Per-resource budgets (where applicable)
    vector_qps: int
    downstream_write_rps: int
    gpu_concurrent_long_context: int

    # Behavior under exhaustion
    budget_exceeded_policy: str  # "fail_open_degraded", "fail_closed_error", "queue", "escalate"

    # Premium overrides
    premium_features: list[str]  # e.g., ["dedicated_capacity", "priority_routing"]

    # Audit
    last_updated_at: datetime
    last_updated_by: str
    effective_from: datetime
    effective_until: datetime | None  # null = open-ended
```

Every tenant has exactly one active budget record. Historical budgets are kept for audit.

### 2.2 The tier defaults

Tier defaults are version-controlled config:

```yaml
tiers:
  free:
    cost_daily_usd: 5
    cost_monthly_usd: 50
    api_rpm: 60
    llm_rpm: 30
    llm_tpm: 50_000
    vector_qps: 10
    downstream_write_rps: 5
    budget_exceeded_policy: fail_closed_error

  standard:
    cost_daily_usd: 100
    cost_monthly_usd: 2500
    api_rpm: 500
    llm_rpm: 200
    llm_tpm: 300_000
    vector_qps: 100
    downstream_write_rps: 50
    budget_exceeded_policy: fail_open_degraded

  premium:
    cost_daily_usd: 500
    cost_monthly_usd: 12_000
    api_rpm: 2000
    llm_rpm: 1000
    llm_tpm: 1_500_000
    vector_qps: 500
    downstream_write_rps: 200
    budget_exceeded_policy: escalate
    premium_features: [dedicated_capacity, priority_routing]
```

New tenants get tier defaults at onboarding; overrides come later as needed.

### 2.3 Per-tenant overrides

When a tenant has specific negotiated terms:

```python
def get_effective_budget(tenant_id: str) -> TenantBudget:
    tenant = fetch_tenant(tenant_id)
    base = tier_defaults[tenant.tier]
    overrides = fetch_overrides(tenant_id)  # may be empty
    return merge(base, overrides)
```

The merge respects override fields; everything else inherits.

### 2.4 The current-consumption record

Separate from the budget, the current-period consumption:

```python
@dataclass
class TenantConsumption:
    tenant_id: str
    period: str  # "daily", "monthly"
    period_start: datetime
    period_end: datetime

    cost_spent_usd: float
    api_calls_count: int
    llm_calls_count: int
    llm_tokens_consumed: int
    vector_queries: int
    downstream_writes: int

    last_updated_at: datetime  # for monitoring lag
```

Consumption records reset at period boundaries. Daily resets at UTC midnight (configurable); monthly resets on calendar month boundary.

### 2.5 The cost-bearing event record

Per-call records that feed consumption aggregation:

```python
@dataclass
class CostBearingEvent:
    tenant_id: str
    user_id: str
    feature: str
    timestamp: datetime
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    request_id: str
    cached: bool  # was this a cache hit?
```

Cross-link to [cost-attribution.md](./cost-attribution.md) for the attribution pattern.

### 2.6 The pre-flight authorization record

For pre-flight check, a temporary "authorized but not yet consumed" record:

```python
@dataclass
class PreflightAuthorization:
    tenant_id: str
    request_id: str
    estimated_cost_usd: float
    authorized_at: datetime
    expires_at: datetime  # short TTL, e.g., 60s
```

Authorized amount is debited from headroom; the actual cost reconciles afterward (refund if less, no-op if more, but in either case the authorization expires).

### 2.7 The storage tier

- **Real-time read/write:** Redis or DynamoDB. Sub-millisecond reads for pre-call checks. Sharded by tenant_id.
- **Aggregation tier:** SQL DB or columnar store (BigQuery, ClickHouse) for analytics, dashboards, chargeback.
- **Audit tier:** append-only log (S3, immutable storage) for compliance.

Multi-tier because real-time enforcement needs speed; analytics needs query flexibility; audit needs immutability.

---

## 3. The enforcement mechanisms

How the budgets actually constrain behavior at runtime.

### 3.1 The pre-call check

Before any LLM call:

```python
def pre_call_check(context: RequestContext, estimated_cost: float) -> AuthResult:
    budget = get_effective_budget(context.tenant_id)
    consumption = get_consumption(context.tenant_id)

    # Multi-dimensional check
    if consumption.cost_spent_usd + estimated_cost > budget.cost_daily_usd:
        return AuthResult(authorized=False, reason="daily_cost_limit_exceeded")

    if consumption.llm_tokens_consumed + estimate_tokens(...) > budget.llm_tpm * 60:  # per-minute budget
        return AuthResult(authorized=False, reason="tpm_limit_exceeded")

    if not acquire_rpm_token(context.tenant_id):
        return AuthResult(authorized=False, reason="rpm_limit_exceeded")

    # Reserve the estimated cost
    authorize_preflight(context.tenant_id, context.request_id, estimated_cost)

    return AuthResult(authorized=True)
```

Multi-dimensional check ensures no single dimension goes unchecked.

### 3.2 The post-call reconciliation

After the LLM call:

```python
def post_call_reconcile(context: RequestContext, actual_cost: float):
    consume(context.tenant_id, context.request_id, actual_cost)

    # Refund the pre-flight authorization
    refund_preflight(context.tenant_id, context.request_id)

    # Update consumption
    update_consumption(context.tenant_id, actual_cost)

    # Emit cost-bearing event
    emit_event(context, actual_cost)
```

The reconcile step is non-blocking on the request path (the response can return before reconcile completes), but eventually-consistent reconcile is required for correct accounting.

### 3.3 The token-bucket for rate limits

RPM and TPM enforcement uses token buckets. Redis-backed for shared state across consumers:

```python
def acquire_rpm_token(tenant_id: str) -> bool:
    return redis.eval("""
        local bucket = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local last_refill = tonumber(redis.call("hget", bucket, "last_refill") or now)
        local tokens = tonumber(redis.call("hget", bucket, "tokens") or capacity)

        local elapsed = math.max(0, now - last_refill)
        tokens = math.min(capacity, tokens + (elapsed * refill_rate))

        if tokens >= 1 then
            tokens = tokens - 1
            redis.call("hset", bucket, "tokens", tokens, "last_refill", now)
            return 1
        else
            redis.call("hset", bucket, "tokens", tokens, "last_refill", now)
            return 0
        end
    """, ..., capacity, refill_rate, now)
```

Atomic operation; no race conditions across consumer pods.

### 3.4 The cost-tracker for $-based budgets

Cost tracking is similar but with USD:

```python
def consume(tenant_id: str, request_id: str, cost_usd: float) -> bool:
    return redis.eval("""
        local daily_key = KEYS[1]
        local monthly_key = KEYS[2]
        local cost = tonumber(ARGV[1])
        local daily_limit = tonumber(ARGV[2])
        local monthly_limit = tonumber(ARGV[3])

        local daily = tonumber(redis.call("get", daily_key) or 0)
        local monthly = tonumber(redis.call("get", monthly_key) or 0)

        if daily + cost > daily_limit or monthly + cost > monthly_limit then
            return 0  -- Reject
        end

        redis.call("incrbyfloat", daily_key, cost)
        redis.call("incrbyfloat", monthly_key, cost)
        return 1
    """, daily_key, monthly_key, cost, daily_limit, monthly_limit)
```

Daily and monthly keys have TTL set to expire at period boundary.

### 3.5 The fleet-wide budget contention

Many concurrent consumers may try to acquire from the same tenant's budget. Lua scripts make Redis operations atomic, but contention exists.

**Pattern.** Per-tenant Redis key sharded across nodes for high-volume tenants. The very largest tenants get their own dedicated Redis instance.

**Pattern.** Local caching with periodic sync. Consumers maintain a small local budget allotment (e.g., 5% of fleet allocation) that's refilled from Redis periodically. Reduces Redis traffic; accepts brief over-allocation.

### 3.6 The cross-region enforcement

For multi-region deployments, per-tenant budgets can be:

- **Global.** Single budget across regions; one Redis cluster (typically in one region) is the source of truth. Latency-sensitive; cross-region calls add ~50-200ms.
- **Per-region.** Each region has its own budget for the tenant; budgets sum to the tenant's total. Faster; loses some flexibility.
- **Hybrid.** Most of the budget allocated per-region; small reserve held centrally for cross-region balancing.

For most use cases, per-region with periodic rebalancing is the right trade-off.

### 3.7 The "downstream resource" budgets

Vector store and downstream write budgets work like rate limits on other resources. The application wraps the resource call:

```python
def vector_search(context: RequestContext, query: Embedding) -> list[Match]:
    if not acquire_vector_qps_token(context.tenant_id):
        raise BudgetExceeded("vector_qps_limit_exceeded")
    return underlying_vector_search(query)
```

The wrapper enforces; the underlying call only runs if budget allows.

---

## 4. Pre-flight cost estimation

For high-cost workloads, pre-flight estimation prevents single-call catastrophes.

### 4.1 The pre-flight estimation formula

```python
def estimate_cost(model: str, input_tokens: int, expected_output_tokens: int) -> float:
    rates = get_rates(model)
    estimated_input_cost = input_tokens * rates.per_input_token
    estimated_output_cost = expected_output_tokens * rates.per_output_token
    return estimated_input_cost + estimated_output_cost
```

The `input_tokens` is counted from the actual prompt; `expected_output_tokens` is workload-specific (max_tokens parameter, historical average, or default).

### 4.2 The safety margin

Estimates can be wrong; output sometimes exceeds expectation. Apply a safety margin:

```python
def estimate_with_margin(model: str, input_tokens: int, expected_output_tokens: int) -> float:
    base_estimate = estimate_cost(model, input_tokens, expected_output_tokens)
    return base_estimate * 1.5  # 50% safety margin
```

A 50% margin is typical; over-reserve to absorb estimate errors.

### 4.3 The agent pre-flight

For agent workloads, a single agent task may produce many LLM calls. Pre-flight estimates the total:

```python
def estimate_agent_task_cost(task: AgentTask) -> float:
    estimated_calls = expected_agent_calls_for_task_type(task.type)  # e.g., 5-10
    avg_call_cost = average_call_cost_for_task_type(task.type)  # historical
    return estimated_calls * avg_call_cost * 1.5  # safety margin
```

Tenants whose remaining budget cannot cover the estimated task cost are refused before the agent starts.

### 4.4 The customer-facing pre-flight

For UI-visible pre-flight, show the customer:

```
"This task is estimated to cost approximately $0.85.
Your remaining daily budget is $47.32.
Proceed?"
```

The customer confirms or cancels. Useful for expensive operations; not used for routine calls.

### 4.5 The estimate accuracy review

Track estimate vs actual:

```python
metric: cost_estimate_accuracy
  estimate_actual_ratio = actual_cost / estimated_cost
  // Should be approximately 1.0
```

If estimates are consistently 30% low, increase the safety margin. If consistently high, reduce.

### 4.6 The pre-flight expiry

Pre-flight authorizations expire (e.g., 60s):

- Authorization is debited from headroom immediately.
- The actual call must happen within the TTL.
- If the call doesn't happen, the authorization expires and the headroom is refunded.

Prevents indefinite holds on tenant budget by abandoned requests.

---

## 5. Tier configuration

Tier defaults are the starting point; per-tenant overrides handle exceptions.

### 5.1 The tier hierarchy

Tiers are ordered (free < standard < premium < enterprise). Tenant moves between tiers via:

- **Onboarding decision.** New customer's contract specifies the tier.
- **Upgrade.** Customer's contract changes; tenant moves up.
- **Downgrade.** Customer's contract reduces; tenant moves down.

Each tier change is logged and reversible.

### 5.2 The override approval flow

Per-tenant overrides require approval:

- **Engineering-level overrides** (e.g., temporary capacity for a customer test): approved by platform team lead.
- **Commercial-level overrides** (e.g., enterprise customer with negotiated budgets): approved by commercial + platform.

All overrides documented in the tenant budget audit log.

### 5.3 The temporary override pattern

For short-term needs (a customer's burst expected, a one-off load test):

```yaml
override:
  tenant_id: tenant-x
  effective_from: 2026-06-01T00:00:00Z
  effective_until: 2026-06-07T23:59:59Z
  cost_daily_usd: 1000  # temporary increase from 100
  reason: "annual planning workload; agreed with customer success"
```

Auto-expires at `effective_until`; tenant returns to tier defaults.

### 5.4 The premium-feature toggle

Premium tier may enable features that aren't budget-related:

- `dedicated_capacity`: tenant has reserved provider capacity (cross-link to [noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md)).
- `priority_routing`: tenant's requests get queue priority.
- `enhanced_monitoring`: tenant has access to detailed monitoring API.
- `dedicated_model_tier`: tenant's requests use a more expensive model by default.

These features are tag-driven; the application checks the tag before applying the behavior.

### 5.5 The grandfathered budget

Some long-term customers have legacy budgets that don't match current tier defaults. The data model accommodates:

- The tenant's budget is a full per-tenant override (not inheritance from current tier).
- Annual review may renegotiate.

### 5.6 The "tier doesn't fit" customer

Occasionally a customer doesn't fit any tier (e.g., needs enterprise-level $/month but premium-tier RPM). The architecture supports:

- Tier defaults + selective override.
- Custom tier (rare; only for very large customers).

Avoid proliferation of custom tiers; they multiply maintenance.

---

## 6. Budget-exceeded handling

What happens when a tenant's budget is exhausted is a per-feature decision.

### 6.1 The four handling modes

**Mode A: fail-open with degraded mode.** The request proceeds, but with a degraded path (cached response, smaller model, simpler prompt). User sees some result, possibly with a "degraded" indicator. Best for UX-driven features (chat, search).

**Mode B: fail-closed with structured error.** The request is rejected; a structured error returned. User sees a "limit exceeded" message. Best for safety-critical features (clinical, financial) where degraded is unsafe.

**Mode C: queue for later.** Request is queued; processed when budget resets (next day, next minute). Best for batch workloads where latency tolerance is high.

**Mode D: escalate to human.** Request is paused; a human (account manager, on-call) is notified and decides. Best for premium tenants where automatic refusal damages relationship.

### 6.2 The per-feature decision

```yaml
features:
  patient-chat:
    budget_exceeded_policy: fail_open_degraded
    degraded_path: cached_response_with_disclaimer

  clinical-decision-support:
    budget_exceeded_policy: fail_closed_error
    error_message: "Clinical decision support is unavailable; please contact administrator."

  bulk-classification:
    budget_exceeded_policy: queue
    queue_max_wait: 24h

  premium-tenant-clinical:
    budget_exceeded_policy: escalate
    escalation: account_manager
```

The choice is made by the feature owner; documented; tested.

### 6.3 The two-threshold policy (warning + hard)

Most teams use two thresholds:

- **Warning (80%):** notification; no enforcement change.
- **Hard limit (100%):** enforcement applies.

Between warning and hard, the tenant has time to react.

```python
def check_consumption_status(tenant_id: str) -> Status:
    consumption = get_consumption(tenant_id)
    budget = get_effective_budget(tenant_id)

    usage_fraction = consumption.cost_spent_usd / budget.cost_daily_usd

    if usage_fraction >= 1.0:
        return Status.AT_HARD_LIMIT
    elif usage_fraction >= 0.80:
        return Status.AT_WARNING
    else:
        return Status.NORMAL
```

### 6.4 The grace-period pattern

Some workloads benefit from a grace period: after hard limit, allow one more call (or up to a small overrun) to complete in-flight work. Avoids killing operations mid-stride.

```python
def check_with_grace(tenant_id: str, estimated_cost: float) -> AuthResult:
    consumption = get_consumption(tenant_id)
    budget = get_effective_budget(tenant_id)

    grace_overrun_pct = 0.05  # 5% grace
    grace_limit = budget.cost_daily_usd * (1 + grace_overrun_pct)

    if consumption.cost_spent_usd + estimated_cost > grace_limit:
        return AuthResult(authorized=False, reason="hard_limit_exceeded")

    if consumption.cost_spent_usd + estimated_cost > budget.cost_daily_usd:
        return AuthResult(authorized=True, warning="in_grace_period")

    return AuthResult(authorized=True)
```

Grace period is short (5%); after that, hard fail.

### 6.5 The in-flight work handling

When budget is exceeded, in-flight work (already started agent tasks, queued requests) needs handling:

- **Let in-flight complete.** Agent tasks already running finish; new requests refused. Cleanest tenant experience.
- **Kill in-flight at limit.** Agent tasks at next step boundary check budget; abort if exceeded. Tighter cost control; rougher experience.
- **Hybrid.** In-flight has its own bounded budget; once exceeded, that aborts too.

Default: let in-flight complete; refuse new. Premium tenants may have different policy.

### 6.6 The error envelope

When fail-closed fires, the structured error:

```json
{
  "status": "limit_exceeded",
  "limit_type": "daily_cost_limit",
  "limit_value": 100.00,
  "consumed_value": 100.00,
  "resets_at": "2026-05-28T00:00:00Z",
  "alternative_actions": [
    "Wait until daily reset",
    "Contact account manager to discuss limit increase"
  ],
  "documentation_url": "https://docs.meridian.example.com/limits"
}
```

The customer's UI can render meaningfully.

---

## 7. Chargeback integration

Per-tenant cost rolls up to invoices. The engineering side produces defensible numbers.

### 7.1 The chargeback statement

A monthly per-tenant chargeback statement:

```
Tenant: meridian-southwest
Period: 2026-05-01 to 2026-05-31

Cost by feature:
  - Care Coordinator:        $42,150.00
  - Patient API Chat:        $18,400.00
  - Analytics Copilot:       $ 9,750.00
  Total LLM cost:            $70,300.00

Cost by model:
  - Anthropic Sonnet 4.6:    $58,200.00
  - Anthropic Haiku 4.5:     $ 5,100.00
  - Self-hosted Llama 3 70B: $ 4,000.00
  - Embedding (BGE-large):   $ 3,000.00
  Total:                     $70,300.00

Reconciliation:
  Sum of per-call costs:     $70,300.00
  Vendor invoices (Anthropic): $63,400.00 (Sonnet + Haiku)
  Vendor infrastructure (self-hosted): $7,000.00 prorated
  Reconciliation drift: -$100 (0.14%; within tolerance)
```

Customer sees this; can verify; question if they disagree.

### 7.2 The reconciliation discipline

Monthly reconciliation against vendor invoices:

```
For each vendor (Anthropic, OpenAI, AWS Bedrock):
  - Sum of per-call costs attributed to this vendor.
  - Vendor invoice total.
  - Drift: (per_call_sum - invoice) / invoice.
  - Target: |drift| < 2%.
```

Drift > 2% triggers investigation. Common causes:

- Attribution missed some calls (wrapper bypass).
- Rate table outdated.
- Cached calls counted incorrectly.
- Cross-tenant attribution error.

### 7.3 The chargeback API

Tenants can query their chargeback data:

```http
GET /api/v1/account/billing/statement?period=2026-05
GET /api/v1/account/billing/usage?period=2026-05&breakdown=feature
GET /api/v1/account/billing/usage?period=2026-05&breakdown=user
GET /api/v1/account/billing/usage?period=2026-05&breakdown=model
```

JSON responses; downloadable PDF / CSV.

### 7.4 The dispute resolution path

When a tenant disputes a charge:

- Customer support pulls the per-call records for the period.
- Each call has tenant_id, user_id, feature, model, input_tokens, output_tokens, cost.
- Customer can verify which calls drove the cost.
- Disputes are usually resolved by showing the call detail.

If the dispute is valid (attribution error), correction is made; rolled into next period's statement.

### 7.5 The internal chargeback variant

For internal use (one team within the company is using AI; cost attributed to their cost center):

- Each internal team is a "tenant" with their own budget.
- Monthly chargeback is internal financial movement, not customer-facing.
- Same engineering infrastructure; different UI.

### 7.6 The "show me cost projection" feature

Tenants want to project end-of-month cost based on current burn:

```python
def project_eom_cost(tenant_id: str) -> float:
    consumption = get_consumption(tenant_id)
    days_elapsed = current_day_of_month()
    days_total = days_in_current_month()

    return consumption.cost_spent_usd * (days_total / days_elapsed)
```

Shown in the customer dashboard; useful for budget planning.

---

## 8. The customer-facing budget API

Tenants self-serve their own budget management.

### 8.1 The query endpoints

```http
GET /api/v1/account/quotas
GET /api/v1/account/usage/current
GET /api/v1/account/usage/historical?period=2026-05
GET /api/v1/account/budget/projections
```

Return current budgets, current consumption, historical usage, end-of-month projection.

### 8.2 The internal sub-budget configuration

Tenants want to set per-user or per-feature sub-budgets within their tenant budget:

```http
POST /api/v1/account/sub-budgets
Body: {
  "scope": "user",
  "scope_id": "user-abc",
  "cost_daily_usd": 10
}
```

Enforces a sub-budget within the tenant's allocation. Useful for the tenant to limit a specific user / department.

### 8.3 The alert configuration

```http
POST /api/v1/account/alerts
Body: {
  "trigger": "daily_cost_usage > 80%",
  "channels": ["email:admin@example.com", "webhook:https://example.com/billing"]
}
```

The tenant configures their own alert thresholds and channels.

### 8.4 The budget-change request

```http
POST /api/v1/account/budget-change-request
Body: {
  "requested_cost_daily_usd": 500,
  "reason": "expanding to additional clinics",
  "effective_date": "2026-06-15"
}
```

Initiates a budget-change request that goes through account-manager approval. The customer doesn't directly change their own budget; they request a change.

### 8.5 The webhooks

Webhook notifications for budget events:

- `budget.warning_threshold_reached`
- `budget.hard_limit_reached`
- `budget.daily_reset`
- `budget.monthly_reset`
- `budget.change_approved`

Tenants subscribe; their internal systems can react (e.g., halt automation when warning fires).

---

## 9. Worked Meridian example

Meridian's per-tenant cost control supports 12 paying tenants ranging from free trial to enterprise. The infrastructure has prevented multiple incidents.

### 9.1 The tenant budget catalog

```yaml
tier_defaults:
  free:
    cost_daily_usd: 5
    cost_monthly_usd: 50
    llm_rpm: 30
    llm_tpm: 50_000
    budget_exceeded_policy: fail_closed_error
  standard:
    cost_daily_usd: 100
    cost_monthly_usd: 2500
    llm_rpm: 200
    llm_tpm: 300_000
    budget_exceeded_policy: fail_open_degraded
  premium:
    cost_daily_usd: 500
    cost_monthly_usd: 12_000
    llm_rpm: 1000
    llm_tpm: 1_500_000
    budget_exceeded_policy: escalate
    premium_features: [dedicated_capacity, priority_routing]
  enterprise:
    # Per-customer negotiated; example:
    cost_daily_usd: 5000
    cost_monthly_usd: 150_000
    llm_rpm: 5000
    llm_tpm: 8_000_000
    budget_exceeded_policy: escalate
    premium_features: [dedicated_provider_account, dedicated_vector_index]

tenants:
  - id: meridian-southwest
    tier: standard
  - id: meridian-coastal
    tier: premium
  - id: meridian-regional-system
    tier: enterprise
    overrides:
      cost_monthly_usd: 200_000  # negotiated
  - id: atlantic-maple-health
    tier: standard
    overrides:
      region: ca-central-1
  - id: free-trial-tenant-x
    tier: free
  # ... 7 others
```

### 9.2 The enforcement infrastructure

- Redis cluster (3-node): per-tenant token buckets and consumption counters. ~150 RPS at peak.
- AWS Bedrock cost rates: fetched from a config table; updated when Anthropic publishes price changes.
- Pre-flight estimation: applies to all Care Coordinator agent tasks (typical task is 5-10 LLM calls; $0.40-0.85 estimated).
- Post-call reconciliation: async writes to PostgreSQL aggregation tier; Redshift for analytics.

### 9.3 The incident: free-tier runaway agent (January 2026)

A free-trial tenant's misconfigured agent recursively called itself. Within 18 minutes:

- The pre-flight estimation caught that one recursive task estimated $2 (the agent's prompt template grew with each iteration).
- The pre-flight authorization tried to debit $2 from the $5 daily budget.
- Initially passed; agent ran.
- Next iteration: pre-flight estimated $3.20; passed; ran.
- Next iteration: pre-flight estimated $4.80; pre-flight checked total consumption + estimate = $9 > $5 limit; FAIL_CLOSED.
- Agent ran ~3 iterations totaling $4.30.
- Cost circuit-breaker prevented further runs.

Customer-success contacted the tenant; tenant fixed the agent. Total spend: $4.30. Without per-tenant cost control: estimated $200-500 in the same window.

### 9.4 The incident: premium tenant's bulk operation (March 2026)

A premium tenant ran a bulk operation: re-process 30k documents through the Care Coordinator agent.

- Pre-flight on the bulk: 30k tasks × avg $0.65 = $19.5k estimated.
- Tenant's daily budget: $500. Monthly: $12k.
- Pre-flight check: would exceed monthly within hours.
- `escalate` policy fired: account manager notified.
- AM contacted customer; agreed on a temporary monthly override of $25k.
- Bulk completed over 2 days; total spend $18.2k.

The escalation prevented a denial of the customer's legitimate bulk operation while ensuring the spend was approved.

### 9.5 The reconciliation discipline

Monthly reconciliation against Anthropic Bedrock invoice:

- April 2026: per-call sum $58,200; invoice $58,089; drift 0.19%.
- May 2026: per-call sum $61,400; invoice $61,150; drift 0.41%.

Both within < 2% tolerance. Drift comes from minor rounding and timing differences (per-call records timestamp doesn't exactly match Bedrock's billing timestamp).

### 9.6 The chargeback adoption

Meridian's internal teams use the chargeback for accountability:

- Each AI feature's team has a monthly cost budget assigned by engineering leadership.
- The chargeback statement is reviewed in monthly engineering business reviews.
- Features that exceed budget are reviewed; cause identified; mitigation planned.

This discipline has kept platform-wide AI spend within 5% of budget for 8 consecutive months.

### 9.7 What the infrastructure cost to build

- 4 weeks: data model, Redis-backed token buckets, basic pre-flight check.
- 3 weeks: reconciliation pipeline, vendor invoice ingestion, monthly statement generation.
- 2 weeks: customer-facing API, dashboard, sub-budget configuration.
- 1 week: webhook delivery, alert configuration.
- Total: ~10 weeks of 1.5 engineers.

Ongoing: ~5% of platform team's time for maintenance, tenant onboarding, reconciliation review.

### 9.8 What the infrastructure prevents

- Zero documented uncapped cost incidents on tenant runaway agents (5+ caught and capped in 18 months).
- Zero customer-facing budget surprises (warning + hard threshold pattern).
- Reconciliation drift < 0.5% on all monthly cycles.
- Tenant disputes: 3 in 18 months; all resolved via per-call detail walk-through.

Estimated avoided cost: $400-800k/year in uncapped tenant incidents + ongoing support burden reduction.

---

## 10. Anti-patterns

### 10.1 The "single dimension budget" pattern

**Pattern.** Per-tenant RPM only. Tenant respects RPM but exhausts TPM; other tenants get 429s from provider.

**Corrective.** Multi-dimensional budgets per §2 (RPM, TPM, $/day, $/month, vector QPS, downstream write).

### 10.2 The "we'll add budgets when we have a problem" deferral

**Pattern.** No per-tenant budgets at launch. First multi-tenant incident is the trigger; budgets added during a P0.

**Corrective.** Per-tenant budgets from launch. Defaults at tier; refinement over time.

### 10.3 The cost tracker that's not atomic

**Pattern.** Cost tracking uses GET-MODIFY-WRITE without atomicity. Concurrent calls race; under-counting common; over-charging possible.

**Corrective.** Atomic Redis operations (Lua scripts) per §3.3 and §3.4.

### 10.4 The pre-flight estimate that's always wrong

**Pattern.** Pre-flight estimates ignore output_tokens; or assume fixed call count for agents; or don't update with actual usage. Real cost is 2-5x estimate; safety margin not enough.

**Corrective.** Estimate accuracy review per §4.5; tune the margin and the model.

### 10.5 The reconciliation drift ignored

**Pattern.** Monthly reconciliation drift is 8%; "close enough." Customer disputes amount to half a person's time per month.

**Corrective.** Drift target < 2%; investigate sources; close the gap.

### 10.6 The cross-region budget that doesn't sync

**Pattern.** Multi-region deployment; per-region budgets sum to less than tenant's total; tenant constrained more than they should be.

**Corrective.** Cross-region rebalancing per §3.6; or document the per-region tradeoff.

### 10.7 The chargeback statement that's wrong

**Pattern.** Chargeback statement shows numbers that don't match dashboard or invoice. Customer trust eroded.

**Corrective.** Source-of-truth discipline. Per-call records are authoritative; everything (dashboard, statement, invoice reconciliation) joins to them.

### 10.8 The "premium gets nothing different" labeling

**Pattern.** Premium tier has higher budget; otherwise identical. The "priority routing" feature isn't actually implemented; "dedicated capacity" is a label.

**Corrective.** Premium features must change runtime behavior measurably per §5.4.

### 10.9 The budget-change request without audit

**Pattern.** Account managers can change tenant budgets via Slack DM. No audit; no approval flow; cost surprises.

**Corrective.** Budget changes go through formal request flow per §8.4; logged; approved.

### 10.10 The customer-visible budget that's behind reality

**Pattern.** Customer dashboard updates every hour; customer hits limit; sees "you've used 50% of budget"; confused.

**Corrective.** Real-time or near-real-time customer dashboard (max lag 1 minute for usage); webhook for limit events.

---

## 11. Findings (sprint-assignable)

### COST-PTC-001 — Severity: Critical
**Finding.** No per-tenant cost budgets enforced.
**Recommendation.** Implement multi-dimensional per-tenant budgets per §2 and §3; tier defaults; hard enforcement on production tenants.
**Owner.** AI platform + FinOps, sprint N+1.

### COST-PTC-002 — Severity: Critical
**Finding.** Single-dimension budget (RPM only); TPM exhaustion possible.
**Recommendation.** Add TPM and $/day budgets per §2.2; enforce all three.
**Owner.** AI platform, sprint N+1.

### COST-PTC-003 — Severity: Critical
**Finding.** Pre-flight estimation not implemented for agent workloads.
**Recommendation.** Pre-flight per §4; especially for agent tasks where single tasks can be $0.50+.
**Owner.** AI platform, sprint N+1.

### COST-PTC-004 — Severity: Critical
**Finding.** Cost tracking not atomic; race conditions cause under-counting.
**Recommendation.** Atomic Redis Lua scripts per §3.3 and §3.4.
**Owner.** AI platform, sprint N+1.

### COST-PTC-005 — Severity: High
**Finding.** Budget-exceeded policy is platform-default; not per-feature.
**Recommendation.** Per-feature policy per §6.2; document; test.
**Owner.** product + AI platform, sprint N+2.

### COST-PTC-006 — Severity: High
**Finding.** No warning threshold; tenants surprised by hard-limit hits.
**Recommendation.** Two-threshold pattern per §6.3 (warning at 80%, hard at 100%).
**Owner.** AI platform, sprint N+2.

### COST-PTC-007 — Severity: High
**Finding.** No monthly reconciliation against vendor invoices.
**Recommendation.** Monthly reconciliation per §7.2; target drift < 2%; investigate >2%.
**Owner.** AI platform + finance, sprint N+2.

### COST-PTC-008 — Severity: High
**Finding.** Customer-facing budget API absent; tenants can't self-serve.
**Recommendation.** Query / configure / alert endpoints per §8.
**Owner.** product + AI platform, sprint N+2.

### COST-PTC-009 — Severity: High
**Finding.** Chargeback statements not generated.
**Recommendation.** Monthly statement per §7.1; queryable detail; PDF + CSV download.
**Owner.** AI platform + finance + product, sprint N+2.

### COST-PTC-010 — Severity: Medium
**Finding.** Premium tier features (dedicated capacity, priority routing) labeled but not implemented.
**Recommendation.** Implement premium features per §5.4; verify with synthetic load.
**Owner.** AI platform + product, sprint N+3.

### COST-PTC-011 — Severity: Medium
**Finding.** No sub-budget configuration (per-user, per-feature) within tenant.
**Recommendation.** Sub-budget API per §8.2.
**Owner.** AI platform + product, sprint N+3.

### COST-PTC-012 — Severity: Medium
**Finding.** No webhook notifications for budget events.
**Recommendation.** Webhook subscriptions per §8.5; tenants can react in their systems.
**Owner.** AI platform, sprint N+3.

### COST-PTC-013 — Severity: Medium
**Finding.** Pre-flight authorization doesn't expire; abandoned authorizations hold budget.
**Recommendation.** TTL (60s default) per §4.6; auto-refund on expiry.
**Owner.** AI platform, sprint N+3.

### COST-PTC-014 — Severity: Medium
**Finding.** Cross-region budgets not designed; tenants constrained more than necessary.
**Recommendation.** Per-region with rebalancing or global with cross-region budget Redis per §3.6.
**Owner.** AI platform, sprint N+4.

### COST-PTC-015 — Severity: Medium
**Finding.** In-flight work killed mid-stride on budget exceedance.
**Recommendation.** Default policy: let in-flight complete; refuse new (§6.5).
**Owner.** AI platform, sprint N+4.

### COST-PTC-016 — Severity: Low
**Finding.** Budget change requests via informal channels; no audit.
**Recommendation.** Formal API + approval flow per §8.4; logged; audit-queryable.
**Owner.** product + AI platform, sprint N+5.

### COST-PTC-017 — Severity: Low
**Finding.** Cost projection not surfaced to customer.
**Recommendation.** Projection endpoint per §7.6; dashboard panel.
**Owner.** product + AI platform, sprint N+5.

### COST-PTC-018 — Severity: Low
**Finding.** Internal team chargeback not differentiated from customer chargeback.
**Recommendation.** Use same infrastructure with internal-mode UI per §7.5; engineering teams have monthly budget reviews.
**Owner.** engineering leadership, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Implement data model (§2).** Tenant budgets, consumption records, cost-bearing events.
- [ ] **Choose tier defaults (§2.2).** Free / standard / premium / enterprise.
- [ ] **Build Redis-backed token bucket (§3.3).** Atomic Lua scripts.
- [ ] **Build cost-tracker (§3.4).** Daily + monthly; atomic.
- [ ] **Implement pre-flight check (§3.1).** Multi-dimensional; for every LLM call.
- [ ] **Implement post-call reconciliation (§3.2).** Async-OK on response path.
- [ ] **Implement pre-flight estimation (§4).** Especially for agents.
- [ ] **Define per-feature budget-exceeded policy (§6.2).** Document; test.
- [ ] **Implement two-threshold (§6.3).** Warning at 80%; hard at 100%.
- [ ] **Build monthly reconciliation (§7.2).** Drift < 2% target.
- [ ] **Build chargeback statement (§7.1).** Customer-visible; auditable.
- [ ] **Build customer-facing budget API (§8).** Query, sub-budget, alert config, change request, webhook.
- [ ] **Implement premium-tier features (§5.4).** Dedicated capacity, priority routing.
- [ ] **Implement budget audit log (§2.4).** Every change tracked.
- [ ] **Pre-production test (§9.8 analog):** synthetic runaway tenant; verify per-tenant isolation; verify other tenants unaffected.
- [ ] **Quarterly reconciliation review.** Drift trend; root cause if drift > 2%.

---

## 13. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — per-call attribution that feeds per-tenant consumption.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — the circuit-breaker primitive used in budget-exceeded handling.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) — per-tenant dashboards built on this data.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — model-tier routing per tenant tier.
- [caching-for-cost.md](./caching-for-cost.md) *(companion)* — caching reduces effective cost per call.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) *(companion)* — rate-limit integration with per-tenant cost.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) *(companion)* — batch usage per tenant.
- [cost-incident-runbook.md](./cost-incident-runbook.md) — runbook for budget-exceeded incidents.
- [finops-process.md](./finops-process.md) — chargeback integration with FinOps cadence.

**Elsewhere in this repo.**
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — broader cost observability that consumes per-tenant data.
- [agent-engineering/agent-cost-control.md](../agent-engineering/agent-cost-control.md) — agent-specific cost control that composes with per-tenant.

**Sibling repos.**
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — architectural counterpart; per-tenant fairness design.
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / isolation-models.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/isolation-models.md) — the three isolation models that this engineering implements.
- [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md) — model rates feed this enforcement.
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — queueing topology that includes per-tenant fairness.

**External.**
- Stripe metering API documentation — gold-standard pattern for per-customer usage tracking.
- Redis Lua scripting documentation — atomic operations for budget enforcement.
- AWS Bedrock pricing documentation for cost-rate input.
- Anthropic pricing documentation.
- OpenAI pricing documentation.
- SaaS metered billing literature (FinOps Foundation, AWS Well-Architected SaaS Lens).
