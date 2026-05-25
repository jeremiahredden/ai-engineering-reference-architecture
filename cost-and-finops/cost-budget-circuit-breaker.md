# Cost-Budget Circuit Breaker

> **Audience.** Engineers and tech leads operating an AI feature in production where cost is real (not negligible). Anyone who has been paged because the AI bill last month was 4x projection. **Scope.** The *engineering* practice of treating per-call cost as a circuit-breakable concern at four layers (interaction, session, tenant, feature). Not the architectural decision about cost-and-performance trade-offs (the sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `cost-and-performance-architecture/` owns that). **Worked client.** Meridian Health. **Companion docs.** `cost-attribution.md` for the per-call cost telemetry (coming); `cost-incident-runbook.md` for the response side (coming); `tier-routing-for-cost.md` for the engineering pattern that reduces baseline cost before circuit-breakers are needed (coming).

---

## 1. Why this document exists

AI cost in production is not a budget problem; it is a *reliability* problem. A budget says "this feature should cost about $X per month." A reliability concern says "this feature must not consume more than $Y this hour, regardless of what is happening." Cost incidents — a single misbehaving agent burning $50,000 in a weekend, a runaway batch job spending the quarter's budget in an afternoon, an abusive tenant pushing six-figure spend through a feature that was supposed to be free — are reliability incidents. They have a similar shape to a runaway query that DoSes a database, with the additional property that the bill arrives later.

The reliability discipline that prevents these incidents is the *circuit breaker*: a defined budget at each meaningful scope (per single call, per user session, per tenant, per feature), continuous tracking against the budget, and an automatic terminating action when a budget is breached. The discipline is not "alert on high cost"; alerting is a humans-in-the-loop pattern that fails during the cost-incident window because humans cannot react fast enough. The discipline is "stop spending."

This document is opinionated about three things:

1. **Cost is metered at request time, not after the fact.** The invoice from the model provider is the post-hoc confirmation; the engineering signal must be at call time so that decisions (terminate, degrade, escalate) can be made before the call returns.
2. **All four layers are needed.** Per-interaction alone catches loops but not abuse. Per-tenant alone catches abuse but not loops. The four together form the defense.
3. **Breach handling is hard-stop, not warning.** A circuit breaker that warns but continues is not a circuit breaker; it is a dashboard. The discipline is: when the budget is breached, *the call does not happen*.

Structure: (2) the four-layer model; (3) budget calibration; (4) implementation patterns; (5) breach-handling — hard-stop vs graceful degradation; (6) recovery procedures; (7) cost telemetry foundation; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The four-layer model

Cost runaway happens in four distinguishable shapes. Each needs its own circuit. The names matter — vague "cost budget" loses the discipline.

### 2.1 Layer 1: Per-interaction budget

**Definition.** Maximum cost for a single end-to-end user-visible interaction. A chat turn, a tool call's enclosing agent loop, a single batch-task unit.

**What it catches.** Runaway loops. An agent that gets confused and calls a tool 40 times. A planner that retries 12 times. A retrieval that explodes to top-200 chunks because something went wrong with filtering.

**Typical magnitude.** Set at 2–4x the expected p99 cost. For Meridian Care Coordinator: expected p99 ~$0.35, circuit at $0.50.

**Breach action.** Hard-stop. The next LLM call within the interaction is rejected at the gateway with a structured response that the agent (or workflow) handles as a terminal condition. The user-facing response is "I cannot complete this request right now; please try again or contact your supervisor."

### 2.2 Layer 2: Per-session budget

**Definition.** Maximum cost for an extended user session (a conversation across multiple turns, an analyst's working session, a coordinator's worklist run).

**What it catches.** Users who legitimately interact many times but where the aggregate cost exceeds reasonable bounds — usually a signal of misuse, confusion, or an unusually-expensive conversation that should be escalated.

**Typical magnitude.** Set at 5–10x the per-interaction budget. For Meridian: $1.50 per chat session, calibrated from observed user behavior (the p99 session has 8 interactions).

**Breach action.** Hard-stop on the session. Further interactions in the same session return a graceful failure asking the user to start a fresh session (or to escalate). The session is logged for review.

### 2.3 Layer 3: Per-tenant budget

**Definition.** Maximum cost per tenant per period (typically daily, sometimes hourly for high-velocity workloads).

**What it catches.** Tenant-level abuse, misconfigured tenant automation, an unexpectedly-hot tenant. Also the tenant whose pricing plan does not cover the actual usage — the early-warning before the customer-success conversation.

