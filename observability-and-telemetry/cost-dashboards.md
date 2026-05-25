# Cost Dashboards

> **Audience.** Engineers and FinOps practitioners building the cost-observability surface for AI workloads. Tech leads who have been asked "where is the cost going?" and couldn't answer. **Scope.** The *engineering* practice of cost dashboards for AI — dimensions to track, drill-down patterns, attribution accuracy, integration with cost circuits. Pair with [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md), [llm-call-instrumentation.md](./llm-call-instrumentation.md), [alerting-and-paging-design.md](./alerting-and-paging-design.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI cost dashboards are different from general cloud-cost dashboards. The latter aggregate by service, region, account; the former need per-feature, per-tenant, per-model, per-prompt-version dimensions to be actionable. A team that ships an AWS Cost Explorer view as their AI dashboard has the right intent but the wrong dimensions; cost incidents are not diagnosable.

The discipline this document codifies: cost dashboards are themselves engineered — the attribution pipeline produces per-call telemetry, dashboards aggregate at the right dimensions, drill-down supports incident diagnosis. Without engineering, cost is a monthly invoice surprise; with engineering, cost is an operationally-managed metric.

This document is opinionated about three things:

1. **Per-call cost attribution is the foundation.** Per [llm-call-instrumentation.md](./llm-call-instrumentation.md), every call records cost at call time. Dashboards aggregate; they don't compute.
2. **Multiple dimensions matter.** Per-feature, per-tenant, per-model, per-prompt-version, per-question-class. Each dimension supports a different operational question. The dashboard supports drill-down across them.
3. **Cost dashboards drive cost circuits.** Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md), circuits trip based on aggregate cost; the dashboard is where the team sees the cost-against-budget trajectory.

Structure: (2) the cost dimension taxonomy; (3) the per-call cost telemetry; (4) the aggregation pipeline; (5) the dashboard layouts; (6) drill-down patterns; (7) attribution accuracy; (8) integration with cost circuits; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The cost dimension taxonomy

The dimensions cost dashboards aggregate across.

### 2.1 The primary dimensions

- **Per-feature.** Which AI feature (Care Coordinator chat, patient-API assist, analytics copilot) is consuming cost.
- **Per-tenant.** Which tenant is driving cost.
- **Per-model.** Which models (Opus / Sonnet / Haiku, plus specialty models like rerankers and embedders) are consuming cost.
- **Per-prompt-version.** Which prompt version is driving cost (useful for catching cost-bloat from prompt changes).

### 2.2 The secondary dimensions

- **Per-question-class.** Within a feature, which question class costs the most.
- **Per-user-role.** Which user role (RN, physician, etc.) drives cost.
- **Per-session vs per-interaction.** Cost per session (full conversation) vs per interaction.
- **Per-hour-of-day.** Diurnal patterns.

### 2.3 The cost components

Within each dimension, cost decomposes:

- **Input tokens (uncached).**
- **Input tokens (cached).** Provider's prompt-prefix cache; typically 1/10th the cost.
- **Output tokens.**
- **Embedding API calls.**
- **Reranker API calls.**
- **Compute (for self-hosted inference).**

The decomposition supports diagnosis ("the cost rose because cache hit rate dropped" vs "the cost rose because of more output tokens").

### 2.4 The temporal dimensions

- **Per-hour.** Real-time cost rate.
- **Per-day.** Daily cost.
- **Per-week.** Weekly trend.
- **Per-month.** Monthly accumulation (matches provider invoice cycle).

Different temporal horizons surface different patterns.

### 2.5 The dimension combinations

The dashboard supports filtering and grouping across dimensions:

- "Cost per tenant, broken out by feature, last 7 days."
- "Cost per prompt version, for Care Coordinator, last 30 days."
- "Cost per question class, for the drug-interaction subset, last 24 hours."

The combinations are workload-specific; the dashboard accommodates the most-common queries.

---

## 3. The per-call cost telemetry

The foundation of the dashboards is per-call telemetry.

### 3.1 The cost computation

Per [llm-call-instrumentation.md](./llm-call-instrumentation.md):

- Every LLM call's cost is computed at call time.
- Input tokens, cached vs uncached, output tokens, model pricing applied.
- Cost is a span attribute (`ai.cost.usd`).

