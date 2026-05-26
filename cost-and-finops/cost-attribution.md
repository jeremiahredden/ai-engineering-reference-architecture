# Cost Attribution

> **Audience.** Engineers building the cost telemetry layer. Tech leads who need cost-by-feature, cost-by-tenant, cost-by-model dashboards to manage AI spend. Finance partners who need the chargeback model to work. **Scope.** The *engineering* practice of attributing every LLM (and every cost-bearing tool) call to its feature, tenant, user, model, prompt-version, and time — at request time, not after the fact via vendor invoices. The storage and aggregation pattern. The integration with observability. Reconciliation with provider invoices. Chargeback. Not the cost-incident response (see [cost-incident-runbook.md](./cost-incident-runbook.md) *(coming)*). Not the circuit-breaker primitive (see [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Without cost attribution, AI spend is a single number on an invoice that arrives 30 days after the spend happened. The team knows the total; the team does not know which feature is responsible, which tenant drove the cost, which model generated which fraction, or which prompt version inflated the spend after the last deploy. Cost incidents take days to triage instead of minutes. Chargeback is impossible. Cost-controlled engineering decisions (tier routing, caching, prompt optimisation) cannot be validated because there's no per-feature signal to validate against.

The attribution layer is foundational. Every other cost discipline in this folder — circuit breakers, dashboards, alerts, tier routing decisions, caching investment decisions, per-tenant cost caps — depends on the team's ability to answer "where did this dollar go?" at fine-grained, near-real-time granularity. A team that hasn't built attribution cannot build any of the other disciplines effectively; they're operating in the dark and making decisions on aggregate hindsight.

In 2026, building attribution is not the heavy investment it once was. Provider APIs return per-call cost information directly (or the information needed to compute it). Observability tools support cost-as-attribute span data. The discipline is wrapping the LLM call site so that every call's cost is computed, attributed, persisted, and made queryable. The infrastructure investment fits within a sprint; the value compounds across every other cost discipline.

This document is opinionated about four things:

1. **Attribution is computed at request time, not after the fact.** The vendor invoice is the post-hoc confirmation; the engineering signal must be at call time so that decisions (terminate, degrade, alert, attribute) can be made before the call returns or shortly after.
2. **The attribution dimensions are non-negotiable.** Feature, tenant, user, model, prompt version, agent invocation, trace, timestamp. Each dimension is queryable. Dropping any dimension cripples a class of analysis.
3. **The wrapper is the source of truth.** Every LLM call goes through one wrapper that handles attribution; no calls bypass. Tools that incur cost also wrap; attribution propagates.
4. **Reconciliation with vendor invoices is part of the discipline.** The engineering attribution and the vendor invoice should match within a small tolerance. Discrepancies are investigated, not papered over.

Structure: (2) the attribution dimensions; (3) per-call cost computation at request time; (4) the wrapper pattern; (5) storage and aggregation; (6) integration with observability and traces; (7) reconciliation with vendor invoices; (8) the chargeback model; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The attribution dimensions

Each LLM call's cost is tagged with these dimensions. Each is queryable.

### 2.1 The required dimensions

| Dimension | What it answers | Example value |
| --- | --- | --- |
| `feature` | Which product feature consumed the cost | `care-coordinator` |
| `tenant_id` | Which customer paid for / consumed it | `uuid-tenant-a` |
| `user_id` | Which end-user's actions drove it | `uuid-user-b` |
| `model` | Which model (provider + variant + version) | `anthropic/claude-sonnet-4-6` |
| `prompt_version` | Which prompt version was used | `care-coord-system-v23` |
| `agent_invocation_id` | Which agent invocation (for agent features) | `uuid-inv-c` |
| `trace_id` | Which top-level request trace | `trace-abc` |
| `timestamp` | When the call happened | `2026-05-26T14:33:17.421Z` |
| `cost_usd` | The call's cost in USD | `0.04` |

These nine cover most analysis. Add more if the team has a recurring query that needs them (e.g., `deployment_id` for per-deploy analysis, `region` for region-specific cost).

### 2.2 The secondary dimensions

Useful but not required:

| Dimension | What it answers | Example value |
| --- | --- | --- |
| `tool_name` | Which tool (if cost is from an LLM-as-tool call) | `summarise_clinical_notes` |
| `session_id` | Which user session | `uuid-session-d` |
| `tenant_tier` | Tenant tier (standard / enterprise / etc.) | `enterprise` |
| `request_id` | The originating request | `uuid-req-e` |
| `cached` | Whether this call hit the prompt cache | `true` |
| `batch_mode` | Whether this call was via batch API | `false` |
| `latency_ms` | Call latency (for cost/latency analysis) | `1240` |

### 2.3 Dimensional analysis use cases

Each dimension enables specific analyses:

- `feature` → cost-by-feature dashboard; informs feature investment decisions.
- `tenant_id` → cost-by-tenant; informs chargeback; informs noisy-neighbor detection.
- `user_id` → cost-by-user; catches outlier users and abuse.
- `model` → cost-by-tier; validates tier routing.
- `prompt_version` → cost-by-prompt-version; catches regressions from prompt changes.
- `agent_invocation_id` → per-agent-invocation cost; catches runaway invocations.
- `trace_id` → end-to-end cost per request; joins with traces for deep analysis.
- `timestamp` → time-series cost; trend analysis.
- `cached` → cache hit rate; validates caching investment.
- `batch_mode` → batch vs realtime cost split.

The dimensions are queryable from the moment they're recorded; analysis is a query away.

### 2.4 What attribution does NOT include (carefully)

PII and sensitive content are NOT in the attribution dimensions. The cost record carries IDs (tenant_id, user_id, agent_invocation_id), not PII (no patient names, no clinical content, no payment data). The attribution layer is queryable broadly; sensitive content lives in the redacted trace data.

This separation is important for privacy compliance and for query performance. The attribution layer is high-volume, low-sensitivity; the trace data is high-sensitivity, lower-volume.

---

## 3. Per-call cost computation at request time

Cost must be computed when the call returns, not days later from the invoice.

### 3.1 The cost equation

For most provider APIs:

```
cost_usd = (input_tokens × input_token_rate)
        + (output_tokens × output_token_rate)
        + (cached_input_tokens × cached_token_rate)  // if prompt caching used
```

Each component is observable from the call's response:

- `input_tokens` and `output_tokens` are returned in the response payload (usage block).
- `cached_input_tokens` is returned if prompt caching was used.
- The rates are known per (model, region, contract) — typically held in a config file.

The wrapper applies the equation and produces `cost_usd`.

### 3.2 The rate table

A versioned config file mapping (model, region) to rates:

```yaml
- model: "anthropic/claude-sonnet-4-6"
  region: "us-east-1"
  input_per_million_tokens_usd: 3.00
  output_per_million_tokens_usd: 15.00
  cached_input_per_million_tokens_usd: 0.30
  effective_date: "2026-04-01"
- model: "anthropic/claude-haiku-4-5"
  region: "us-east-1"
  input_per_million_tokens_usd: 0.80
  output_per_million_tokens_usd: 4.00
  cached_input_per_million_tokens_usd: 0.08
  effective_date: "2026-04-01"
```

Updated when:

- The vendor changes rates (rare but happens).
- A new model variant is supported.
- A contractual rate (negotiated discount) changes.

The wrapper looks up the rate for the call's (model, region) at call time.

### 3.3 Non-LLM cost components

Some tools have cost beyond LLM tokens:

- **Paid external APIs.** The tool's implementation tracks the API's cost and returns it as part of the tool result. The agent runner accumulates.
- **LLM-as-tool calls.** The tool internally calls an LLM (e.g., a summariser); the inner LLM cost is recorded under the tool's name and bubbled up.
- **Compute-intensive tools.** Some tools (e.g., a sandboxed code execution tool) have measurable compute cost; the implementation tracks and reports.

The attribution layer accepts cost from sources beyond LLM provider calls. The unified `cost_usd` field carries all.

### 3.4 The "no cost recorded" failure mode

A call that doesn't record cost is invisible to the attribution layer. The next dashboard query will under-report. The discipline:

- Every LLM call goes through the wrapper; no bypass.
- The wrapper records cost or fails the call.
- A check in the observability pipeline counts LLM provider calls vs attributed cost records; discrepancy alerts.

The "no cost recorded" failure mode is the silent failure mode of attribution; engineer against it.

### 3.5 Cost estimation vs cost recording

Some operational decisions need a cost estimate *before* the call (e.g., budget checks). Estimation uses approximate input-token count + an output-token max budget. The recorded cost (after the call) is the actual.

Estimates and actuals are both useful:

- Estimates for budget checks pre-call.
- Actuals for attribution and dashboards post-call.
- Variance between estimate and actual is itself a useful signal (estimates that consistently under-report indicate calibration drift).

---

## 4. The wrapper pattern

The wrapper is the single source of truth for LLM call cost attribution. Engineering it well pays off across every dependent system.

### 4.1 The wrapper's responsibilities

- Accept the LLM call request with attribution metadata.
- Perform pre-call checks (budgets, rate limits, attribution validation).
- Make the actual provider API call (with retry policy).
- Compute cost from the response.
- Record the cost-attribution record.
- Emit the LLM call span (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)).
- Return the result to the caller.

### 4.2 The wrapper interface

```python
class LLMClient:
    def call(
        self,
        model: str,
        messages: list[Message],
        max_tokens: int | None = None,
        # Attribution metadata (required):
        feature: str,
        tenant_id: str,
        user_id: str,
        prompt_version: str,
        # Optional metadata:
        agent_invocation_id: str | None = None,
        trace_id: str | None = None,
        session_id: str | None = None,
        request_id: str | None = None,
        tool_name: str | None = None,  # if this call is from an LLM-as-tool
        # Other params...
        **provider_params,
    ) -> LLMResponse:
        ...
```

The required attribution arguments cannot be defaulted; the wrapper refuses calls without them.

### 4.3 The attribution validation

Pre-call validation:

- All required attribution fields present.
- `feature` is in the known features list.
- `tenant_id` is a valid tenant.
- `model` is in the rate table.
- `prompt_version` matches a known prompt version.

Invalid attribution fails the call with a structured error. This catches code paths that try to bypass attribution.

### 4.4 The wrapper as the gateway

In a typical architecture, the wrapper is also the gateway (per [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md)). The cost checks and budget enforcement happen at the same layer that records the attribution. The wrapper:

- Checks per-tenant cap; rejects if breached.
- Checks per-feature cap; rejects if breached.
- Performs the call.
- Records cost + attribution.
- Updates the cap usage.
- Returns the result.

Single layer; coherent responsibility.

### 4.5 Wrapper deployment patterns

Two common patterns:

**Pattern A: in-process library.** The wrapper is a library imported by every component that calls LLMs. Each component instantiates it; the library handles attribution and cost recording locally (often by writing to a metrics/logs pipeline).

**Pattern B: out-of-process gateway service.** A dedicated gateway service exposes the LLM call API; components call the gateway; the gateway handles attribution centrally.

Pattern A is simpler initially; Pattern B is more robust at scale (centralised enforcement; cross-language support; easier policy updates).

Most production deployments evolve from A to B as scale demands.

### 4.6 The bypass-prevention discipline

Calls that bypass the wrapper produce attribution gaps. Discipline:

- Static analysis: a lint rule that bans direct provider-SDK imports outside the wrapper.
- Architecture review: any new code that calls LLM providers directly is rejected at PR review.
- Observability: alert on attribution gap (provider calls per minute vs attribution records per minute).
- Periodic audit: scan codebase for direct-provider-SDK usage.

The bypass-prevention is what makes the wrapper the source of truth. Without it, attribution silently degrades.

---

## 5. Storage and aggregation

The cost records are a high-volume time-series. Engineer storage and aggregation deliberately.

### 5.1 The raw records

Each LLM call produces one cost-attribution record:

```json
{
  "timestamp": "2026-05-26T14:33:17.421Z",
  "feature": "care-coordinator",
  "tenant_id": "uuid-tenant-a",
  "user_id": "uuid-user-b",
  "model": "anthropic/claude-sonnet-4-6",
  "prompt_version": "care-coord-system-v23",
  "agent_invocation_id": "uuid-inv-c",
  "trace_id": "trace-abc",
  "session_id": "uuid-session-d",
  "input_tokens": 4210,
  "output_tokens": 412,
  "cached_input_tokens": 0,
  "cost_usd": 0.0188,
  "latency_ms": 1240,
  "cached": false,
  "batch_mode": false
}
```

At scale, this volume is significant: 1M LLM calls / day → 1M records / day → ~300 GB / year (at ~1 KB / record uncompressed).

### 5.2 Storage choices

Common options:

- **Time-series databases.** InfluxDB, TimescaleDB, ClickHouse, Druid, BigQuery. Strong for time-bucketed aggregation queries.
- **Log aggregators.** Loki, Datadog logs, Elasticsearch. Strong for ad-hoc queries; expensive for analytical queries.
- **Data warehouses.** Snowflake, BigQuery, Redshift. Strong for analytical queries; latency higher.
- **Combined.** Recent data (last 7 days) in time-series; archive in warehouse.

ClickHouse or BigQuery are common production choices for AI cost telemetry — both handle the scale, both support analytical queries efficiently, both integrate with dashboarding tools.

### 5.3 The aggregation layer

Raw records are heavy for dashboards. Pre-aggregate:

- **Per-minute rollup.** (feature, tenant, model, minute) → (call_count, total_cost, total_tokens).
- **Per-hour rollup.** Same grain, hourly buckets.
- **Per-day rollup.** Same grain, daily buckets.

Dashboards query the appropriate rollup for the time range; raw records only for deep drilldown.

### 5.4 Retention

- Raw records: 30–90 days (covers most ad-hoc queries).
- Per-minute rollup: 90 days.
- Per-hour rollup: 13 months (year-over-year).
- Per-day rollup: indefinite (small volume).

Aligned with privacy and observability retention policies.

### 5.5 The "single source of truth"

The cost-attribution storage is the canonical record. Dashboards, alerts, chargeback, and reconciliation all query this storage. Avoid parallel cost-tracking systems that diverge.

### 5.6 Performance considerations

A typical dashboard query (cost per feature in last 24h, grouped by hour) should return in < 2 seconds. Engineer the rollups and indexes to make this true.

Slow queries indicate either the schema is wrong or the rollups are insufficient. Investigate before users build workarounds (which fragment the attribution practice).

---

## 6. Integration with observability and traces

Attribution and observability are siblings. Joined together, they support deep analysis.

### 6.1 The shared trace_id

Every LLM call's attribution record includes the `trace_id`. The corresponding LLM call span (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) also includes `trace_id`. They are joinable.

Queries that join them:

- "Show me the trace for the most expensive request today."
- "Show me cost per agent invocation, with the trajectory."
- "Show me the per-tool cost contribution within trace abc."

### 6.2 The shared attribution attributes

Per-call cost-attribution dimensions appear as span attributes on the LLM call span. The observability tool's UI shows them; queries against the tool can return them.

Discipline: the attribute names match. `tenant_id` is `tenant_id` in both systems. Mismatched names produce broken joins.

### 6.3 The "cost per span" surface

Per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) and [agent-observability.md](../agent-engineering/agent-observability.md), spans for tools, turns, and agent invocations roll up cost. The roll-up:

- Each LLM call span has its `cost_usd`.
- The parent tool span sums the costs of LLM calls within its scope.
- The parent turn span sums the cost of the LLM call(s) and tool calls within the turn.
- The agent invocation span sums all turns.
- The request span sums all agent invocations.

The roll-up makes per-trace cost analysis possible without joining to the attribution storage for every query.

### 6.4 The "cost-in-trace" debugging workflow

When debugging a high-cost request:

1. Open the trace (per [agent-observability.md](../agent-engineering/agent-observability.md) section 7).
2. The trace shows per-span cost.
3. Identify the contributing component (which turn, which tool, which retrieval).
4. Drill into the component's behaviour.

Without cost-in-trace, the debugging requires separate dashboards and manual correlation; with it, the trace itself is the cost debugger.

### 6.5 Per-feature dashboards

Per [cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md). Standard panels:

- Cost over time (per minute, per hour).
- Cost by tenant (top 10).
- Cost by user (top 10).
- Cost by model tier.
- Cost by prompt version.
- Cost per request (p50, p95, p99).
- Cache hit rate impact.
- Batch vs realtime split.

Each panel is a query against the attribution storage; the rollups make queries fast.

### 6.6 The per-tenant view

Tenant-specific dashboards for account managers, sales, support:

- Tenant's cost over time.
- Tenant's cost by feature.
- Tenant's top users.
- Tenant's quota usage (against per-tenant caps).
- Tenant's incident history (cost-related).