**Typical magnitude.** Calibrated per tenant based on contracted usage, observed baseline, and a per-tenant headroom multiplier. Meridian uses $50/day for the standard tier, $200/day for the premium tier, with per-tenant overrides for documented heavy users.

**Breach action.** Tenant moves to read-only for the remainder of the period. Existing in-flight interactions complete; new interactions return a tenant-level rate-limit response. Tenant admin is notified; customer-success is notified.

### 2.4 Layer 4: Per-feature budget

**Definition.** Maximum cost per feature across all tenants per period.

**What it catches.** Platform-wide cost runaways that affect multiple tenants simultaneously — a misbehaving prompt deployed in a release, an upstream pricing change that hit all tenants, a model auto-upgrade that increased per-call costs.

**Typical magnitude.** Set at 130–150% of the trailing 7-day average. Meridian's Care Coordinator: ~$1,500/day across all tenants, with the 150% ceiling at $2,250.

**Breach action.** Feature is throttled platform-wide: all new interactions return a degraded response or a "try again later" message. On-call is paged. Existing interactions complete. The feature stays throttled until human investigation determines whether to raise the budget or roll back the change that caused the spike.

### 2.5 Why all four

| Failure mode | Caught by |
|---|---|
| Single confused agent loops 40 times | Per-interaction |
| User has 50-turn debugging conversation | Per-session |
| Tenant's scripted automation hits the API in a loop | Per-tenant |
| Bad prompt deploy raises per-call cost 5x for everyone | Per-feature |
| Compromised API key fans out across many tenants | Per-tenant + per-feature (both will trip) |

No single layer catches everything. The defense is the composition.

---

## 3. Budget calibration

Calibration is the hardest part. Set budgets too tight and the circuit trips on normal operations; set them too loose and they do not catch incidents in time. The discipline:

### 3.1 Start with the cost distribution, not the budget

Instrument cost-per-call before setting any circuits. After 2–4 weeks of production traffic:

- Plot the p50, p90, p99, p99.9 per-interaction cost. Set the circuit at 2–4x the p99 — high enough that normal traffic does not trip it, tight enough that runaway loops do.
- Plot per-session cost distribution. Set the session circuit at the p99 session × 1.5 to 2x — long sessions exist legitimately, but the very long tail is rarely legitimate.
- Per-tenant: examine the per-tenant per-day distribution. The tenant circuit should be calibrated per-tenant tier (standard, premium, custom) rather than as a one-size number.
- Per-feature: take the trailing 7-day daily average × 1.3 to 1.5.

The pattern: derive budgets from observed traffic; never set them in advance based on intuition.

### 3.2 Re-calibrate after every meaningful change

- A prompt change that affects average input tokens: re-calibrate per-interaction.
- A new product feature that drives more turns per session: re-calibrate per-session.
- A new customer tier or pricing change: re-calibrate per-tenant.
- A model swap with different per-token pricing: re-calibrate all four.

Stale calibrations cause false trips (re-calibrate up) or missed incidents (re-calibrate down).

### 3.3 The two-tier calibration: alert below circuit

Each circuit has two thresholds:

- **Alert threshold.** At, say, 80% of the circuit. Surfaces in the cost dashboard; pages on-call at the per-tenant / per-feature levels.
- **Circuit threshold.** The hard-stop level.

The alert threshold is the early-warning that lets humans investigate before automatic termination. The circuit is the backstop when the warning was not acted on (or when the spike is faster than human reaction time).

### 3.4 Per-tenant tiers

Tenants on different contracts have different budgets. The calibration is contractual (the tenant agreement specifies the included usage and the overage handling), with the engineering manifestation:

- **Standard tier.** Lower per-tenant per-day budget. Breach moves to read-only or to a "contact customer-success" response.
- **Premium tier.** Higher budget. Breach behavior may be softer (continue serving but bill the overage explicitly).
- **Custom tier.** Per-customer documented budgets. The engineering surface accommodates per-tenant overrides without hardcoding tenant names.

The discipline is that tier mapping is configuration, not code. New tenants are onboarded by setting tier; the budget structure is uniform.

---

## 4. Implementation patterns

The circuit breaker is implemented at the AI gateway layer (or its equivalent platform component). Every LLM call, every reranker call, every embedding call passes through the gateway; the gateway is the natural enforcement point.

### 4.1 The gateway interface