### 3.2 The attribution dimensions per call

Each call's trace carries:

- `ai.tenant.id` — tenant.
- `ai.feature.id` — feature.
- `ai.user.id` — user (when applicable).
- `ai.user.role` — role.
- `ai.session.id` — session.
- `ai.trace.interaction_id` — interaction.
- `ai.llm.provider` — provider.
- `ai.llm.model` — model.
- `ai.llm.model_version` — version.
- `ai.llm.prompt.version` — prompt version.

Each attribute supports a dimension in the dashboard.

### 3.3 The trace-to-cost pipeline

```
LLM call → trace span emitted with cost + dimensions
    │
    ▼
Trace export pipeline
    │
    ▼ (sampling)
Cost-aggregation pipeline (separate from trace storage)
    │
    ▼
Cost data store (queryable; pre-aggregated)
    │
    ▼
Dashboards
```

Cost data is separate from trace data: traces are sampled (per [trace-and-span-design.md](./trace-and-span-design.md) section 5), but cost is not — every call's cost is captured, regardless of trace sampling.

### 3.4 The cost data store

- **Hot tier.** Recent (last 7-30 days); high-resolution; supports drill-down.
- **Cold tier.** Long-term (months to years); aggregated; supports trend analysis.

Storage cost is bounded; cost data is small compared to traces.

### 3.5 The aggregation cadence

Pre-aggregations at multiple granularities:

- **Per-minute.** For real-time cost-circuit monitoring.
- **Per-hour.** For dashboard panels.
- **Per-day.** For daily reports.
- **Per-month.** For invoice reconciliation.

Pre-aggregation makes dashboard queries fast.

---

## 4. The aggregation pipeline

How per-call cost becomes dashboard panels.

### 4.1 The aggregation primitives

For each (dimension combination, time bucket):

- Sum of cost.
- Count of calls.
- Mean cost per call.
- P50, P95, P99 of per-call cost.

Pre-computed per bucket; queried by the dashboard.

### 4.2 The streaming aggregation

Per-call costs stream from the trace pipeline. The aggregation:

- Increments per-dimension-bucket counters in real-time.
- Updates the dashboard within seconds-to-minutes.
- Supports cost-circuit decisions (per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)).

### 4.3 The batch aggregation

Daily / weekly / monthly aggregations:

- Compute summaries per dimension.
- Compute deltas (week-over-week, month-over-month).
- Generate reports for non-engineering consumers.

### 4.4 The reconciliation with invoices

Monthly:

