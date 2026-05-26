# Agent Cost Control

> **Audience.** Engineers and tech leads responsible for the cost envelope of agentic features. On-call engineers paged during a cost incident. Anyone whose feature's cost line will appear on the next finance review. **Scope.** The *engineering* discipline of bounding, attributing, routing, and recovering from cost in agent loops — per-request budgets, per-tenant caps, gateway-side circuit breakers, tier routing inside the loop, and the in-progress incident response. Not the gateway primitive itself (see [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)). Not the architectural cost trade-off framework (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `cost-and-performance-architecture/`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Agents have the worst cost envelope of any AI feature shape in 2026. A single LLM call has a bounded cost (input tokens + output tokens × per-token rates, capped by the response budget). A workflow has a bounded cost (sum of per-step LLM costs over the longest path). An agent has cost bounded only by the budgets the team has explicitly engineered.

The headlines from agent cost incidents over the past two years:

- A coding agent at a mid-sized company looped on a single user's repository, calling expensive frontier-model completions 3,400 times in a weekend. Bill: $58,000.
- A customer-support agent at an early-stage startup got into a hand-off loop with a peer agent; the loop ran for 19 hours before anyone noticed. Bill: $12,000 (small only because the startup had a low per-call budget; otherwise much worse).
- A research agent at a research lab, deployed to internal users, was misused (a user gave it the task of "summarise this 800-page PDF; if anything is unclear, search the web") — the agent searched the web 600 times, downloaded entire result pages into context, and exhausted the lab's monthly budget in a single afternoon.

Each incident has a common pattern: the team did not engineer cost as a primary design concern. Budgets were aspirational (a hope), not enforced (a control). Observability was after-the-fact (the invoice). Tier routing was uniform (frontier model for everything, even when a small model would have sufficed).

The discipline this document covers is preventative. The patterns are tightly aligned with the agent's loop runner ([agent-loop-design.md](./agent-loop-design.md)), the gateway-level circuit-breaker primitive ([cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)), and the cost attribution surface ([cost-attribution.md](../cost-and-finops/cost-attribution.md)). The agent-specific concerns — the loop, the tools, the per-turn behaviour — are this document's focus.

This document is opinionated about four things:

1. **Budgets are enforced, not advisory.** A budget that warns is a dashboard; a budget that stops is a control. Agents specifically warrant the latter because the loop will burn through a warning threshold without noticing.
2. **Tier routing inside the loop is a primary cost lever.** Most "agent cost too high" cases are reducible by 60–80% via tier routing — cheap model for the orchestrator turns, expensive model only for the small minority of turns that need it.
3. **Incident response must be sub-15-minute.** Cost incidents compound by the minute. A 15-minute mean-time-to-mitigation on a runaway agent is the difference between a $500 incident and a $50,000 one. The runbook, observability, and authority to act must be ready.
4. **Attribution is non-optional.** Without per-tenant, per-feature, per-user attribution, cost incidents cannot be triaged and chargeback cannot be implemented. Attribution is part of the cost-control discipline, not an analytics nice-to-have.

Structure: (2) the cost decomposition for agents; (3) per-request budgets; (4) per-tenant cost caps; (5) cost-as-circuit-breaker — the gateway integration; (6) tier routing inside the loop; (7) in-progress cost incident response; (8) cost observability and dashboards; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The cost decomposition for agents

The cost of an agent invocation is not one number; it is a sum across many components.

### 2.1 The components

```
agent_request_cost = sum over turns of (
    per_turn_LLM_input_cost
  + per_turn_LLM_output_cost
  + per_turn_tool_costs
)

per_turn_LLM_input_cost = prompt_tokens × per_input_token_rate(model)
per_turn_LLM_output_cost = completion_tokens × per_output_token_rate(model)
per_turn_tool_costs = sum over tool calls of tool_cost
  where tool_cost depends on tool: LLM-as-tool, external metered API, retrieval, free local function
```

Per-turn LLM input cost dominates in practice because most agents call the model with growing context history. The 1st turn's prompt is small; the 10th turn's prompt is much larger (history + tool results + structured state). The input cost grows quadratically-ish with turn count.

### 2.2 The cost-attribution dimensions

Each request's cost must be attributable to:

- **Tenant** (which customer paid for / consumed this).
- **Feature** (care-coordinator, summary-generator, analytics-copilot).
- **User** (within a tenant, which user's actions).
- **Trace** (which top-level invocation; for joining with traces during debugging).
- **Model** (which model variants contributed; for tier-routing analysis).
- **Tool** (which tools contributed cost; for tool-level analysis).

The attribution lives in span attributes (per [cost-attribution.md](../cost-and-finops/cost-attribution.md)) and rolls up into dashboards (per [cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md)).

### 2.3 The cost-envelope-by-shape comparison

A reminder of why agents specifically warrant this discipline:

| Shape | Cost envelope shape | Worst-case behaviour |
| --- | --- | --- |
| Single LLM call | Bounded by completion_max_tokens | One bad call: cents |
| Workflow with N LLM steps | Bounded by N × per-step cap | Bad workflow run: dollars |
| Hybrid (workflow + agent step) | Workflow bound + agent-step bound | Bad agent step inside hybrid: $1–$10 |
| Agent (no engineered bounds) | Bounded only by budget enforcement | Runaway: $1000s–$100,000s |

The variance in worst-case is the operational reality. Per-feature engineering must match the shape's risk profile.

### 2.4 Cost vs latency vs quality

Cost trades against latency and quality:

- Cheaper model (tier routing): lower cost, faster, lower quality on hard cases.
- Smaller context: lower cost, possibly lower quality if relevant context is dropped.
- Fewer turns: lower cost, faster, may miss the right answer.
- Cheaper tools (e.g., local search vs paid API): lower cost, possibly less accurate.

Cost optimisation is not "make it cheap"; it's "make it the right tradeoff for the use case." Document the tradeoff in the feature's design.

---

## 3. Per-request budgets

The first-level enforcement. Every agent invocation has a maximum cost it can spend before terminating.

### 3.1 The budget shape

A per-request budget for an agent has multiple sub-budgets (per [agent-loop-design.md](./agent-loop-design.md) section 5):

| Sub-budget | Typical scale | Enforced by |
| --- | --- | --- |
| Turns | 5–50 | Runner before each new turn |
| Cost (USD) | $0.50–$10 (varies by feature) | Runner before each LLM call; gateway as backstop |
| Wall-clock time | 30s–300s | Runner before each new turn |
| Tool calls | 10–100 | Runner before each tool dispatch |

The budgets are AND-ed: breach of *any* terminates. The most operationally important is the cost budget; the others are correlative (a turn budget bound the cost budget approximately, but the cost budget is the source of truth).

### 3.2 Calibrating the cost budget

The budget is set per feature. The calibration process:

1. **Baseline from observed traffic.** Run the feature on representative golden traffic; observe the p50, p95, p99 cost per request.
2. **Set the budget at 2–4× p99.** Allows headroom for outliers without permitting runaway.
3. **Review monthly.** If the budget is hit > 0.1% of the time, either (a) the budget is too tight, (b) the feature has a new failure mode, (c) the model or prompt regressed.

Worked example for Meridian's care-coordinator:

- p50: $0.08
- p95: $0.16
- p99: $0.22
- Budget: $0.50 (~2.3× p99)
- Budget breach rate: 0.04%

The 0.04% breach rate is acceptable; breaches are flagged for review (was this a real runaway, or a legitimate edge case?).

### 3.3 The runner's enforcement

Before each LLM call, the runner:

```python
def _check_budget(self, state: AgentState) -> BudgetCheck:
    if state.turn >= self.budgets.max_turns:
        return BudgetBreach(kind="turns", value=state.turn)
    if state.cost_usd >= self.budgets.max_cost_usd:
        return BudgetBreach(kind="cost", value=state.cost_usd)
    if state.elapsed_time >= self.budgets.max_time_s:
        return BudgetBreach(kind="time", value=state.elapsed_time)
    if state.tool_calls >= self.budgets.max_tool_calls:
        return BudgetBreach(kind="tool_calls", value=state.tool_calls)
    return BudgetOK()
```

Breaches trigger the final-turn pattern (per [agent-loop-design.md](./agent-loop-design.md) section 5.3): one last LLM call with a tight cost cap, instructed to produce a graceful failure response, then terminate.

### 3.4 The "near-breach" signal

Before the breach itself, the runner can signal "near-breach" — e.g., at 80% of any budget. The agent's prompt can incorporate this signal:

```
You are at 80% of your cost budget for this request. Move toward producing a final answer or escalating; do not start new investigations.
```

The model often responds to the signal, producing a faster termination than the hard breach would. The signal is a soft control; the breach is the hard control.

### 3.5 Per-request cost as a tool-result attribute

When a tool call has cost (LLM-as-tool, paid API), the tool returns the cost as part of the result; the runner accumulates it. Without per-tool cost reporting, the runner can underestimate the request's cost (only counting the agent's own LLM calls).

### 3.6 The per-request budget vs the gateway-side circuit breaker

Two layers, not one:

- **Runner enforcement.** First line of defense. The runner stops the agent gracefully on breach. The agent produces a final answer. The user gets a coherent response.
- **Gateway circuit breaker.** Second line of defense. The gateway refuses LLM calls beyond a stricter threshold. Catches cases where the runner's enforcement fails (bug, misconfiguration, runner bypass). Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).

Both fire on most runaway cases; the gateway is the backstop.

---

## 4. Per-tenant cost caps

The second-level enforcement. Cost summed across a tenant's requests.

### 4.1 Why per-tenant

A per-request budget catches single-request runaway. It does not catch:

- A tenant with many users, each running normal-cost requests, but in aggregate spending far above the tenant's quota.
- A single abusive user within a tenant making repeated requests.
- A misbehaving downstream system that invokes the agent in a loop.

Per-tenant caps catch all three.