```python
class AIGateway:
    def call_llm(
        self,
        provider: str,
        model: str,
        messages: list,
        ...,
        context: CallContext,   # tenant, user, session, interaction, feature, trace
    ) -> LLMResponse:
        # Pre-call cost estimate (input tokens + max-output tokens)
        estimated_cost = self._estimate_cost(provider, model, messages, ...)

        # Check all four budgets
        budget_check = self._check_all_budgets(estimated_cost, context)
        if not budget_check.allowed:
            self._record_circuit_trip(budget_check.layer, context)
            raise CircuitBreakerTripped(
                layer=budget_check.layer,
                user_message=self._user_message_for(budget_check),
                agent_message=self._agent_message_for(budget_check),
            )

        # Make the call
        response = self._do_call(provider, model, messages, ...)

        # Record actual cost (input + actual output tokens)
        actual_cost = self._compute_actual_cost(response)
        self._record_cost(actual_cost, context)

        return response
```

Three patterns to notice:

- **Pre-call estimate.** The check runs against an *estimate* of the cost (input tokens are known; output tokens are bounded by max-output). A call that would push past the circuit is rejected before being made.
- **Post-call record.** The actual cost is recorded after the response, so the running totals are accurate.
- **Structured exception.** `CircuitBreakerTripped` is structured so callers (agents, workflows) can handle it as a known condition rather than as an unexpected error.

### 4.2 The four budget checks

```python
def _check_all_budgets(self, estimated_cost: float, ctx: CallContext) -> BudgetCheck:
    # Layer 1: per-interaction
    interaction_total = self._get_interaction_cost(ctx.interaction_id) + estimated_cost
    if interaction_total > self._limit_for("interaction", ctx):
        return BudgetCheck(allowed=False, layer="interaction", ...)

    # Layer 2: per-session
    session_total = self._get_session_cost(ctx.session_id) + estimated_cost
    if session_total > self._limit_for("session", ctx):
        return BudgetCheck(allowed=False, layer="session", ...)

    # Layer 3: per-tenant (per period)
    tenant_total = self._get_tenant_cost_today(ctx.tenant_id) + estimated_cost
    if tenant_total > self._limit_for("tenant", ctx):
        return BudgetCheck(allowed=False, layer="tenant", ...)

    # Layer 4: per-feature (per period)
    feature_total = self._get_feature_cost_today(ctx.feature_id) + estimated_cost
    if feature_total > self._limit_for("feature", ctx):
        return BudgetCheck(allowed=False, layer="feature", ...)

    return BudgetCheck(allowed=True)
```

Order matters. Per-interaction trips first because it is fastest to evaluate and most often relevant. Per-feature is last because it is the rarest and most expensive check (it requires a platform-wide aggregation).

### 4.3 Running totals

Each budget needs a current-total counter that updates in near-real-time. The implementation choices:

- **Per-interaction and per-session.** Easy — kept in process memory or in a short-lived cache keyed by interaction/session ID. Cost is bounded by the interaction/session's lifetime.
- **Per-tenant and per-feature.** Harder — needs to be shared across all gateway instances, persistent across process restarts, fast to read on every call. Redis (with a per-day key that auto-expires) is the typical choice. Each call writes a small increment; each call reads the current value before its budget check.

The latency budget for the budget check itself must be sub-5ms. A 50ms budget check would add unacceptable per-call latency.

### 4.4 Distributed-system correctness

In a distributed deployment (multiple gateway instances), the budget counters must be consistent across instances. Two patterns:

- **Centralized counter (Redis INCR).** Each call atomically increments the shared counter and reads the new value. Strong consistency; bottleneck if call volume is very high.
- **Local-buffered with periodic flush.** Each gateway instance maintains a local counter; periodically flushes to the shared store. Eventually consistent; can over-shoot the budget by the flush interval.

The Meridian Care Coordinator uses centralized Redis INCR for per-tenant and per-feature; per-interaction and per-session are in-process (single-instance scope for those layers).

### 4.5 The kill-switch override

A per-tenant or per-feature kill switch can immediately halt a tenant or feature, even before the budget is breached. The pattern: a feature flag readable by the gateway at every call; toggled by on-call in response to an incident. The kill switch is the manual circuit, complementing the automatic ones.

---

## 5. Breach handling

The behavior on breach is the most consequential design choice. Get this wrong and either the circuit does nothing (warning without termination) or it tears down user-visible functionality unnecessarily.

### 5.1 Per-interaction breach: hard-stop with graceful response

The current call is rejected. The agent / workflow receives the structured `CircuitBreakerTripped` exception. The agent's prompt is built to handle this:

> If you receive a `circuit_breaker_tripped` response, do not retry. Inform the user that you cannot complete this request right now and offer them the option to start a fresh interaction or escalate to a clinician.

The user-facing message is polite, specific to what happened, and offers a next action. The interaction terminates cleanly.

### 5.2 Per-session breach: end the session, allow new sessions