- Compare the cost-pipeline's monthly total to the provider invoice.
- Discrepancy > 2% triggers investigation:
  - Pricing-table drift (provider changed price; team didn't update).
  - Missed calls (calls not going through the wrapper; uncaptured).
  - Per-call cost computation bug.

Reconciliation is the integrity check on the cost telemetry.

### 4.5 The retention

- Per-call detail: 30-90 days.
- Hourly aggregations: 1 year.
- Daily aggregations: 5 years.
- Monthly aggregations: 7+ years (matches financial retention).

Storage cost is small; retention supports historical analysis.

---

## 5. The dashboard layouts

The dashboards organized for operational use.

### 5.1 The operational dashboard

The "what's happening right now" view:

- **Real-time cost rate** (per-minute aggregation).
- **Today's cost vs daily budget** (with budget overlay).
- **Top features by cost** (current day).
- **Top tenants by cost** (current day; alerts on outliers).
- **Cost-circuit status** (any tripped circuits).

This is the on-call view; updates every few minutes.

### 5.2 The financial dashboard

The "what's the spend look like" view:

- **Monthly cost trend** (last 12 months).
- **Per-feature cost trend.**
- **Per-tenant cost trend** (for chargeback).
- **Cost per interaction trend** (efficiency over time).
- **Per-model cost split** (which models drive cost).

This is the FinOps view; updates daily.

### 5.3 The diagnostic dashboard

The "why is cost what it is" view:

- **Drill-down by feature → tenant → user → interaction.**
- **Per-prompt-version cost comparison.**
- **Per-question-class cost.**
- **Cache hit rate trend** (lower hit rate = higher cost).
- **Cost composition** (input cached vs uncached vs output vs embedding vs rerank).

Used when investigating cost anomalies.

### 5.4 The dashboard panel set

A typical dashboard:

```
+--------------------------------+--------------------------------+
| Real-time cost rate            | Today vs daily budget          |
| (per-minute spark)             | (gauge + alert if > 80%)       |
+--------------------------------+--------------------------------+
| Top features by cost today     | Top tenants by cost today      |
| (bar chart)                    | (bar chart with outlier flag)  |
+--------------------------------+--------------------------------+
| Per-feature cost trend (7d)    | Cost composition (7d)          |
| (line; one line per feature)   | (stacked area chart)           |
+--------------------------------+--------------------------------+
| Cost per interaction trend     | Cache hit rate trend           |
| (line; per-feature)            | (line; per-feature)            |
+--------------------------------+--------------------------------+
```

Layout is iterated; what's most-watched gets prime position.

### 5.5 The dashboard ownership

Per [retrieval-observability.md](../rag-engineering/retrieval-observability.md) section 3.5:

- Operational dashboard: ai-platform-eng + on-call.
- Financial dashboard: ai-platform-eng + finops.
- Diagnostic dashboard: ai-platform-eng.

Each dashboard has an owner who reviews weekly.

---

## 6. Drill-down patterns

When a cost anomaly fires, the drill-down workflow.

### 6.1 The cost-spike drill-down

Cost-spike alert fires:

1. **Open operational dashboard.** Identify which feature is driving the spike.
2. **Drill into the feature.** Per-tenant breakdown; is it one tenant or distributed?
3. **Drill into the tenant** (if applicable). Per-user breakdown; is it one user?
4. **Drill into per-interaction cost.** Are individual interactions expensive, or is volume up?
5. **For high per-interaction cost:** examine prompt-version cost (recent change?), model cost (model upgrade?), cache hit rate (drop?).

The drill-down ends with a specific diagnosis.

### 6.2 The prompt-version cost diagnostic

When a recent prompt change is suspected:

- Per-prompt-version cost panel.
- Compare current version's cost to prior version.
- If significantly higher: prompt change is the cause.
- Rollback per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md).

### 6.3 The model-version cost diagnostic

When a model version change is suspected:

- Per-model-version cost panel.
- Compare current to prior version.
- If significantly higher: model alias resolved to a new version (per the 2026-04-29 incident in [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)).
- Re-pin per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).

### 6.4 The tenant abuse diagnostic

When a single tenant drives cost:

- Per-user breakdown within the tenant.
- Per-interaction breakdown.
- Is one user driving most of it? Or is it broad volume?
- If broad: customer-success conversation about usage / pricing tier.
- If one user: investigate (compromised account? Automation accident?).

### 6.5 The cache-degradation diagnostic

When cache hit rate drops:

- Per-prompt-version cache hit rate trend.
- Did a recent prompt change invalidate the cache?
- Per [llm-call-instrumentation.md](./llm-call-instrumentation.md) section 7, cache utilization is per-call.

The diagnostic ends with a specific cause; remediation follows.

### 6.6 The drill-down latency

Drill-down should be fast — within seconds to a few minutes per query. The pre-aggregation pipeline per section 4.1 supports this.

If drill-down is slow: more aggressive pre-aggregation; or move to a different cost-data-store technology.

---

## 7. Attribution accuracy

The dashboards are only useful if attribution is accurate.

### 7.1 The attribution-accuracy verification

Monthly:

- Compare cost-pipeline totals to provider invoice totals.
- Discrepancy < 2% is acceptable.
- Discrepancy > 2% triggers investigation.

The verification keeps the dashboards trustworthy.

### 7.2 The common attribution gaps

- **Calls not going through the wrapper.** Per [llm-call-instrumentation.md](./llm-call-instrumentation.md) section 2.3, the wrapper is the chokepoint; bypasses miss attribution.
- **Pricing-table staleness.** Per section 4.4; provider price changes not reflected.
- **Missing dimensions.** Calls without proper feature/tenant tags; cost ends up "unattributed."
- **Sampling artifacts** (rare; the cost data is meant not to be sampled).

### 7.3 The unattributed bucket

Some cost is genuinely unattributed (system-level operations, internal tooling). The dashboard shows an "unattributed" bucket; trends in this bucket are watched (significant growth = attribution gap).

### 7.4 The wrapper enforcement