Often gated behind appropriate access controls; not every internal team sees every tenant's data.

---

## 7. Reconciliation with vendor invoices

Engineering attribution and vendor invoices should match. The discipline that ensures they do.

### 7.1 Why reconcile

- **Validation.** The attribution is correct only if it matches the vendor's view of the spend.
- **Variance investigation.** A discrepancy may indicate bypass calls, cost calculation bugs, or contract complications.
- **Finance trust.** The internal cost numbers are the basis for chargeback; finance trusts them only after they're validated against the invoice.
- **Forecast calibration.** Reconciliation history calibrates forecasts.

### 7.2 The reconciliation cadence

Monthly:

1. Retrieve the vendor invoice (with sufficient detail: by model, by API key, by region, ideally with daily breakdown).
2. Aggregate the attribution storage over the same period.
3. Compare totals; investigate discrepancies.
4. Adjust the rate table or wrapper if the discrepancy is structural.
5. Report the reconciliation result to finance.

### 7.3 The reconciliation tolerance

Engineering attribution should match the vendor invoice within ~2% for mature deployments. Larger discrepancies warrant investigation:

- **5–10% under-attribution.** Bypass calls (the wrapper isn't catching all calls). Audit code paths for direct provider-SDK usage.
- **5–10% over-attribution.** Rate-table error (the wrapper is using outdated or wrong rates). Cross-check rates.
- **> 10% discrepancy.** Structural issue; deep investigation.

### 7.4 Vendor invoice limitations

Provider invoices have limitations:

- Often monthly, not daily — limits granular reconciliation.
- May lack per-prompt or per-feature breakdown (only per-API-key).
- May have provider-side adjustments (credits, discounts, errors) that aren't visible at call time.

The reconciliation accepts these limitations; it's a sanity check, not a precision match.

### 7.5 Per-API-key strategy

Some vendors charge per API key. Using separate keys per (feature, region) makes vendor-side breakdowns useful. The trade-off is more keys to manage; for cost-sensitive deployments, the trade-off is worth it.

### 7.6 The "vendor changed pricing" trigger

When a vendor changes pricing, the rate table is updated. The reconciliation following the change validates the wrapper applied the new rates correctly. Discrepancies indicate the rate-table update missed something (e.g., a model variant the wrapper supports but the update didn't cover).

---

## 8. The chargeback model

Cost attribution enables chargeback — pushing financial accountability to the team that controls the cost levers.

### 8.1 Why chargeback

- Aligns financial incentives with engineering decisions.
- Forces teams to consider cost when designing features.
- Provides a basis for per-tenant pricing decisions.
- Makes cost discussions concrete ("your feature spent $X this month").

### 8.2 The chargeback dimensions

Most common:

- **By feature / team.** Internal allocation; each team owns its features' cost.
- **By tenant.** Customer-facing; basis for usage-based pricing.
- **By project / initiative.** For specific initiatives that span teams.

The attribution dimensions support all of these; the chargeback model is a query on the attribution data.

### 8.3 The monthly chargeback statement

For each team / tenant:

- Total cost for the period.
- Cost breakdown by feature, by model, by day.
- Trend (vs prior period).
- Notes (incidents, special events).

The statement is the artefact for monthly cost reviews.

### 8.4 The chargeback's effect on engineering decisions

Once teams see their cost monthly:

- Prompt optimisation gets attention (a small per-call cost reduction × high call volume = real savings).
- Tier routing gets adopted (the team's own dashboard shows the savings).
- Caching gets adopted (same).
- Wasteful features get re-examined or retired.

The chargeback is itself a cost-control mechanism. Without it, cost is "someone else's problem"; with it, the team has skin in the game.

### 8.5 The shared infrastructure problem

Some cost is shared infrastructure (the cost of the wrapper / gateway itself, the cost of the attribution storage, the cost of the observability stack). Allocate this:

- Pro-rata (each feature pays proportional to its consumption).
- Per-team flat fee.
- Bundled into a platform charge.

The choice depends on org structure. Document the choice; finance audits it.

### 8.6 The "I don't agree with my chargeback" conversation

When a team disagrees with the chargeback, the attribution data is the evidence. The conversation is "let's look at the data" — specific records, specific spans, specific calls. The data resolves disputes quickly when the attribution discipline is strong.

When the attribution is weak (gaps, errors, broken reconciliation), the conversations become long and trust-damaging. The investment in attribution pays off here too.

---

## 9. Worked Meridian example

Meridian's cost attribution stack, in production for ~16 months.

### 9.1 Stack

- **Wrapper.** A custom Python library `meridian.llm_client` wraps Anthropic SDK; in-process across all components.
- **Gateway service.** Out-of-process gateway service `meridian-llm-gateway` for centralised policy; the wrapper library proxies through it.
- **Attribution storage.** ClickHouse cluster; records ingested via a Kafka topic.
- **Aggregations.** Materialised views in ClickHouse for per-minute, per-hour, per-day rollups.
- **Dashboards.** Grafana on ClickHouse; per-feature, per-tenant, per-model dashboards.
- **Reconciliation.** Monthly cron job comparing attribution vs vendor invoice.

### 9.2 Attribution dimensions

All nine required dimensions plus secondary:

- `cached` (prompt-cache hit / miss).
- `batch_mode` (false for all; Meridian doesn't currently use batch APIs).
- `tenant_tier`.
- `tool_name` (when call originates from a tool).

### 9.3 The rate table

Versioned YAML in the gateway service's config. Updated on every vendor pricing change (~2 per year typically). Effective-date tracking allows historical queries to compute cost at the rate in effect at the time.

### 9.4 The storage volume

- Raw records: ~3M / day.
- Storage: ~3 GB / day uncompressed; ~700 MB / day compressed in ClickHouse.
- Retention: raw 60 days, per-minute rollup 90 days, per-hour 13 months, per-day indefinite.
- Total ClickHouse footprint: ~70 GB.

### 9.5 The dashboards

Standard per-feature dashboard (the care-coordinator's example):

- Cost over time (per-hour, last 7 days).
- Cost by tenant (top 20).
- Cost by user (top 20).
- Cost by model tier (haiku / sonnet / opus split).
- Cost by prompt version (highlight: recent version changes).
- Cost per request (p50, p95, p99).
- Cache hit rate (panel with hit% over time).
- Tool cost contribution.

Cross-feature view:

- Total daily cost.
- Per-feature breakdown.
- Anomaly detection (vs baseline).

### 9.6 The reconciliation

Monthly, ~2nd business day after invoice arrives:

- Pull invoice data via Anthropic's API.
- Aggregate attribution for the prior calendar month.
- Compare totals at (model, region) grain.
- Variance < 1% over the last 12 months consistently.

The one significant variance (Q1-25, 4% under-attribution) traced to a batch script that bypassed the wrapper. Fixed with a wrapper-required lint rule; no recurrence.

### 9.7 The chargeback

Monthly chargeback per feature team. The statement:

- Total cost for the month.
- Per-feature breakdown.
- Per-tenant breakdown (top 20).
- Per-model breakdown.
- Trend vs prior month.
- Notes (any cost incidents).

The statement drives the monthly cost review meeting; ~30 minutes per team.

### 9.8 Effects observed

After 16 months of the chargeback discipline:

- Per-request cost on care-coordinator dropped from $0.28 (launch) to $0.10 (current) — ~65% reduction.
- Driven by: tier routing adoption (50% of savings), prompt optimisation (25%), prompt caching (15%), tool refactors (10%).
- Each was implemented by the feature team in response to their own cost data.

The team didn't need a central cost-reduction initiative; the chargeback created the incentive.

### 9.9 Incidents the attribution caught

Three significant cost incidents over 14 months; for each, the attribution data identified the cause in < 15 minutes:

- Q3-25: care-coordinator regression. Per-prompt-version dashboard showed the cost spike correlated with a specific prompt version; rollback fixed it.
- Q1-26: analytics-copilot tenant runaway. Per-tenant dashboard showed a single tenant's cost rate jumping 9×; per-user breakdown identified the misconfigured automation.
- Q1-26: care-coordinator small drift. Continuous monitoring of cost per request showed a 5% drift over 3 days; traced to a tool's upstream change.

Without the attribution dashboards, each incident's diagnosis time would have been hours, not minutes.

### 9.10 What didn't work initially

- **Pattern A only (in-process library).** Early deployment had only the library; multiple components had drift in how they used it. Moving to Pattern B (gateway service) centralised policy.
- **Single-key vendor account.** Initial setup used one Anthropic API key for everything; reconciliation lacked granularity. Split to per-feature keys; reconciliation became precise.
- **Rate table updates lagging vendor changes.** Initially manual; missed a vendor change once; reconciliation flagged it. Now automated alerts on vendor pricing-page changes.
- **Tenant-aware UI access too late.** Per-tenant dashboards were initially open to all engineers; tightened to need-to-know after a customer concern about visibility.

---

## 10. Anti-patterns

### 10.1 "Post-hoc only — wait for the invoice"

The team has no per-call cost telemetry; they wait for the monthly invoice. Cost decisions are made on stale aggregate data.

**Corrective.** Per-call attribution at request time per section 3. Real-time dashboards.

### 10.2 "Single dimension only"

The team has total cost (one number) but no breakdown. Cost incidents can't be triaged.

**Corrective.** Full nine-dimension attribution per section 2.

### 10.3 "Direct provider-SDK calls bypassing the wrapper"

Some code paths call the provider SDK directly; attribution doesn't see them. Reconciliation reveals discrepancy but the source is hard to identify.

**Corrective.** Bypass-prevention discipline per section 4.6.

### 10.4 "No reconciliation with vendor invoice"

Engineering attribution may be wrong; nobody checks. Finance distrusts the numbers.

**Corrective.** Monthly reconciliation per section 7.

### 10.5 "Attribution but no chargeback"

The attribution data exists but isn't consumed by teams. Cost discipline is the platform's problem, not the feature team's.

**Corrective.** Chargeback statements per section 8.3; monthly review.

### 10.6 "Attribution captures PII"

The cost records include patient names, clinical content, or other sensitive data. Privacy review (eventually) blocks broader access.

**Corrective.** IDs only per section 2.4; sensitive content in (redacted) trace data, not in attribution.

### 10.7 "Rate table not updated when vendor changes pricing"

The vendor changed rates; the wrapper still computes at old rates; cost estimates are wrong; reconciliation reveals the issue weeks later.

**Corrective.** Rate-table update process per section 3.2 + section 7.6; monitoring for vendor pricing changes.

### 10.8 "Dashboards slow because aggregations missing"

Dashboards query raw records; queries take 30+ seconds; users build workarounds.

**Corrective.** Pre-aggregation per section 5.3; query patterns calibrated against rollups.

---

## 11. Findings (sprint-assignable)

### COST-ATTR-001 — Severity: Critical
**Finding.** No per-call cost attribution; cost decisions made on monthly invoice.
**Recommendation.** Build wrapper + attribution per sections 3 and 4; one sprint.
**Owner.** ai-platform-eng, sprint N+1.

### COST-ATTR-002 — Severity: Critical
**Finding.** Direct provider-SDK calls bypass the wrapper; attribution gaps in reconciliation.
**Recommendation.** Bypass-prevention per section 4.6; lint rule + PR review + audit.
**Owner.** ai-platform-eng, sprint N+1.

### COST-ATTR-003 — Severity: Critical
**Finding.** Attribution captures PII / sensitive content; privacy concern.
**Recommendation.** IDs only per section 2.4; remove sensitive content from attribution records; backfill cleanup.
**Owner.** ai-platform-eng + privacy, sprint N+1.

### COST-ATTR-004 — Severity: High
**Finding.** No reconciliation with vendor invoice; attribution accuracy unverified.
**Recommendation.** Monthly reconciliation per section 7; tolerance < 2% target.
**Owner.** ai-platform-eng + finance, sprint N+2.

### COST-ATTR-005 — Severity: High
**Finding.** Chargeback statements not produced; teams not accountable for cost.
**Recommendation.** Monthly chargeback per section 8.3; monthly cost review meeting.
**Owner.** ai-platform-eng + finance, sprint N+2.

### COST-ATTR-006 — Severity: High
**Finding.** Per-tenant and per-feature dashboards absent; incident triage slow.
**Recommendation.** Standard dashboards per section 6.5 / 6.6.
**Owner.** ai-platform-eng + ops, sprint N+2.

### COST-ATTR-007 — Severity: High
**Finding.** Attribution storage doesn't pre-aggregate; dashboards slow.
**Recommendation.** Pre-aggregation per section 5.3; rollups for per-minute, per-hour, per-day.
**Owner.** ai-platform-eng, sprint N+2.

### COST-ATTR-008 — Severity: Medium
**Finding.** Rate table is informal / undocumented; updates missed.
**Recommendation.** Versioned rate table per section 3.2; effective-date tracking; vendor-price-change monitoring.
**Owner.** ai-platform-eng, sprint N+3.

### COST-ATTR-009 — Severity: Medium
**Finding.** Tool cost (LLM-as-tool, paid APIs) not captured; cost dashboard understated.
**Recommendation.** Non-LLM cost components per section 3.3; tool implementations report cost.
**Owner.** ai-platform-eng, sprint N+3.

### COST-ATTR-010 — Severity: Medium
**Finding.** Attribution and observability use different attribute names; joins broken.
**Recommendation.** Shared attribute schema per section 6.2; consistency across systems.
**Owner.** ai-platform-eng, sprint N+3.

### COST-ATTR-011 — Severity: Medium
**Finding.** Cost-per-span roll-up not implemented; per-trace cost analysis requires external joins.
**Recommendation.** Cost roll-up per section 6.3; cost as span attribute on parent spans.
**Owner.** ai-platform-eng, sprint N+3.

### COST-ATTR-012 — Severity: Medium
**Finding.** Single vendor API key; reconciliation lacks granularity.
**Recommendation.** Per-feature or per-region keys per section 7.5.
**Owner.** ai-platform-eng + ops, sprint N+3.

### COST-ATTR-013 — Severity: Medium
**Finding.** Wrapper as in-process library only; policy enforcement inconsistent across components.
**Recommendation.** Out-of-process gateway per section 4.5 Pattern B; centralised policy.
**Owner.** ai-platform-eng, sprint N+4.

### COST-ATTR-014 — Severity: Medium
**Finding.** Retention policies for attribution not aligned with privacy / observability; data retention risk.
**Recommendation.** Retention per section 5.4; aligned with privacy policy.
**Owner.** ai-platform-eng + privacy, sprint N+4.

### COST-ATTR-015 — Severity: Low
**Finding.** Cache-hit-rate not tracked in attribution; caching effectiveness opaque.
**Recommendation.** `cached` field per section 2.2; cache-hit-rate panel in dashboards.
**Owner.** ai-platform-eng, sprint N+4.

### COST-ATTR-016 — Severity: Low
**Finding.** Cost estimation pre-call not implemented; budget checks rely on post-call data.
**Recommendation.** Cost estimation per section 3.5; estimate-vs-actual variance tracked.
**Owner.** ai-platform-eng, sprint N+5.

### COST-ATTR-017 — Severity: Low
**Finding.** Vendor invoice reconciliation discrepancies not investigated; small variances accumulate.
**Recommendation.** Investigation discipline per section 7.3; record root causes.
**Owner.** ai-platform-eng + finance, sprint N+5.

### COST-ATTR-018 — Severity: Low
**Finding.** Shared infrastructure cost (gateway, storage, observability) not allocated; chargeback incomplete.
**Recommendation.** Allocation per section 8.5; documented; reviewed annually.
**Owner.** ai-platform-eng + finance, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team starting from scratch:

- [ ] **Sprint 0 — wrapper design.** Define the wrapper interface; required attribution dimensions; rate table format.
- [ ] **Sprint 1 — wrapper implementation.** In-process library; computes cost; emits records.
- [ ] **Sprint 1 — storage.** ClickHouse / BigQuery / TimescaleDB; ingestion pipeline.
- [ ] **Sprint 1 — rate table.** Initial population; versioned; deploy process.
- [ ] **Sprint 2 — dashboards.** Per-feature, per-tenant, per-model.
- [ ] **Sprint 2 — bypass-prevention.** Lint rule + audit + alert.
- [ ] **Sprint 2 — observability integration.** Cost on LLM-call spans; trace_id consistency.
- [ ] **Sprint 3 — reconciliation.** Monthly cron job; first reconciliation; investigation if needed.
- [ ] **Sprint 3 — chargeback.** First chargeback statement; review meeting cadence.
- [ ] **Sprint 4 — gateway service.** If scale warrants Pattern B.
- [ ] **Sprint 4 — pre-aggregation.** Rollups for fast dashboards.
- [ ] **Sprint 5 — tool cost integration.** Non-LLM tool cost flows to attribution.
- [ ] **Ongoing — quarterly review.** Rate table, reconciliation tolerance, dashboard usefulness.

For a team retrofitting attribution on existing AI features:

- [ ] **Sprint 0 — gap analysis.** What's instrumented today? What's the coverage?
- [ ] **Sprint 0 — wrapper.** Centralised wrapper that wraps existing call paths.
- [ ] **Sprint 1 — backfill or accept gap.** Decide: backfill historical attribution from logs, or accept the gap and start fresh.
- [ ] **Sprint 1 — migrate critical paths.** Move the highest-cost code paths to use the wrapper first.
- [ ] **Sprint 2 — storage + dashboards.** Get visibility.
- [ ] **Sprint 3 — reconciliation.** Validate attribution accuracy.
- [ ] **Sprint 4 — chargeback.** Drive accountability.

A team that completes the sequence has cost as a first-class engineering metric, manageable at the per-feature and per-tenant level. A team that doesn't has spend that grows unchecked and incidents that diagnose slowly.

---

## 13. References

- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — the gateway-side breaker that consumes attribution.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — tier routing that's validated against the per-tier attribution.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) *(coming)* — the dashboards and alerts that query attribution data.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) *(coming)* — per-tenant caps and chargeback that depend on per-tenant attribution.
- [caching-for-cost.md](./caching-for-cost.md) *(coming)* — caching whose hit rate is in attribution.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) *(coming)* — batch-mode flagged in attribution.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) *(coming)* — rate limiting that uses cost as the metric.
- [cost-incident-runbook.md](./cost-incident-runbook.md) *(coming)* — incident response that queries attribution.
- [finops-process.md](./finops-process.md) *(coming)* — the cross-functional process that consumes chargeback statements.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — span shape for LLM calls; attributes align with attribution.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — agent spans that roll up cost.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — dashboard patterns.
- [agent-engineering/agent-cost-control.md](../agent-engineering/agent-cost-control.md) — agent-side cost discipline that consumes attribution.
- [agent-engineering/agent-observability.md](../agent-engineering/agent-observability.md) — trace observability that joins with attribution.
- Sibling repo: [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture) `cost-and-performance-architecture/` — architectural cost framing (planned).
- ClickHouse, BigQuery, TimescaleDB documentation — storage choices for attribution data.
- FinOps Foundation framework — general FinOps reference; this document is the AI-specific overlay.