The session is marked as exceeded. Further calls within this session ID are rejected with a "this session has used its budget; please start a fresh conversation" message. New session IDs start fresh.

This pattern accommodates the legitimate user who hit the cap by accident — they can restart and continue. It also stops the runaway-conversation pattern where a single session generates unbounded cost.

### 5.3 Per-tenant breach: read-only mode with notification

The tenant moves to a read-only mode: all retrieval-only operations still work; all write/drafting/side-effect operations are rejected with a "your tenant has reached its daily budget; contact customer success" response. The tenant administrator is emailed; customer-success is paged.

The pattern preserves some functionality (read-only is often useful) while halting the cost spend.

### 5.4 Per-feature breach: throttle platform-wide

The feature is throttled platform-wide. New interactions return either a degraded response (smaller model, no retrieval, simpler answer) or a "service is currently unavailable, please try again later" message. The choice between degraded-mode and unavailable-mode depends on the feature's design:

- **Care Coordinator chat:** degraded mode (smaller model, no retrieval, the agent responds with "I can't access the full clinical knowledge base right now; here's what I can offer..."). The user is informed; the clinical-safety bar is maintained because the response explicitly disclaims limitations.
- **Async coordination tasks:** unavailable-mode. The task is rejected at submission with "the coordination feature is temporarily unavailable; please try again in an hour."

On-call is paged. The throttle lifts only after human investigation determines: was the budget calibration wrong (raise it), or was there a cost-spike incident (find the cause and roll back, then raise it back to normal).

### 5.5 Degraded mode vs unavailable mode

Degraded mode is the better user experience but harder to engineer. The system must have a pre-configured fallback path that does not consume the over-budget resource. For Care Coordinator: the fallback is Haiku-class generation without retrieval, returning an explicitly-disclaimed response.

Unavailable mode is the simpler engineering: reject the request entirely. Use when the feature cannot be degraded meaningfully without compromising its purpose (a clinical-decision-support feature that returns "here's my guess" is worse than one that returns "I'm unavailable").

The choice is per-feature; document it in the feature's runbook.

### 5.6 What breach handling MUST do

- **Be deterministic.** The same condition always produces the same response. No randomness, no flakiness.
- **Be observable.** Every circuit trip is logged at high severity; the trace shows it; dashboards count it.
- **Be reversible.** Recovery is a documented procedure (section 6).
- **Never produce silent quality degradation.** If the system falls back to degraded mode, the user must be informed; downstream consumers (other features, audit logs) must see that the response was degraded.

### 5.7 What breach handling MUST NOT do

- **Retry the call after waiting.** "Maybe by the time we retry, the budget will be back" is the failure mode that ensures budgets never enforce anything.
- **Silently switch to a cheaper model without notifying the user.** Cost-driven model swap without telemetry is the silent-quality-regression failure.
- **Bypass the breach for "high priority" calls.** Unless there is a separate priority circuit (which is rare and complex), the budget applies uniformly. The discipline degrades the moment exceptions accumulate.
- **Disable the circuit during an incident.** The temptation is to disable the circuit so users can continue using the feature; this defeats the protection. The right response is to raise the budget after investigation, not to disable enforcement.

---

## 6. Recovery procedures

Circuit-breaker recovery is operational, not automatic. The runbook:

### 6.1 Per-interaction / per-session recovery

Automatic. The next interaction in a new interaction-ID or session-ID starts fresh. No human action required.

The metric to watch: the rate of per-interaction circuit trips. A baseline rate is normal (the agent occasionally encounters edge cases). A spike is a signal.

### 6.2 Per-tenant recovery

Manual. The runbook:

1. Acknowledge the page.
2. Inspect the tenant's cost breakdown for the day — which interactions consumed cost, which user accounts, which features.
3. Triage: is this legitimate usage (the tenant launched a campaign, ran a backfill, deployed a new automation), or is it abuse / misconfiguration?
4. If legitimate: raise the tenant's daily budget (per pre-approved limit for the tier); document the override; reset the day's counter to allow continued operation.
5. If misconfiguration: contact the tenant admin to resolve the misconfiguration; the tenant remains in read-only until they confirm.
6. If abuse: escalate to customer-success and security; the tenant remains in read-only.

### 6.3 Per-feature recovery

Manual, high-priority. The runbook:

1. Acknowledge the page.
2. Identify the cause: prompt-version change today? Model-version change today? Upstream cost-per-token change? Traffic spike?
3. If a deploy-related cause: roll back the change; the cost should return to baseline. Lift the throttle.
4. If a traffic spike with no engineering cause: capacity-plan; raise the daily budget (with leadership awareness); lift the throttle.
5. If an upstream pricing change: escalate to leadership for budget reset; communicate the new baseline.
6. Post-incident: document the cause, the time-to-detect, the time-to-recover; update calibration if needed.