The chokepoint discipline per [llm-call-instrumentation.md](./llm-call-instrumentation.md):

- All LLM calls through the wrapper.
- Lint rule against direct SDK imports.
- Code review enforces.

Without enforcement, attribution gaps accumulate.

### 7.5 The dimension completeness check

Periodic check:

- What fraction of calls have all primary dimensions populated?
- Target: 99%+.
- Below: investigate missing-dimension code paths.

---

## 8. Integration with cost circuits

Cost dashboards and cost circuits operate together.

### 8.1 The visualization of circuit state

The dashboard shows:

- Per-circuit status (per-tenant per-day, per-feature per-day, etc.).
- Current consumption vs budget.
- Time-to-budget-breach (projected).
- Recently-tripped circuits (last 7 days).

### 8.2 The pre-trip warning

The dashboard panel shows:

- "Tenant X is at 75% of daily budget" — visible before circuit trip.
- "Feature Y is on pace to exceed daily budget by 6pm" — projection.

These visual cues let the team react before circuits fire.

### 8.3 The post-trip diagnostics

When a circuit trips:

- The drill-down workflow (per section 6) identifies the cause.
- The dashboard shows the cost trajectory that led to the trip.
- The remediation steps follow per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).

### 8.4 The budget-vs-actual reconciliation

For finops:

- Monthly: per-tenant budget vs actual spend.
- Per-tier: standard tenants vs premium tenants vs internal.
- Forecast: end-of-month projection.

The reconciliation supports pricing and contracting conversations.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The dashboard suite

Meridian's cost dashboards:

- **Operational** (`/dashboards/ai-cost-operational`). On-call view.
- **Financial** (`/dashboards/ai-cost-financial`). FinOps view; daily updates.
- **Diagnostic** (`/dashboards/ai-cost-diagnostic`). Drill-down for investigations.

Three views; same underlying data; different layouts.

### 9.2 The dimensions

Tracked dimensions:
- Per-feature (5 features in production).
- Per-tenant (240 standard tenants, 1 premium, 5 internal).
- Per-model (5 active model tiers).
- Per-prompt-version (~15 active versions across features).
- Per-question-class (per Care Coordinator's classifier).
- Per-role.
- Per-hour-of-day.

Each dimension supports filtering and grouping.

### 9.3 The 2026-04-29 cost incident drill-down

Recreating the cost incident (from [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) section 8.3):

1. **Operational dashboard**: Tier 1 alert; care-coordinator cost at 100% of daily budget.
2. **Drill into care-coordinator feature**: per-tenant breakdown is uniform (not one bad tenant); per-prompt-version cost is stable (no prompt change); per-model-version cost is elevated.
3. **Drill into per-model-version**: claude-opus-4-7 cost up ~30% per call from the prior week.
4. **Hypothesis**: model alias resolved to new version with different pricing.
5. **Confirm**: model registry shows the alias resolution; the previous specific version had different pricing.
6. **Remediation**: re-pin to the previous specific version; circuit lifts; cost normalizes.

The drill-down took ~5 minutes from alert to root cause.

### 9.4 The daily review

Daily morning review by ai-platform-eng team lead:

- Operational dashboard: any anomalies overnight?
- Financial dashboard: are we on track for the monthly budget?
- Anomalies investigated within the day.

### 9.5 The monthly reconciliation

Monthly:

- ai-platform-eng + finops review the cost dashboard's monthly totals.
- Compare to provider invoice.
- Discrepancy < 1% (well within tolerance).
- Per-feature cost reported to product / leadership.

### 9.6 The cost-per-interaction efficiency tracking

A specific dashboard panel:

- Care Coordinator chat: cost per interaction trend.
- Target: $0.18.
- Actual: $0.17-0.19 range; healthy.
- Drift would trigger investigation.

The efficiency metric is the team's KPI for cost-engineering work.

### 9.7 The cache hit rate

Per [llm-call-instrumentation.md](./llm-call-instrumentation.md) section 7:

- Care Coordinator supervisor cache hit rate: 87%.
- Classifier cache hit rate: 94% (small stable prompt).
- Drafting worker cache hit rate: 65% (more variable inputs).

Tracked weekly; drops trigger investigation.

### 9.8 The cost-circuit visualization

The operational dashboard shows:

- Per-tenant circuit gauges: tenant X at 45%, tenant Y at 80% (warning), tenant Z at 15%.
- Per-feature circuit gauges: care-coordinator at 65%, patient-api at 38%.
- Recent trips: 1 trip last week (tenant Y on standard daily budget); investigated; no recurrence.

### 9.9 The platform discipline

- Per-call cost attribution via the wrapper.
- Three dashboards (operational / financial / diagnostic).
- Daily review by ai-platform-eng lead.
- Monthly reconciliation with invoice.
- Drill-down workflow documented.

---

## 10. Anti-patterns

### 10.1 "Cloud-cost dashboard as AI cost dashboard"

AWS Cost Explorer view by service. Aggregates AI cost as "Anthropic API"; no per-feature, per-tenant, per-prompt-version dimension; diagnosis impossible.

**Corrective.** Per-call attribution + AI-specific dimensions per section 2.

### 10.2 "Monthly invoice as the cost monitor"

Cost is observed monthly via invoice. Cost incidents happen within hours; the monthly cadence misses them.

**Corrective.** Real-time cost dashboard + circuit breakers per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).