### 4.2 The cap shape

| Window | Typical magnitude | Reset |
| --- | --- | --- |
| Per-minute | $5–$50 (varies by tenant tier) | Sliding |
| Per-hour | $50–$500 | Sliding |
| Per-day | $250–$5000 | Sliding |
| Per-month | $5000–$100k (contractual) | Calendar month |

The windows together prevent both burst spending (per-minute) and sustained over-spending (per-day, per-month).

### 4.3 Calibration

The per-tenant caps reflect the contractual cost commitment (or the implied service tier for a free product). They are typically set with 1.5–2x headroom over expected usage; breach is unusual and warrants investigation.

For Meridian:

- Standard tier: $50/min, $300/hr, $1500/day, $30k/month.
- Enterprise tier: $200/min, $1200/hr, $6000/day, $120k/month.
- Configurable per-tenant overrides for customers with bespoke commitments.

### 4.4 Enforcement

The cap is enforced at the gateway, not the runner. The runner doesn't see tenant-aggregate cost; only the gateway does.

```python
# At the gateway, before every LLM call:
tenant_cost = cost_cache.get(tenant_id, window="minute")
if tenant_cost + estimated_call_cost > tenant_caps.per_minute:
    raise TenantBudgetExceeded(tenant_id, window="minute")
```

Cost-cache is approximated; reconciliation against actuals happens hourly. The approximation is intentional — exact accounting at every call is prohibitively expensive.

### 4.5 Breach behaviour

When a tenant cap is breached:

- The next LLM call returns a structured error to the runner: `{error: "tenant_budget_exceeded", retryable: false, message: "Your organisation has exceeded its hourly AI usage limit."}`.
- The runner treats it as a permanent failure (per [error-and-partial-failure.md](./error-and-partial-failure.md) section 3.3); the agent terminates gracefully.
- The user sees a clear message ("Your organisation's AI usage limit has been reached; this resets at HH:MM.").
- An alert fires for the tenant's account manager.

### 4.6 Per-tenant overrides during incidents

The tenant cap is configurable at runtime. During a cost incident on a single tenant:

- On-call can tighten the tenant's cap (e.g., reduce per-minute from $50 to $5) without affecting other tenants.
- The incident is contained to the offending tenant; other tenants continue normally.
- The cap can be restored once the incident is resolved.

This is the most operationally valuable feature of per-tenant caps: blast radius isolation.

### 4.7 Per-user caps within a tenant

For high-tenant-count features, the per-user cap is a finer-grained control. A misbehaving user (or a compromised account) can be capped without affecting their colleagues. The mechanism is the same shape as per-tenant; the scope is the user.

---

## 5. Cost-as-circuit-breaker — the gateway integration

The agent runner is the first defense; the gateway is the backstop.

### 5.1 The gateway-side breaker (recap)

Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md), the gateway implements four layers:

- Per-interaction (per request).
- Per-session.
- Per-tenant.
- Per-feature.

The agent's per-request budget aligns with per-interaction; per-tenant caps align with per-tenant. Per-session and per-feature add additional defenses.

### 5.2 Per-feature caps

A feature-level cap protects against:

- A misbehaving deploy: one feature's prompt or tool change causes a cost regression; the cap prevents company-wide impact.
- A new feature with unknown cost behaviour: the cap is set conservatively until production data justifies higher.
- A feature whose product owner has agreed to a cost ceiling.

For Meridian: care-coordinator capped at $500/min company-wide; summary-generator at $200/min; analytics-copilot at $1000/min.

### 5.3 Per-session caps

A session is a multi-request conversation. A session cap prevents:

- A single user's long conversation from spending disproportionately.
- An abuser who tries to drain the per-tenant cap via many small interactions in a short session.

For Meridian's care-coordinator: $5/session.

### 5.4 The breaker hierarchy

```
LLM call request
  → per-feature cap check
  → per-tenant cap check
  → per-session cap check
  → per-interaction cap check
  → make call
```

The most specific check fires first. The per-feature is the broadest; per-interaction the most granular. Any breach short-circuits.

### 5.5 The runner-gateway contract

The runner sends each LLM call with metadata:

```json
{
  "model": "claude-sonnet-4-6",
  "messages": [...],
  "metadata": {
    "tenant_id": "uuid-xxx",
    "feature": "care-coordinator",
    "user_id": "uuid-yyy",
    "session_id": "uuid-zzz",
    "trace_id": "abc",
    "agent_invocation_id": "uuid-www",
    "estimated_input_tokens": 4200,
    "estimated_max_output_tokens": 800
  }
}
```

The gateway uses metadata for attribution and budget checking. The runner uses the gateway's structured error responses for budget enforcement.

### 5.6 The breaker as the last word

The runner's per-request budget should fire before the gateway's breaker. If the gateway fires first, that indicates the runner's budget is not calibrated correctly (too loose) or is not enforced (bug). Either way, an alert fires for the misconfiguration.

---

## 6. Tier routing inside the loop

The single highest-leverage cost optimisation for most agents.

