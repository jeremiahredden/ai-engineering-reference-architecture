# Cost and FinOps

## What this folder is

The engineering and operational practice of keeping AI cost predictable and attributable — token-level cost accounting, per-feature and per-tenant chargeback, cost-aware routing and caching, batch optimization, cost-budget-as-circuit-breaker, and the cost-incident response runbook. The material here is what I put in front of a leadership team when the question is: *the AI bill last month was 4x last quarter's projection, finance is upset, half the spend is in two features we did not know were that expensive, and we are about to launch three more — how do we get cost under engineering control?*

## The organizing principle

AI cost is engineering's problem, not finance's. The model-vendor invoice is a downstream consequence of architectural choices (which model tier, which caching tier, which retrieval depth), engineering choices (whether the prompt is bloated, whether retry is bounded, whether the agent has a turn budget), and operational choices (whether anyone is watching the cost line, whether anomalies page someone, whether there is a kill switch). Finance can ask for a number; only engineering can make the number be what was asked for.

So the patterns here treat AI cost as a *first-class engineering metric* alongside latency and quality. Cost has SLOs. Cost has alerts. Cost has runbooks. Cost has dashboards at the per-feature, per-tenant, per-user, and per-model granularity that supports diagnosis. Cost has a kill switch — the cost-budget-as-circuit-breaker that fails open or fails safe (depending on the feature) when the per-tenant or per-feature budget is exceeded. And cost has a chargeback model that pushes financial accountability to the team that controls the architectural and engineering levers.

The folder is opinionated about three things specifically. First, that *cost attribution to feature, tenant, and model is non-optional* — without it, cost incidents are not actionable. Second, that *the largest cost savings on most workloads come from caching and tier-routing*, not from negotiating with the vendor; engineering effort buys more cost reduction than procurement effort does. Third, that *cost-budget-as-circuit-breaker is the only reliable defense against runaway cost incidents* — every other defense relies on someone noticing in time.

## Planned documents

- **cost-attribution.md** *(coming)* — Per-call cost computation at request time (not after the fact via vendor invoices), the attribution dimensions (feature, tenant, user, model, prompt-version, time), the storage / aggregation pattern, and the integration with the sibling `observability-and-telemetry/` folder.
- **cost-dashboards-and-alerts.md** *(coming)* — The cost dashboards that support action: cost-per-feature trends, cost-per-tenant rankings (catches noisy-neighbor or abuse), cost-per-user distribution (catches outlier users), cost-per-model-tier (validates routing). The alert design: budget-burn-rate, spike-vs-trend, and the runbook each alert points at.
- **per-tenant-cost-control.md** *(coming)* — Per-tenant cost budgets and rate limits, premium-tier configuration (bigger budgets for premium tenants), the cost-budget-exceeded handling (fail-open with degraded mode, fail-closed with explicit error, or human-escalate), and the chargeback model integration.
- **caching-for-cost.md** *(coming)* — The four cost-reducing caching tiers and the engineering practice for each: prompt-prefix cache (configure provider-side, near-free), response cache (exact-match LRU, useful for FAQ-shaped workloads), semantic cache (embedding-similarity match), retrieval cache (cache retrieval results between requests). Cache-hit-rate as SLI and cache-invalidation discipline.
- **tier-routing-for-cost.md** *(coming)* — The engineering practice of cheap-first model routing with escalation on signal (low confidence, structured-output failure, complex query class). Typical 40–70% cost savings on tiered workloads. Integration with the architecture sibling's `model-strategy/`, and the eval discipline that validates routing does not degrade quality.
- **batch-vs-realtime-cost.md** *(coming)* — Batch APIs (Anthropic Batch, OpenAI Batch) for workloads that tolerate hours of latency in exchange for ~50% cost reduction. The decision framework, the integration patterns (async / queue / callback), and the workload classification that determines what is batch-eligible.
- **cost-aware-rate-limiting.md** *(coming)* — Rate limits that target cost rather than request count: token-budget-per-minute-per-tenant, cost-budget-per-day-per-feature, the priority-lane pattern (premium tenants get more budget headroom under contention). The interaction with the broader rate-limiting strategy.
- **cost-budget-circuit-breaker.md** *(coming)* — The cost-as-circuit-breaker pattern: per-feature daily / monthly budgets, automatic graceful degradation when the budget is approached, automatic feature-off when the budget is exceeded. The configuration discipline and the on-call runbook.
- **cost-incident-runbook.md** *(coming)* — Cost-incident response: detection (cost-spike alert), triage (which feature / tenant / model / prompt-version is responsible), mitigation (rate-limit, route-down-tier, kill switch), root cause (prompt bloat, retrieval bloat, agent loop, abuse, vendor pricing change), and prevention (the circuit-breaker that would have caught it earlier).
- **finops-process.md** *(coming)* — The cross-functional cadence that keeps AI cost in engineering's accountability loop: monthly cost review, per-team chargeback statements, the budget-vs-actual report for engineering leadership, and the launch-readiness cost gate that prevents a new feature from shipping without a budget and a kill switch.

## How to use this section

**If your team does not know which features cost what**, `cost-attribution.md` is the first move. The wrapper-pattern instrumentation can ship in a sprint and starts producing the signal that makes everything else in this folder possible.

**If your AI cost is growing faster than usage**, the diagnosis is usually in `caching-for-cost.md` or `tier-routing-for-cost.md` — most teams under-cache and under-route, leaving 50%+ of cost on the table.

**If you have had a cost incident** (one feature consumed 10x its expected budget in a weekend), `cost-incident-runbook.md` is the postmortem framework and `cost-budget-circuit-breaker.md` is the prevention.

## What this section is not

- **A vendor-pricing comparison.** Vendor pricing changes quickly; comparisons go stale. The patterns here are about how to engineer cost in whatever pricing environment is current.
- **A FinOps Foundation reference.** The general FinOps discipline is well-covered by the FinOps Foundation and its certification material. This folder is the AI-specific overlay — the practices that the general FinOps reference does not yet address in depth.