### 10.3 "Dashboard without drill-down"

Dashboard shows aggregate trends; cannot drill into per-feature, per-tenant, per-interaction.

**Corrective.** Drill-down per section 6.

### 10.4 "Attribution gaps unmeasured"

Significant cost is "unattributed"; team doesn't notice; cost-circuit decisions are based on incomplete data.

**Corrective.** Attribution accuracy verification per section 7.

### 10.5 "Pricing-table stale"

Provider changes pricing; cost-pipeline still uses old pricing; cost numbers are wrong.

**Corrective.** Monthly invoice reconciliation per section 4.4; pricing-table update PRs.

### 10.6 "Single dashboard for all consumers"

Operational, financial, and diagnostic needs share one cluttered dashboard; nobody can find what they need.

**Corrective.** Multiple dashboards per section 5; per-consumer layout.

### 10.7 "No prompt-version cost panel"

Cost spikes after a prompt change; team can't quickly identify the cause; spends hours investigating.

**Corrective.** Per-prompt-version dashboard panel per section 6.2.

### 10.8 "No reconciliation"

Cost-pipeline totals never compared to invoice; pricing drift or attribution gaps accumulate.

**Corrective.** Monthly reconciliation per section 4.4.

---

## 11. Findings (sprint-assignable)

### DASH-001 — Severity: Critical
**Finding.** No AI-specific cost dashboard; AI cost observed only via monthly cloud invoice.
**Recommendation.** Per-call telemetry per [llm-call-instrumentation.md](./llm-call-instrumentation.md); dashboards per section 5.
**Owner.** ai-platform-eng + observability-eng + finops, sprint N+1.

### DASH-002 — Severity: Critical
**Finding.** Cost attribution lacks dimensions; per-feature, per-tenant, per-prompt-version not tracked.
**Recommendation.** Dimensions per section 2 captured per call.
**Owner.** ai-platform-eng, sprint N+1.

### DASH-003 — Severity: High
**Finding.** Dashboard lacks drill-down; aggregates only.
**Recommendation.** Drill-down per section 6; supports diagnosis.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### DASH-004 — Severity: High
**Finding.** Monthly invoice reconciliation not done; pricing-table or attribution drift undetected.
**Recommendation.** Monthly reconciliation per section 4.4.
**Owner.** ai-platform-eng + finops, sprint N+2.

### DASH-005 — Severity: High
**Finding.** Unattributed cost bucket not tracked; gaps accumulate silently.
**Recommendation.** Unattributed bucket dashboard panel per section 7.3.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### DASH-006 — Severity: High
**Finding.** Cost-circuit state not visualized on dashboard.
**Recommendation.** Circuit status panel per section 8.1.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### DASH-007 — Severity: High
**Finding.** Per-prompt-version cost panel absent; prompt-induced cost regressions hard to diagnose.
**Recommendation.** Per-prompt-version panel per section 6.2.
**Owner.** ai-platform-eng, sprint N+3.

### DASH-008 — Severity: High
**Finding.** Per-model-version cost panel absent; model-alias drift hard to diagnose.
**Recommendation.** Per-model-version panel per section 6.3.
**Owner.** ai-platform-eng, sprint N+3.