### 6.1 The observation

Inside an agent loop, turns vary in difficulty:

- **Orchestration turns.** The model decides which tool to call. The decision space is small (5–15 tools); the right answer is usually obvious; the model's reasoning needs are minimal.
- **Specialist turns.** The model is producing the final answer, synthesising from multiple sources, or making a hard judgement call. Reasoning needs are high.

A frontier model (claude-opus-4-7) handles both well but charges frontier prices for both. A smaller model (claude-haiku-4-5) handles orchestration turns well at a fraction of the cost but may underperform on specialist turns.

### 6.2 The tier-routing pattern

Route each turn to the appropriate tier:

```python
def select_model_for_turn(state: AgentState, decision_complexity: str) -> str:
    if state.turn == 0:
        # First turn: planning, often complex
        return "claude-sonnet-4-6"
    if state.is_final_turn or state.needs_synthesis:
        return "claude-sonnet-4-6"  # or opus for hard synthesis
    if decision_complexity == "low":
        return "claude-haiku-4-5"
    return "claude-sonnet-4-6"
```

The routing logic depends on the loop's design. For supervisor/worker (per [multi-agent-coordination.md](./multi-agent-coordination.md) section 4), the supervisor runs on a cheaper model; workers vary.

### 6.3 The orchestrator-specialist pattern

A specific instance: the agent's loop runs on a cheap orchestrator model that decides which tool to call. When a tool is invoked that itself contains an LLM call (a "specialist" — summariser, analyser, generator), the specialist is the expensive model.

```
orchestrator turn (haiku): chooses tool "summarise_clinical_notes"
specialist tool (sonnet inside): produces high-quality summary
orchestrator turn (haiku): chooses tool "format_for_clinician"
specialist tool (sonnet inside): produces high-quality format
orchestrator turn (haiku): chooses to terminate
final response
```

The pattern reduces total cost by 50–70% in typical agent workloads while preserving quality where it matters.

### 6.4 The "always frontier" anti-pattern

Many teams ship agents on frontier models because the prototype used frontier. They never revisit. The cost is 3–5× what it could be without quality difference for orchestration turns.

The corrective: tier audit on every agent in production. For each turn type, evaluate quality on a smaller model. Promote downsized models where quality holds.

### 6.5 The "always cheap" anti-pattern

The opposite mistake: a team tries to save cost by running everything on a small model. Quality drops on hard cases. The team adds elaborate prompt engineering to compensate. The result: marginal cost saving for substantial quality loss and increased engineering investment.

The corrective: use the right model for each turn type. Cheap for easy, expensive for hard.

### 6.6 Calibrating the tier router

The router's rules are calibrated against the eval set:

- For each turn type (orchestration, retrieval-result-processing, synthesis, final-answer), evaluate the per-tier quality on golden cases.
- Pick the cheapest tier whose quality is within tolerance.
- Re-calibrate quarterly (models improve; cost rates change; quality bars may shift).

### 6.7 Per-tenant tier routing

Some tenants accept lower quality for lower cost; some demand frontier-everywhere for highest quality. The tier router supports per-tenant configuration:

```python
def select_model_for_turn(state, decision_complexity, tenant_tier):
    if tenant_tier == "enterprise_premium":
        return "claude-opus-4-7"  # always frontier
    if tenant_tier == "standard":
        return _standard_routing(state, decision_complexity)
    if tenant_tier == "value":
        return _value_routing(state, decision_complexity)
```

The per-tenant config is part of the tenant's profile and is updated alongside contractual changes.

### 6.8 Beyond tiering: prompt-caching, batch-mode

Adjacent cost levers worth mentioning:

- **Prompt caching.** When the agent's system prompt + tools + recent context is similar across calls, prompt caching can reduce input-token cost by 70–90% for cached portions. Aggressively applied to long stable prompts.
- **Batch mode.** For non-interactive agents (background tasks, batch analytics), provider batch APIs offer 30–50% discount with delayed completion. Eligible only for non-time-sensitive workloads.

These are not agent-specific but apply to agent loops naturally.

---

## 7. In-progress cost incident response

When the alert fires, what happens.

### 7.1 The incident shape

A cost incident in progress: a feature, tenant, or user is spending at 10–100× the expected rate. The clock is ticking; every minute of delay is more spend. The on-call response targets mean-time-to-mitigation under 15 minutes.

### 7.2 The detection signals

- **Per-feature spike alert.** Feature's cost rate jumps above threshold.
- **Per-tenant spike alert.** A tenant's cost rate jumps.
- **Budget-breach rate alert.** Per-request budget breaches > 1% over 15 minutes.
- **Gateway breaker activation rate.** Breaker firing more than baseline.
- **Trace-pattern alert.** Traces showing > N turns per request more than baseline (correlated with cost).

Each maps to a runbook section.

### 7.3 The first five minutes

Triage:

1. **Acknowledge the alert.** Stop the clock on the page; signal to others you're on it.
2. **Open the cost dashboard.** Identify which feature, which tenant, which user is the contributor.
3. **Open a trace sample.** Read 2–3 sample traces of the misbehaviour. Identify the pattern.