### 6.4 What recovery should not do

- **Permanently disable a circuit.** Even after recovery, the circuit stays in place; only the threshold is adjusted.
- **Skip the post-incident review.** Cost incidents are reliability incidents; they need the same review discipline.
- **Hide the trip from users.** If users experienced degraded service or rejected requests, the incident communication says so.

---

## 7. Cost telemetry foundation

The circuit breaker depends on cost telemetry that does not exist by default. The instrumentation must be in place before the circuit can be:

### 7.1 Per-call cost computation

Every LLM call (or reranker / embedding call) records cost. The computation:

- **Input tokens.** Use the provider's tokenizer for the model.
- **Output tokens.** Use the provider's reported output tokens (returned in the response).
- **Cached tokens.** Some providers report cached vs uncached separately; price them separately.
- **Provider pricing.** A lookup table by provider × model × token-type. Updated on every provider pricing change.

The pattern: compute cost at call time, not by reconciling with the provider's invoice later. The invoice is the post-hoc verification.

### 7.2 Attribution dimensions

Every cost entry carries attribution:

- **Feature ID.** Which AI feature is this call part of?
- **Tenant ID.** Which tenant?
- **User ID.** Which user (when relevant)?
- **Session ID.** Which session?
- **Interaction ID.** Which user-facing interaction?
- **Trace ID.** The full trace this call belongs to.
- **Model + version.** Which model produced this cost?
- **Prompt version.** Which prompt version was used?
- **Worker / agent role.** Supervisor, knowledge-worker, drafting, classifier?

Without these dimensions, attribution is impossible; with them, the cost dashboard answers "which feature is driving cost?" and "which tenant?" and "which prompt change correlates with the cost change?"

### 7.3 Per-call vs aggregate storage

- **Per-call.** Written to the trace (for the trace-as-debugging-surface use case). Sampled to the cost-attribution store for aggregations.
- **Aggregates.** Pre-computed rollups (per-tenant-per-day, per-feature-per-day, per-tenant-per-hour) in the cost data store. Used by dashboards and by circuit-breaker checks.

The aggregates must be fresh (sub-minute) for circuit-breaker checks to be effective. Stale aggregates miss the spike that the circuit was supposed to catch.

### 7.4 The cost dashboard

The cost dashboard supports three queries:

- **Per-feature trend.** Daily cost per feature, with rolling 7-day average.
- **Per-tenant ranking.** Top tenants by cost today, this week, this month.
- **Per-feature × per-tenant.** Drill-down: this feature, this tenant, this user, this interaction.

The dashboard is the human-readable view of the same data the circuit breakers consume. Alerts route from the dashboard.

---

## 8. Worked Meridian Health example

### 8.1 The four-layer configuration

| Layer | Limit | Alert at |
|---|---|---|
| Per-interaction (chat) | $0.50 | $0.40 |
| Per-interaction (async coordination task, per-patient subtask) | $0.75 | $0.60 |
| Per-session (chat) | $1.50 | $1.20 |
| Per-tenant per day (standard tier) | $50 | $40 |
| Per-tenant per day (premium tier) | $200 | $160 |
| Per-tenant per day (dedicated-infrastructure customer) | $500 | $400 |
| Per-feature per day (Care Coordinator chat across all tenants) | $1,500 | $1,200 |
| Per-feature per day (async coordination across all tenants) | $400 | $320 |

Calibration was done across 4 weeks of pilot traffic (3 hospitals). The p99 chat interaction was $0.32; the circuit at $0.50 gives ~50% headroom over normal traffic.

### 8.2 The implementation

The AI gateway in front of every LLM call enforces all four budgets. Per-tenant and per-feature counters are kept in Redis (per-day keys with TTL of 25 hours so the rollover is clean). Per-interaction and per-session counters are in-process (each gateway instance handles a sticky-routed session).

The gateway exposes the cost telemetry to the observability stack (Datadog) and to the cost dashboard. The dashboard's drill-down lets ai-platform-eng investigate any tenant or feature's cost distribution.

### 8.3 Real circuit-breaker trips observed

- **2026-03-22, per-interaction trips.** A prompt change introduced a verbose chain-of-thought instruction that pushed average input tokens from 4.1K to 6.8K. Per-interaction cost rose from $0.18 average to $0.31 average; some interactions exceeded $0.50. The circuit caught 14 individual interactions; the prompt change was rolled back the same day; cost normalized.