### DASH-009 — Severity: High
**Finding.** Real-time cost rate dashboard absent; cost incidents detected only by circuit trips.
**Recommendation.** Real-time rate panel with budget overlay per section 5.1.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### DASH-010 — Severity: Medium
**Finding.** Dashboard owners not assigned; dashboards decay.
**Recommendation.** Per-dashboard ownership per section 5.5.
**Owner.** ai-platform-eng team lead, sprint N+3.

### DASH-011 — Severity: Medium
**Finding.** Cache-hit-rate panel absent; cache-utilization regressions invisible.
**Recommendation.** Per-feature cache hit rate panel per section 9.7.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### DASH-012 — Severity: Medium
**Finding.** Cost-per-interaction trend not tracked; efficiency over time invisible.
**Recommendation.** Cost-per-interaction panel per section 9.6.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### DASH-013 — Severity: Medium
**Finding.** Per-tenant cost ranking absent; tenant outliers undetected.
**Recommendation.** Top-tenants panel with outlier flagging per section 5.1.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### DASH-014 — Severity: Medium
**Finding.** Daily morning review not scheduled; anomalies linger.
**Recommendation.** Daily review per section 9.4.
**Owner.** ai-platform-eng team lead, sprint N+3.

### DASH-015 — Severity: Medium
**Finding.** Dimension completeness not measured; missing-tag code paths go undetected.
**Recommendation.** Completeness check per section 7.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### DASH-016 — Severity: Low
**Finding.** Cost data retention not aligned with regulatory; financial records may be lost too soon.
**Recommendation.** Tiered retention per section 4.5.
**Owner.** ai-platform-eng + finops + compliance, sprint N+5.

### DASH-017 — Severity: Low
**Finding.** Drill-down latency is slow; investigations are sluggish.
**Recommendation.** More aggressive pre-aggregation per section 4.1.
**Owner.** ai-platform-eng + observability-eng, sprint N+5.

### DASH-018 — Severity: Low
**Finding.** Dashboard documentation thin; panels unclear to non-engineers.
**Recommendation.** Tooltips + panel-level documentation.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building AI cost dashboards:

- [ ] **Sprint 0 — design.** Identify required dimensions; choose data store.
- [ ] **Sprint 1 — per-call telemetry.** Per [llm-call-instrumentation.md](./llm-call-instrumentation.md); attribution dimensions captured.
- [ ] **Sprint 1 — aggregation pipeline.** Per-hour, per-day pre-aggregations.
- [ ] **Sprint 2 — operational dashboard.** Real-time rate, top features, top tenants, circuit status.
- [ ] **Sprint 2 — drill-down.** Per-feature → per-tenant → per-interaction paths.
- [ ] **Sprint 3 — financial dashboard.** Monthly trend, per-feature, per-tenant for chargeback.
- [ ] **Sprint 3 — diagnostic dashboard.** Per-prompt-version, per-model-version, cache hit rate.
- [ ] **Sprint 4 — reconciliation.** Monthly invoice comparison; alert on drift.
- [ ] **Sprint 4 — dimension completeness.** Monitor; alert on attribution gaps.
- [ ] **Sprint 5 — daily review.** Cadence; ownership; action loop.
- [ ] **Ongoing — discipline.** Pricing-table updates; quarterly dashboard review.

A team that completes this sequence has the cost-observability surface that turns AI cost from a monthly surprise into an operationally-managed metric.

---

## 13. References

- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — per-call cost telemetry.
- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — trace structure.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](./alerting-and-paging-design.md) — cost alerts.
- This repo: [observability-and-telemetry/quality-drift-detection.md](./quality-drift-detection.md) — sibling drift discipline.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — circuit pattern this dashboard supports.
- This repo: [cost-and-finops/cost-attribution.md](../cost-and-finops/) (coming) — broader attribution patterns.
- This repo: [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/) (coming) — incident response.
- This repo: [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — model-version attribution source.
- This repo: [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) — prompt-version attribution source.
- This repo: [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — release manifest pinning.
- Sibling repo: [ai-architecture-reference-architecture/cost-and-performance-architecture/](https://github.com/jeremiahredden/ai-architecture-reference-architecture/tree/main/cost-and-performance-architecture) — architecture context.
- FinOps Foundation cost-observability practices.