The dashboard and the traces together identify the failure mode within 5 minutes for known patterns; longer for novel patterns.

### 7.4 The next five minutes — mitigation

Apply the appropriate control:

- **Single-tenant runaway.** Tighten the tenant's per-minute cap. Optional: notify the tenant's account manager.
- **Single-user runaway within tenant.** Tighten the per-user cap or disable the user temporarily.
- **Feature-wide regression after recent deploy.** Rollback the deploy.
- **Unknown pattern.** Tighten the per-feature cap as a containment measure while investigation continues.

The authority to take these actions must rest with on-call. If the on-call engineer needs to escalate for approval, the 15-minute target is missed.

### 7.5 The next five minutes — communication

Status update to the appropriate audiences:

- **Internal team.** What's happening, what was done, ETA on resolution.
- **Affected tenants** (if user-visible degradation occurred). Brief, clear, with ETA.
- **Finance team** (if the incident is large enough). The bill will reflect this.

The communication template is in the runbook; on-call adapts.

### 7.6 The investigation phase

After mitigation, the investigation:

- Was this caused by a deploy? (Most cost incidents are.)
- Was this caused by a tool's upstream change? (A retrieval returning many more chunks than expected; an external API rate-limiting differently.)
- Was this caused by a misuse or abuse pattern? (A user discovering they can drain the budget intentionally.)
- Was this caused by a model regression? (A new model version producing longer outputs.)
- Was this caused by a prompt change? (A subtle change that altered behaviour at scale.)

Root cause leads to the fix.

### 7.7 The post-incident review

Standard format. Specific to cost incidents:

- **Detection time.** How long from incident start to alert? Target < 5 minutes.
- **Triage time.** How long from alert to root-cause identification? Target < 10 minutes.
- **Mitigation time.** How long from alert to mitigation? Target < 15 minutes.
- **Cost impact.** Total over-spend.
- **What controls failed.** Which budgets / breakers should have caught this earlier.
- **What controls to add.** New alerts, new caps, new validations.

The PIR drives updates to the runbook, the eval set, the observability surface, and the budget calibrations.

### 7.8 The "incident review of incidents" cadence

Quarterly, the cost-incident pattern is reviewed across all incidents:

- Are there recurring patterns? (Same kind of failure across features.)
- Are the controls maturing? (Time-to-mitigation trending down.)
- Are budgets calibrated? (Frequent legitimate breaches indicate too-tight; lack of breaches indicate too-loose.)
- Is the runbook current? (New incident patterns appended.)

The review feeds an annual cost-discipline maturity assessment.

---

## 8. Cost observability and dashboards

The visibility layer.

### 8.1 The per-feature dashboard

For each agent feature:

- **Cost per request (p50, p95, p99).** Trend over 30 days.
- **Total cost per hour / day.** Trend.
- **Cost by tier.** What fraction of cost is each model tier contributing.
- **Cost by tool.** What tools are contributing cost.
- **Per-tenant breakdown.** Top 10 tenants by cost; concentration ratio.
- **Budget breach rate.** % of requests breaching budget; trend.
- **Tier routing effectiveness.** Cost per request with tier routing vs cost per request without (calculated from canary or A/B).

### 8.2 The per-tenant dashboard

For tenant-facing views (or for on-call inspection of a specific tenant):

- **Cost over time** with the cap shown.
- **Cost by feature.**
- **Cost by user (top contributors).**
- **Breach events** (when caps were hit).

### 8.3 The cost-attribution chain

Per [cost-attribution.md](../cost-and-finops/cost-attribution.md), every LLM call's cost is attributed via span attributes. The dashboards roll up the span data. The pipeline:

```
LLM call → gateway records cost + attribution → span emitted → observability backend → dashboard query
```

The chain must be continuous; gaps mean unattributed cost, which means cost shows up on the finance invoice without a feature / tenant owner.

### 8.4 Cost alerts

- **Per-feature spike** (>2x rolling 7-day avg cost-per-minute for > 5 minutes).
- **Per-tenant spike** (>3x prior-day cost-per-hour).
- **Budget-breach rate spike** (>1% over 15 minutes for a feature).
- **Unattributed cost** (a non-trivial fraction of spend not attributed to a feature).
- **Tier-routing drift** (frontier-model usage as fraction of total has crept above baseline; suggests routing degraded).

### 8.5 The monthly review

Once a month, the cost team and the AI platform team review:

- Per-feature cost trends.
- Per-tenant cost trends (with chargeback / billing implications).
- Tier-routing effectiveness.
- Incidents in the period.
- Calibration updates needed.

The review feeds budget calibration and capacity planning.

### 8.6 Cost forecasting

Beyond observability, the team forecasts cost ahead of deploys:

- A new feature: estimated cost per request × expected traffic = monthly cost projection.
- A model upgrade: re-priced at new rates × current traffic mix.
- A prompt change: re-evaluated on golden traffic; cost delta extrapolated.