- **2026-04-08, per-tenant trip.** A pilot hospital's clinical-informatics team enabled an automated workflow that called the Care Coordinator's async API in a loop, generating ~600 unintended interactions per hour. The tenant breached $50/day in 90 minutes; circuit moved them to read-only; tenant admin was alerted; the automation was identified and disabled within an hour.

- **2026-04-29, per-feature trip.** A model auto-upgrade by Anthropic (which the team had not configured an aggressive pin for at that point) shifted the supervisor calls to a new model version with slightly higher per-token cost. Aggregate per-feature cost rose ~15% across all tenants. The trip happened at 4pm UTC; on-call investigated; the model pin was tightened to the previous version; the trip threshold was raised to accommodate normal variance going forward. Finding `ARCH-CARE-004` was filed as a result.

Each trip surfaced something real — a regression, an abuse, an unexpected dependency change — that would have manifested as a budget surprise at month-end without the circuit.

### 8.4 The on-call runbook

Cost-incident runbook (per-tenant or per-feature trip):

1. Acknowledge page within 5 minutes.
2. Open the cost dashboard; identify the trip (tenant or feature).
3. Open the cost-attribution drill-down for the affected scope. Identify the breakdown: which interaction class, which user, which prompt version.
4. Triage class:
   - Engineering cause (prompt / model / code change today): roll back the change.
   - Tenant cause (tenant's automation): contact tenant admin.
   - Genuine traffic (no engineering, no tenant cause): capacity-plan; raise budget with leadership awareness.
5. Document the trip in the cost-incidents log. Update calibration if needed.
6. Hold the throttle / read-only until the cause is addressed.

The runbook target: trip-to-throttle-lift < 1 hour for engineering causes, < 4 hours for tenant causes (depends on tenant response).

---

## 9. Anti-patterns

### 9.1 "Alert on high cost"

The team has dashboards and alerts but no termination action. Alerts fire; the on-call investigates; by the time the investigation completes, hours have passed and the budget is already blown.

**Corrective.** Alerts are the warning; the circuit is the protection. Both are needed; alerts alone are not protection.

### 9.2 "One global budget for the whole feature"

The team set a per-feature budget but no per-tenant or per-interaction. One tenant's misuse consumes the platform-wide budget; everyone is throttled because one tenant misbehaved.

**Corrective.** All four layers. Per-tenant catches per-tenant problems before they affect everyone.

### 9.3 "Circuit can be disabled by feature flag"

A feature flag exists that disables the circuit entirely, intended for "emergencies." In practice, it gets used during cost incidents because users are complaining about being blocked; the circuit is bypassed; the bill arrives.

**Corrective.** Per-tenant budget overrides are configurable; the circuit itself is not. The override is bounded (raise a specific tenant's budget to X) rather than open-ended (disable enforcement).

### 9.4 "Retry on circuit trip"

The agent or the calling code retries when the circuit trips. The retry hits the same trip; the retry-backoff exponential delays compound; the user-facing latency explodes; cost is not actually saved because the underlying issue persists.

**Corrective.** Circuit trips are terminal for that call/session. The structured exception is the signal; the agent's prompt knows to handle it as terminal.

### 9.5 "Calibration was set six months ago and never updated"

Budgets were calibrated against historical traffic that no longer matches current behavior. Either the circuit trips on normal traffic (over-restrictive) or it never trips when it should (over-permissive).

**Corrective.** Quarterly calibration review. Re-derive from current cost distribution. Update thresholds.

### 9.6 "Per-tenant budget is the same for all tenants"

The standard-tier tenant and the dedicated-infrastructure tenant have the same daily budget. The premium customer is throttled at normal-tenant scale; the standard tenant has more headroom than their contract entitles.

**Corrective.** Per-tier budgets; per-tenant overrides for documented heavy users; configuration-driven, not code-driven.

### 9.7 "Budget check adds 100ms to every call"

The budget check went into production without latency-optimization. The check added meaningful latency to every LLM call; the team disabled the per-feature check to reclaim performance.

**Corrective.** The budget check must be sub-5ms (Redis-backed counters, in-process caches). Performance is a precondition for the circuit being usable in production.

### 9.8 "Cost telemetry is in a separate system from the gateway"

Cost telemetry lives in a billing system that ingests provider invoices monthly. The gateway has no idea what costs have been incurred today. Circuit decisions are impossible because the data needed to make them is in another system on a monthly cadence.

**Corrective.** Per-call cost is computed at the gateway, written to a fast aggregation store, available to the circuit check at sub-5ms.

---

## 10. Findings (sprint-assignable)

### COST-001 — Severity: Critical
**Finding.** No per-interaction cost circuit exists; runaway agent loops have caused observed cost incidents.
**Recommendation.** Implement per-interaction circuit at the gateway; hard-stop on breach; structured response to the calling agent.
**Owner.** ai-platform-eng, sprint N+1.

### COST-002 — Severity: Critical
**Finding.** No per-tenant cost circuit exists; tenant misconfigurations can consume unbounded spend.
**Recommendation.** Per-tenant per-day budget; read-only mode on breach; tenant admin and customer-success notification.
**Owner.** ai-platform-eng + customer-success, sprint N+1.

### COST-003 — Severity: Critical
**Finding.** Circuit trips are warning-only; no automatic termination.
**Recommendation.** Convert to hard-stop with graceful response per section 5; verify trips actually halt spending.
**Owner.** ai-platform-eng, sprint N+1.

### COST-004 — Severity: High
**Finding.** Per-feature circuit does not exist; a deploy-related regression could affect all tenants simultaneously.
**Recommendation.** Per-feature per-day budget; throttle platform-wide on breach; on-call paging.
**Owner.** ai-platform-eng + sre, sprint N+2.

### COST-005 — Severity: High
**Finding.** Per-call cost is computed by reconciling provider invoices, not at call time; circuit decisions cannot be made on real-time data.
**Recommendation.** Implement per-call cost computation at the gateway; aggregate in Redis for circuit consumption.
**Owner.** ai-platform-eng, sprint N+1.

### COST-006 — Severity: High
**Finding.** Cost telemetry lacks attribution dimensions (feature, tenant, user, prompt-version, model-version); diagnosis of cost incidents requires manual trace correlation.
**Recommendation.** Add attribution per section 7.2; surface in cost dashboard.
**Owner.** ai-platform-eng, sprint N+2.

### COST-007 — Severity: High
**Finding.** Budgets are set once and never re-calibrated; circuits trip on normal traffic OR fail to catch genuine incidents.
**Recommendation.** Quarterly calibration review per section 3.2; re-derive thresholds from current cost distribution.
**Owner.** ai-platform-eng + finops, sprint N+3.

### COST-008 — Severity: High
**Finding.** Per-session budget is not enforced; long debugging conversations have consumed disproportionate spend.
**Recommendation.** Per-session circuit; session moves to budget-exceeded mode on breach; user is prompted to start a fresh session.
**Owner.** ai-platform-eng, sprint N+2.

### COST-009 — Severity: High
**Finding.** Circuit-breaker exception is unstructured; agents do not handle it deterministically.
**Recommendation.** Structured `CircuitBreakerTripped` exception per section 4.1; update agent prompts to handle each circuit class.
**Owner.** ai-platform-eng, sprint N+2.

### COST-010 — Severity: High
**Finding.** Degraded-mode fallback is not configured for the feature; circuit-breached state means total unavailability.
**Recommendation.** Define per-feature degraded mode (smaller model, no retrieval, explicit user disclaimer); engineer the fallback path; eval-test the degraded behavior.
**Owner.** ai-platform-eng, sprint N+3.

### COST-011 — Severity: Medium
**Finding.** No kill-switch / manual circuit exists; on-call cannot immediately halt a tenant or feature outside automated circuits.
**Recommendation.** Add per-tenant and per-feature kill-switch flags readable by the gateway at every call; document runbook.
**Owner.** ai-platform-eng + sre, sprint N+3.

### COST-012 — Severity: Medium
**Finding.** Per-tenant tier mapping is hard-coded; new tenant tiers require code changes.
**Recommendation.** Move tier mapping to configuration; per-tenant overrides supported declaratively.
**Owner.** ai-platform-eng, sprint N+3.

### COST-013 — Severity: Medium
**Finding.** Budget-check latency was not measured; performance impact on per-call latency is unknown.
**Recommendation.** Instrument budget-check latency; target sub-5ms; tune the data store if needed.
**Owner.** ai-platform-eng, sprint N+3.

### COST-014 — Severity: Medium
**Finding.** Cost-incident runbook does not exist; on-call response to a cost trip is ad-hoc.
**Recommendation.** Write the runbook per section 6 / section 8.4; rehearse with on-call.
**Owner.** ai-platform-eng + sre, sprint N+2.
**Cross-link.** `cost-incident-runbook.md` (coming).

### COST-015 — Severity: Medium
**Finding.** Distributed gateway instances use eventually-consistent counters; circuits can over-shoot by the flush interval.
**Recommendation.** Move per-tenant and per-feature counters to centralized Redis with atomic increment; per-interaction and per-session counters stay in-process.
**Owner.** ai-platform-eng, sprint N+3.

### COST-016 — Severity: Medium
**Finding.** Pre-call cost estimation is conservative (assumes max-output tokens); the circuit trips earlier than necessary on short-response interactions.
**Recommendation.** Tune the estimate based on observed output-token distribution per prompt class; re-calibrate quarterly.
**Owner.** ai-platform-eng, sprint N+4.

### COST-017 — Severity: Medium
**Finding.** Circuit trips are not surfaced in the customer-facing tenant dashboard; tenants discover the read-only mode by trying to use the feature.
**Recommendation.** Surface circuit state in the tenant admin UI; proactive email on tenant-circuit trip.
**Owner.** ai-platform-eng + product, sprint N+4.

### COST-018 — Severity: Medium
**Finding.** Two-tier alerting (alert at 80%, circuit at 100%) is not implemented; the team is paged only when the circuit trips.
**Recommendation.** Implement alert thresholds per section 3.3; route alerts to ai-platform-eng before the circuit fires.
**Owner.** ai-platform-eng + sre, sprint N+3.

### COST-019 — Severity: Low
**Finding.** Trip log is captured but trip-rate trends are not reported to leadership.
**Recommendation.** Monthly cost-incident report; track trip rate per layer; flag rising rates as a system-health signal.
**Owner.** ai-platform-eng team lead, sprint N+5.

### COST-020 — Severity: Low
**Finding.** Provider pricing changes are not automatically reflected in cost computation; pricing-table update is manual.
**Recommendation.** Subscribe to provider pricing announcements; build a CI check that flags pricing-table staleness; verify costs against monthly invoice reconciliation.
**Owner.** ai-platform-eng + finops, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team operating an AI feature in production without circuit breakers:

- [ ] **Sprint 0 — telemetry.** Per-call cost computation at the gateway. Attribution dimensions captured. Aggregations in a fast store. Cost dashboard exists. (This is the foundation; circuits cannot land without it.)
- [ ] **Sprint 1 — calibration.** Two weeks of production traffic with telemetry. Distribution analysis. Initial budget thresholds derived.
- [ ] **Sprint 2 — per-interaction + per-session.** Land the two in-process circuits first. Structured exception. Agent prompts updated to handle.
- [ ] **Sprint 2 — runbook.** Cost-incident runbook documented; on-call walked through.
- [ ] **Sprint 3 — per-tenant.** Land the per-tenant circuit. Tenant-tier configuration. Tenant admin notification. Customer-success alerting.
- [ ] **Sprint 3 — per-feature.** Land the per-feature circuit. On-call paging. Degraded-mode fallback engineered.
- [ ] **Sprint 4 — two-tier alerts.** Alert thresholds at 80% in addition to circuit at 100%; route alerts.
- [ ] **Sprint 4 — kill switch.** Manual per-tenant and per-feature kill-switches.
- [ ] **Sprint 5 — UX integration.** Tenant-facing dashboards show circuit state; users see informative messages on breach.
- [ ] **Ongoing — quarterly review.** Calibration re-derived from current traffic. Trip rates reported. Anti-patterns audited (section 9).

A team that completes this sequence has the cost-reliability discipline that turns cost incidents from quarterly surprises into bounded operational events. A team that skips the telemetry foundation finds that they cannot land the circuits even if they want to — without per-call cost data, there is nothing to circuit on.

---

## 12. References

- The SRE canon on circuit breakers (Google SRE book chapter on cascading failures; Hystrix / Resilience4j documentation) — the pattern this document adapts to AI cost.
- AWS, Azure, GCP cost-management documentation — the cloud-side patterns for budget alerts, which this complements (cloud budgets are post-hoc; AI circuit breakers are at-call).
- FinOps Foundation guidance on per-workload cost attribution — the general FinOps discipline this is the AI-specific overlay on.
- This repo: [agent-engineering/agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md) — sections 3.2 (turn / cost / time / tool-call budgets) and 8 (cost control), which this document is the deeper engineering reference for.
- This repo: [cost-and-finops/cost-attribution.md](./) (coming) for the per-call cost telemetry foundation.
- This repo: [cost-and-finops/cost-incident-runbook.md](./) (coming) for the on-call response side.
- This repo: [cost-and-finops/tier-routing-for-cost.md](./) (coming) for the engineering pattern that lowers baseline cost (and thus the circuits trip less often).
- Sibling repo: [ai-architecture-reference-architecture/cost-and-performance-architecture/cost-incident-playbook.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/cost-and-performance-architecture/) (coming) for the architecture-level cost decisions that this engineering practice operationalizes.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked architecture this document references; finding `ARCH-CARE-003` is the cross-link.