Forecasts are wrong; they're useful anyway because they reveal large surprises (a 5× cost-per-request increase from a model upgrade) before deploy.

---

## 9. Worked Meridian example

Meridian's agent-cost-control practice. The team has operated agentic features in production for ~14 months and has run two notable cost incidents.

### 9.1 The cost envelopes by feature

| Feature | p50 / req | p99 / req | Budget / req | Per-tenant per-hour cap |
| --- | --- | --- | --- | --- |
| Care-coordinator (hybrid agent) | $0.08 | $0.22 | $0.50 | $300 / $1200 (std/ent) |
| Patient-summary (workflow) | $0.12 | $0.18 | $0.30 | (less constrained) |
| Patient-API copilot (workflow) | $0.04 | $0.08 | $0.20 | (developer-tier flat fee) |
| Analytics-warehouse copilot (hybrid agent) | $0.14 | $0.52 | $1.50 | $1500 / $6000 (std/ent) |

The agent-shaped features have wider envelopes; their budgets are larger.

### 9.2 Tier routing for care-coordinator

The care-coordinator's hybrid runs:

- Outer workflow: classification + dispatch on `claude-haiku-4-5` ($0.005-0.01 per call).
- Inner agent step (when invoked): orchestration on `claude-haiku-4-5`; the per-domain specialist tools call `claude-sonnet-4-6` internally.
- Final response synthesis: `claude-sonnet-4-6`.

The cost decomposition for a typical care-coordinator request:

| Component | Calls | Cost |
| --- | --- | --- |
| Outer classification | 1 (haiku) | $0.01 |
| Agent step orchestration | 3–6 (haiku) | $0.03 |
| Specialist tool calls (internal LLM) | 1–3 (sonnet) | $0.04 |
| Final synthesis | 1 (sonnet) | $0.02 |
| Tool dispatch overhead (free local funcs) | 5–10 | $0 |
| **Total per request** | | **$0.08–$0.18 typical** |

The tier routing reduces what would be a $0.30 per-request cost (sonnet for everything) to $0.10 average — 65% reduction.

### 9.3 The per-tenant cap experience

Meridian standard-tier tenants: $300/hour. The cap is rarely hit; when hit, it's typically (a) a tenant with unusual concentrated usage (e.g., end-of-quarter analytics push), (b) a per-tenant misconfiguration (their LDAP integration is calling the API in a loop).

For case (a), the tenant's account manager has authority to grant a one-time cap raise for the period. For case (b), on-call investigates and notifies the tenant; the cap remains.

The cap has been raised mid-incident for one tenant (a major customer running a one-time data-migration through the analytics copilot); the raise was approved by the on-call manager and rolled back the next day.

### 9.4 The two notable incidents

**Incident 1 (Q3-25).** Care-coordinator cost-per-request rose from $0.10 to $0.45 average within 6 hours of a prompt change. Detection: per-feature spike alert at +6 hours. Triage: traces showed the agent looping on a specific patient-record pattern; the prompt change had subtly altered the agent's interpretation of an empty field. Mitigation: rollback the prompt; 18 minutes from alert to rollback. Total over-spend: ~$3,500.

Post-incident: the prompt-promotion gate was tightened to include a cost delta check (per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md)). A 2× cost increase on the canary set now blocks promotion.

**Incident 2 (Q1-26).** Analytics-warehouse copilot cost-per-request rose from $0.14 to $1.30 average for a single tenant. Detection: per-tenant spike alert at +4 hours. Triage: the tenant had configured a new automation that called the copilot 200 times per hour in a tight loop. Mitigation: temporary per-tenant cap reduction (from $1500/hr to $300/hr); call to the tenant explaining the misconfiguration; 11 minutes from alert to mitigation. Total over-spend: ~$1,800.

Post-incident: per-user caps within tenants were introduced (per section 4.7) to scope blast radius further; this would have caught the single-source misuse earlier.

### 9.5 The runbook structure

```
== Care-Coordinator Cost Runbook ==
Section 1: Per-feature cost spike
Section 2: Per-tenant cost spike
Section 3: Budget breach rate spike
Section 4: Gateway breaker activation rate
Section 5: Trace-pattern alert (turn count high)
Section 6: Unattributed cost detected
Section 7: Tier-routing drift
Section 8: Recent incidents (last 12 months)
```

Each section: symptoms, dashboard links, triage queries, common causes, mitigation options, escalation criteria, post-incident actions.

### 9.6 Quarterly review

Each quarter:

- Re-calibrate budgets against the period's p99.
- Audit tier-routing effectiveness (any drift toward expensive tiers?).
- Review incidents; update runbook.
- Forecast cost for upcoming product changes.
- Report to finance: per-tenant cost (informs renewals / pricing); per-feature cost (informs product investment).

### 9.7 The discipline that prevented worse incidents

The two incidents in 14 months were each < $4k. Pre-discipline, similar incidents at similar scale could have been 5–10× larger. The investments that paid off:

- Tight runner budgets (the runaway in incident 1 was capped at $0.50 per request; the multiplier of ~4× was contained per-request).
- Tight per-tenant caps (incident 2's tenant was capped; the over-spend was bounded by the cap, not unbounded).
- Per-tier dashboards (the diagnosis was fast because the tier breakdown showed the regression immediately).
- Authority-to-act on on-call (no escalation chain for the rollback or cap adjustments).
- Eval-gate cost checks (incident 1's prompt change would now block at promotion gate).

---

## 10. Anti-patterns

### 10.1 "Budgets are aspirational"

The team writes a budget into a planning doc; no enforcement. The first cost incident reveals the budget was unenforced.

**Corrective.** Budgets are enforced at the runner and at the gateway. Budgets that aren't enforced don't exist.

### 10.2 "Frontier model everywhere"

The agent runs on the largest model for every turn. Cost is 3–5× what it could be.

**Corrective.** Tier routing per section 6. Audit; downsize where quality holds.

### 10.3 "No per-tenant caps"

The system has per-request budgets but no per-tenant aggregate caps. A misbehaving tenant burns through the company-wide capacity.

**Corrective.** Per-tenant caps per section 4. Multiple windows (minute, hour, day, month).

### 10.4 "No per-user caps within tenant"

A single user's misbehaviour or compromise affects the entire tenant.

**Corrective.** Per-user caps per section 4.7.

### 10.5 "Alerts but no authority"

Alerts fire; on-call cannot act without manager approval. Mean-time-to-mitigation exceeds 60 minutes.

**Corrective.** On-call has authority to tighten caps, rollback deploys, and disable users for cost incidents. Authority is documented.

### 10.6 "Cost not attributed"

The cost shows up on the invoice without a feature or tenant owner. Triage is impossible; chargeback is impossible.

**Corrective.** Attribution via span metadata per [cost-attribution.md](../cost-and-finops/cost-attribution.md). Pipeline continuous.

### 10.7 "Reactive only"

Cost is observed monthly. Incidents are discovered after the spending has happened.

**Corrective.** Real-time cost dashboards; spike alerts; sub-15-minute MTTM.

### 10.8 "Prompt and tool changes don't include cost review"

A prompt change ships without cost analysis; production reveals 2× cost regression.

**Corrective.** Promotion gate includes cost delta check on canary set; > X% increase blocks promotion.

---

## 11. Findings (sprint-assignable)

### AGT-COST-001 — Severity: Critical
**Finding.** Agent's per-request cost budget is not enforced (advisory only or absent).
**Recommendation.** Wire the runner's budget enforcement per section 3.3; gateway-side breaker as backstop.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-COST-002 — Severity: Critical
**Finding.** No per-tenant cost caps; a single tenant can exhaust company-wide capacity.
**Recommendation.** Per-tenant caps per section 4; multiple windows (minute, hour, day, month).
**Owner.** ai-platform-eng, sprint N+1.

### AGT-COST-003 — Severity: Critical
**Finding.** Cost is not attributed at LLM-call level; cost incidents cannot be triaged.
**Recommendation.** Attribution metadata per [cost-attribution.md](../cost-and-finops/cost-attribution.md); span attributes; dashboards.
**Owner.** ai-platform-eng + ops, sprint N+1.

### AGT-COST-004 — Severity: High
**Finding.** Agent runs on the largest available model for every turn; cost is 3–5× achievable.
**Recommendation.** Tier routing per section 6; orchestration on cheap model; specialists on expensive.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-COST-005 — Severity: High
**Finding.** On-call lacks authority to tighten caps or rollback deploys without escalation; MTTM > 60 minutes.
**Recommendation.** Authority documented per section 7.4; on-call empowered.
**Owner.** ai-platform-eng + leadership, sprint N+2.

### AGT-COST-006 — Severity: High
**Finding.** Prompt and tool changes promote without cost-delta check; cost regressions reach production.
**Recommendation.** Promotion gate cost-delta check per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md); > X% blocks.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-COST-007 — Severity: High
**Finding.** Per-feature cost spike alert is not configured; incidents are detected only by invoice review.
**Recommendation.** Real-time spike alerts per section 8.4; sub-5-minute detection.
**Owner.** ai-platform-eng + ops, sprint N+2.

### AGT-COST-008 — Severity: High
**Finding.** Tool calls with cost (LLM-as-tool, paid API) do not report cost back; runner underestimates request cost.
**Recommendation.** Per-tool cost reporting per section 3.5; runner accumulates.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-COST-009 — Severity: Medium
**Finding.** Tier-routing rules are static; no quarterly review against eval.
**Recommendation.** Calibration cadence per section 6.6.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-COST-010 — Severity: Medium
**Finding.** No per-user caps within tenants; single-user issues affect whole tenant.
**Recommendation.** Per-user caps per section 4.7.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-COST-011 — Severity: Medium
**Finding.** Cost runbook is absent; on-call response is improvised.
**Recommendation.** Per-section runbook per section 9.5; updated after each incident.
**Owner.** ops + ai-platform-eng, sprint N+3.

### AGT-COST-012 — Severity: Medium
**Finding.** Per-tenant dashboards are absent; tenant cost conversations are slow.
**Recommendation.** Per-tenant view per section 8.2; available to account managers.
**Owner.** ai-platform-eng + ops, sprint N+3.

### AGT-COST-013 — Severity: Medium
**Finding.** Near-breach signal not implemented; agent doesn't get a chance to terminate gracefully before hard breach.
**Recommendation.** Near-breach signal per section 3.4; prompt incorporates the signal.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-COST-014 — Severity: Medium
**Finding.** Per-feature company-wide cap not configured; feature regression can affect company-wide budget.
**Recommendation.** Per-feature cap per section 5.2.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-COST-015 — Severity: Low
**Finding.** Prompt caching is not configured; cacheable system prompts paid full rate.
**Recommendation.** Prompt caching per section 6.8; eligible system prompts cached.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-COST-016 — Severity: Low
**Finding.** Cost forecasts are not produced; product changes ship without cost projection.
**Recommendation.** Forecast per section 8.6; included in change-review template.
**Owner.** ai-platform-eng + product, sprint N+5.

### AGT-COST-017 — Severity: Low
**Finding.** Per-tier cost contribution not tracked; tier-drift undetected.
**Recommendation.** Tier-cost metric per section 8.1; tier-drift alert per section 8.4.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-COST-018 — Severity: Low
**Finding.** Quarterly cost-discipline review not scheduled; calibration drifts.
**Recommendation.** Quarterly review per section 9.6; updates to budgets, caps, routing rules.
**Owner.** ai-platform-eng + finance, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team launching a new agent:

- [ ] **Sprint 0 — cost decomposition.** Estimate per-request cost using sample traffic and prototype.
- [ ] **Sprint 0 — budget calibration.** Per-request budget at 2–4× p99 estimate.
- [ ] **Sprint 0 — per-tenant cap.** Calibrate against contractual commitments.
- [ ] **Sprint 1 — runner enforcement.** Per-request budget enforced at the runner.
- [ ] **Sprint 1 — attribution.** Span metadata; gateway records.
- [ ] **Sprint 1 — gateway breaker integration.** Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).
- [ ] **Sprint 2 — tier routing.** Orchestrator on cheap model; specialists on expensive.
- [ ] **Sprint 2 — dashboards.** Per-feature, per-tenant; live and queryable.
- [ ] **Sprint 2 — alerts.** Per-feature spike, per-tenant spike, breach-rate.
- [ ] **Sprint 3 — runbook.** Per-section incident response.
- [ ] **Sprint 3 — authority.** On-call empowered to tighten caps and rollback.
- [ ] **Sprint 3 — promotion gate.** Cost-delta check on canary; blocks regressions.
- [ ] **Ongoing — quarterly review.** Budgets, caps, tier routing, runbook.

For a team retrofitting cost control on an existing agent:

- [ ] **Sprint 0 — measurement.** What's the current cost? Per request, per tenant, per feature, per tier.
- [ ] **Sprint 0 — gap analysis.** What controls are missing.
- [ ] **Sprint 1 — fix the worst gap.** Often the per-tenant cap or the per-request budget enforcement.
- [ ] **Sprint 1 — attribution.** Get visibility before anything else.
- [ ] **Sprint 2 — tier routing.** Often the highest-leverage cost reduction.
- [ ] **Sprint 2 — runbook + authority.** Prepare for incidents that will happen.
- [ ] **Sprint 3 — promotion gate.** Prevent future cost regressions.
- [ ] **Sprint 4 — periodic review.** Schedule the quarterly cadence.

A team that completes the checklist has cost incidents bounded by minutes and dollars rather than hours and tens of thousands. A team that doesn't will run an incident the bill makes unforgettable.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 8 (cost control).
- [agent-loop-design.md](./agent-loop-design.md) — runner that enforces per-request budgets.
- [tool-architecture.md](./tool-architecture.md) — tools that report cost; tier routing inside tools.
- [memory-engineering.md](./memory-engineering.md) — memory that grows context and therefore cost.
- [error-and-partial-failure.md](./error-and-partial-failure.md) — failure handling that interacts with budget breaches.
- [multi-agent-coordination.md](./multi-agent-coordination.md) — cost decomposition for multi-agent (when applicable).
- [agent-observability.md](./agent-observability.md) — trajectory observability that feeds cost dashboards.
- [agent-evals.md](./agent-evals.md) — eval gate that includes cost-delta check.
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — gateway-side breaker primitive.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — attribution model and span metadata.
- [cost-and-finops/tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md) — broader tier-routing patterns.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — dashboard patterns.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alert patterns for cost.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — eval gate that protects against cost regressions.
- [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — model pinning that prevents silent cost changes from upgrades.
- Sibling repo: [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture) `cost-and-performance-architecture/` — architectural cost tradeoff frame (planned).
- Anthropic prompt caching, OpenAI batch API — provider features that interact with the patterns here.
