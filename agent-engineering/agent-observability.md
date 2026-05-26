# Agent Observability

> **Audience.** Engineers and tech leads responsible for an agentic feature's observability surface. On-call engineers debugging incidents. Anyone whose answer to "why did the agent do that?" should be "open the trace, here." **Scope.** The *practice* of observing agents in production — what to instrument, how vendor tools (LangSmith / Braintrust / Phoenix / open-source LLM observability) fit, alert design, runbook integration, and the debugging workflow. Not the span-shape depth (see [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md)). Not the gateway-side cost telemetry (see [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The agent's behaviour is emergent. Each invocation produces a different trajectory: different tools called, different intermediate decisions, different outcomes from the same nominal input. Without observability, the team has no debugging surface — they read the conversation log and guess. With observability, the team has the trajectory as a navigable artefact: every turn, every tool call, every prompt, every model response, every cost line item, every failure handling decision, all linked in a hierarchy that an on-call engineer can read in minutes.

The observability investment is non-negotiable for production agents. The single most common reason a team's agent feature struggles is not the model, the prompts, or the tools — it is that the team cannot see what's happening, so problems are diagnosed by guessing and fixed by trial-and-error. Investing in observability is investing in the team's ability to make any other improvement quickly.

The vendor tools available in 2026 (LangSmith, Braintrust, Phoenix Arize, Honeycomb's LLM features, Datadog's LLM observability, open-source OpenLLMetry on top of OpenTelemetry) provide much of the heavy lifting. The discipline this document covers is what to instrument, what to alert on, how to integrate with the runbook, and how to debug effectively from the trace. The vendor choice is a tactical detail; the discipline is what matters.

This document is opinionated about four things:

1. **One trace per top-level request; hierarchy preserved.** Every span — loop turn, LLM call, tool call, retrieval, memory access — is rooted under the request's trace. The hierarchy is the debugging surface; a flat list of events is not.
2. **Trajectory data is captured, not just metrics.** Metrics tell you that the error rate went up; the trajectory tells you why. Both are needed; the trajectory is harder and more important.
3. **Alerts are loop-aware.** "Agent went into a loop" is a different alert from "LLM call failed." Generic error-rate alerts miss agent-specific pathologies; alerts must be designed for the agent shape.
4. **The runbook integrates with the trace.** When an alert fires, the on-call engineer's first action is to open a representative trace from the alert window. The link from alert to trace is engineered, not improvised.

Structure: (2) the four observability surfaces (trace, metrics, logs, evaluations); (3) trace shape recap; (4) the vendor tool integration; (5) loop-aware alerts; (6) runbook integration; (7) the debugging workflow; (8) sampling and retention; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The four observability surfaces

A complete agent observability practice has four surfaces; each serves a distinct purpose.

### 2.1 Trace (the trajectory)

Every agent invocation produces a trace: a tree of spans capturing every turn, tool call, LLM call, retrieval, memory access, error, and decision. The trace is the primary debugging surface. When something goes wrong, the trace shows what happened.

Where it lives: an observability backend (vendor-specific or OpenTelemetry-compatible). Searchable, navigable, persistent for a retention period.

### 2.2 Metrics (the aggregates)

Aggregated counters and histograms. Per-feature: request rate, latency p50/p99, cost, error rate. Per-tool: call rate, latency, error rate. Per-model: token throughput, cost.

Where they live: a time-series database (Prometheus, Datadog, observability-backend). Queryable in dashboards and alerts.

### 2.3 Logs (the events and side comments)

Free-text or structured logs that supplement the structured trace data. Often legacy or covering things the trace doesn't (e.g., cron-job events, infra-side incidents).

Where they live: a log aggregator (Loki, Datadog logs). Indexed for search.

### 2.4 Evaluations (the quality measurements)

Continuous evaluation against golden sets, production-trace samples, or LLM-as-judge scores. Not strictly observability but inseparable from it — quality is the slowest-changing observable.

Where it lives: an eval tool (LangSmith, Braintrust, in-house). Reports dashboards and historical trends.

### 2.5 The integration

The four are not separate practices; they reference each other:

- Traces produce metrics (sampling-aware aggregations).
- Alerts (metric-based) link to trace samples.
- Traces feed the eval set (production samples become eval cases).
- Logs and trace events share correlation IDs.

A team that has all four well-integrated can investigate any incident, diagnose it, fix it, and prevent it. A team that has only some of the four has gaps that show up at the wrong moment.

---

## 3. Trace shape recap

The span shape is described in depth in [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md). This section is the agent-engineering-side summary.

### 3.1 The hierarchy

```
trace_id: <request-trace>
├── request.handle (root span — entire user request)
│   ├── agent.invocation (the agent loop)
│   │   ├── agent.turn (1)
│   │   │   ├── prompt.build
│   │   │   ├── llm.call
│   │   │   │   ├── llm.call_attempt (1)
│   │   │   │   └── llm.call_attempt (2)  [retry]
│   │   │   ├── decision.parse
│   │   │   └── tool.dispatch
│   │   │       ├── tool.authorize
│   │   │       ├── tool.execute (fetch_patient)
│   │   │       │   ├── tool.implementation
│   │   │       │   └── retrieval.search  [if this tool retrieves]
│   │   │       └── tool.result
│   │   ├── agent.turn (2)
│   │   │   ├── ...
│   │   └── agent.terminate
│   └── response.format
└── side_effect.execute  [if the request involves side effects]
```

Every level is a span; every span has attributes; the parent-child relationship is preserved.

### 3.2 Per-span attributes (highlights)

Per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md):

- `tenant_id`, `feature`, `user_id`, `session_id`, `agent_invocation_id`.
- `model`, `prompt_version`, `tool_registry_version`.
- For `llm.call`: `input_tokens`, `output_tokens`, `cost_usd`, `latency_ms`.
- For `tool.execute`: `tool_name`, `arguments`, `result_status`, `cost_usd` (if applicable), `failure.category`, `idempotency_key`.
- For `agent.turn`: turn number, current state size, structured-state snapshot.
- For `agent.terminate`: termination reason (final_answer / budget_breach / escalation / error).

The attributes are the queryable surface; the engineer searches by attribute (e.g., `agent_invocation_id`, `failure.category=junk`) to find relevant traces.

### 3.3 The decision attributes

Each `agent.turn` span captures the model's *decision* — what it chose to do this turn. The attributes:

- `decision.kind` (tool_call / final_answer / escalate).
- `decision.tool_name` (if tool call).
- `decision.reasoning_summary` (if the model produced reasoning text; truncated to bounded size).

The decision attributes are how the trace tells the story of the agent's behaviour. An engineer reading the trace sees "turn 1 → tool: search_patients; turn 2 → tool: fetch_patient; turn 3 → tool: fetch_recent_labs; turn 4 → final answer" at a glance.

### 3.4 Sensitive data handling

The trace contains PII (patient IDs, names) and sensitive content (clinical notes, conversation content). Discipline:

- **Redaction at capture.** Tools that return sensitive content have a redaction layer; the trace captures a structured summary, not the full payload.
- **Field-level controls.** Specific fields are marked sensitive in the trace schema; the observability backend honours field-level access controls.
- **Per-environment retention.** Production traces have shorter retention than dev/test traces.
- **Audit on access.** Engineers accessing production traces with sensitive content are logged.

The redaction strategy is decided per feature and reviewed quarterly.

---

## 4. Vendor tool integration

In 2026, several vendor tools provide agent observability with varying integration patterns.

### 4.1 The vendor landscape

| Tool | Strength | Typical fit |
| --- | --- | --- |
| LangSmith | Native LangChain / LangGraph integration; agent-trajectory views | Teams using LangChain ecosystem |
| Braintrust | Eval-first; strong eval / trace integration | Teams whose eval discipline is central |
| Phoenix Arize | Open-source; OpenTelemetry-native | Teams investing in OTEL-based stack |
| Honeycomb | General observability + LLM features | Teams using Honeycomb broadly |
| Datadog LLM Observability | Integrates with Datadog ecosystem | Teams on Datadog already |
| OpenLLMetry (open-source) | OTEL-based instrumentation library | Build-your-own stack on OTEL |

The choice is tactical, not strategic. The discipline (what to instrument, what to alert on) is portable; the tool implements it.

### 4.2 The integration pattern

A clean integration:

1. **Trace ID propagation.** A trace ID is created at the request boundary; propagated through every component (runner, gateway, tool implementations, downstream services).
2. **Span emission.** Each instrumented component emits spans with the correct parent reference; the vendor's SDK or OpenTelemetry handles the wire format.
3. **Attribute discipline.** Standard attribute names per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md); cross-tool consistency.
4. **Backend ingestion.** The observability backend ingests, indexes, and exposes via UI / API.

The pattern is the same regardless of vendor.

### 4.3 Auto-instrumentation vs explicit instrumentation

Vendor SDKs offer auto-instrumentation for LLM providers (OpenAI, Anthropic) and popular frameworks (LangChain, LlamaIndex). Auto-instrumentation gets you started; explicit instrumentation is needed for the agent-specific surfaces (the runner's loop, tool dispatch, structured state).

Recommendation: combine. Use auto-instrumentation for the provider SDK; explicit instrumentation for the agent's own components. Avoid double-instrumentation (which produces duplicate spans).

### 4.4 The "vendor lock-in" mitigation

Vendor tools differ in their ingestion format and query interface. To avoid lock-in:

- **Instrument with OpenTelemetry primitives.** Span creation, attributes, events — all OTEL-standard.
- **Ship to the vendor via OTEL collector.** The collector can route to multiple backends; switching vendors changes the collector config, not the instrumentation.
- **Document the attribute schema.** The schema is portable across vendors; the queries adapt.

Vendor switches are possible but expensive. The mitigation reduces the cost, not the frequency.

### 4.5 The vendor-tool's eval integration

Several vendor tools (LangSmith, Braintrust) integrate eval with observability: production traces can be promoted to eval cases; eval runs link to traces. The integration is valuable for the "regression suite from production traces" pattern (per [agent-evals.md](./agent-evals.md)).

Use the integration where the vendor supports it; build a thin wrapper to capture the pattern where they don't.

### 4.6 Cost of vendor tools

Observability has cost — vendor licenses, storage, ingestion. Budget for it; it's not optional infrastructure. A reasonable benchmark: 2–5% of the AI feature's cost (excluding the vendor tool itself's pricing model nuances).

If observability cost is a concern, the levers are sampling (section 8) and shorter retention. Don't reduce attribute coverage; the attributes are the value.

---

## 5. Loop-aware alerts

Generic observability alerts miss agent-specific pathologies. The alerts here are designed for the agent shape.

### 5.1 Loop-related alerts

- **High turn count.** Agent invocations exceeding N turns (e.g., > 20) at > 1% rate. Indicates loops or excessive multi-turn cases.
- **Turn count outlier.** P99 turn count exceeded threshold; pattern detection on which feature, which tenant.
- **Budget breach rate.** Per-request budget breaches > 0.5% over 15 minutes. Indicates either too-tight budgets or new pathologies.
- **Specific failure category spike.** `failure.category=partial` rate > baseline; `failure.category=junk` rate > baseline; etc.
- **Tool error rate spike.** A specific tool's error rate jumps. Often indicates an upstream change.
- **Tool selection accuracy drift.** Per-tool selection accuracy (measured against eval) drifts down. Indicates a prompt or model regression.

### 5.2 Cost-related alerts

Covered in [agent-cost-control.md](./agent-cost-control.md). Summary:

- Per-feature cost spike.
- Per-tenant cost spike.
- Per-tier cost mix drift (frontier-model usage creeping up).

### 5.3 Quality-related alerts

- **Eval gate failures.** Pre-prod eval failed; promotion blocked.
- **Production-trace eval degradation.** A continuous LLM-judge eval over production samples shows quality dropping.
- **Escalation rate spike.** Agent escalating more than baseline (per [error-and-partial-failure.md](./error-and-partial-failure.md)).

### 5.4 Latency-related alerts

- **Per-feature latency spike.** End-to-end latency exceeds threshold.
- **Per-component latency spike.** A specific component (LLM call, tool dispatch, retrieval) is the contributor.
- **Tail latency degradation.** P99 latency creeps up; investigation before user-visible.

### 5.5 Multi-agent / coordination alerts

For multi-agent systems (per [multi-agent-coordination.md](./multi-agent-coordination.md)):

- Hand-off matrix anomaly (new hand-offs that shouldn't happen).
- Coordinator failure rate.
- Worker contribution ratio drift (one worker dominating cost or invocation).

### 5.6 Alert design discipline

Each alert has:

- **Clear name and description.** "Care-coordinator turn-count P99 > 25 over 10m" — the name is the answer to "what fired?"
- **Severity.** Critical (page now) / High (page in hours) / Medium (review next day) / Low (FYI in weekly report).
- **Runbook link.** The on-call response steps for this specific alert.
- **Sample-trace link.** A pre-built query that returns traces matching the alert condition.
- **Recent-history view.** Has this alert fired recently? What was done?

The discipline keeps alerts actionable. Alerts that are noisy, vague, or without runbooks erode on-call attention and lead to missed real incidents.

### 5.7 Alert tuning

Alerts that fire frequently and are usually ignored should be retuned or removed. Alerts that should fire but don't (incidents discovered without an alert firing) indicate gaps. Quarterly review of alert quality is the discipline.

---

## 6. Runbook integration

When an alert fires, the runbook is the script. Engineered, not improvised.

### 6.1 Runbook structure per agent feature

```
== <Feature> Runbook ==
Section 1: General triage (start here for unknown alerts)
Section 2: <Alert 1 name>
Section 3: <Alert 2 name>
...
Section N: <Alert N name>
Section X: Recent incidents (last 12 months)
```

Each alert has its own section. The general-triage section covers unknown alerts.

### 6.2 Per-alert section content

For each alert:

- **Symptom.** What the alert detected; what the user-visible impact (if any) is.
- **Dashboard link.** The dashboard to open first.
- **Triage queries.** Specific queries (trace search, log search, metric query) that diagnose common causes.
- **Common causes.** A list of patterns observed before; what each looks like in the traces.
- **Mitigation options.** Specific actions on-call can take (rollback, tighten cap, disable user, etc.) — and the authority required for each.
- **Escalation criteria.** When to escalate (and to whom).
- **Post-incident actions.** What to update after the incident (eval set, runbook itself, prompts, alerts).

### 6.3 Trace-first triage

The runbook's first action for almost any alert is "open a representative trace." Pre-built queries pull recent traces matching the alert condition. The on-call engineer reads the trace, identifies the pattern, then applies the appropriate mitigation.

This is the most operationally valuable integration: the alert tells you *something is wrong*; the trace tells you *what*. Without the link from alert to trace, on-call is reduced to guessing.

### 6.4 Runbook update discipline

Each significant incident produces a runbook update:

- New patterns observed → added to "common causes."
- New mitigation options discovered → added to "mitigation."
- New alerts needed → engineering ticket + alert added later → runbook section added.

The runbook is a living document. Stale runbooks are worse than no runbook (false sense of safety).

### 6.5 Pre-incident drills

Periodic drills:

- Synthetic incident triggered (e.g., a fake spike alert).
- On-call engineer walks through the runbook.
- Time-to-diagnosis and time-to-mitigation measured.
- Drill outcome feeds runbook updates.

Drills surface stale runbooks, broken links, and unclear instructions. Monthly cadence in mature operations.

### 6.6 The on-call team's authority

The runbook documents what on-call can do without escalation. Common authorities:

- Rollback a recent deploy.
- Tighten a per-tenant cap.
- Disable a user.
- Toggle a feature flag.

Mitigation requiring approval (e.g., extending a cap, contacting a tenant) is also documented with the approval path.

Authority must be commensurate with the incident response time targets. If a sub-15-minute MTTM is required and on-call needs 30 minutes to get a rollback approved, the authority is wrong.

---

## 7. The debugging workflow

The on-call engineer's debugging workflow, with observability as the substrate.

### 7.1 Step 1 — acknowledge and orient

Acknowledge the page. Open the alert; read the runbook section.

### 7.2 Step 2 — open the dashboard

The runbook's dashboard link. Confirm the alert is real (not a false positive). Identify the affected scope (feature, tenant, user).

### 7.3 Step 3 — open a representative trace

The runbook's trace query. Read 2–3 sample traces. Identify the pattern:

- What was the agent doing?
- At what turn did the problem appear?
- What tool was involved?
- What was the failure category?
- What did the model decide vs what would have been right?

The trace narrative answers most "what's happening" questions in minutes.

### 7.4 Step 4 — apply mitigation

The runbook's mitigation options. Apply the appropriate one. Verify the alert clears or the metric returns to baseline.

### 7.5 Step 5 — communicate

Per the runbook's communication template. Internal team, affected tenants, finance team — as appropriate.

### 7.6 Step 6 — investigate root cause

After mitigation, deeper investigation:

- Was this a deploy? (Git log + deploy log + diff.)
- Was this an upstream change? (Tool implementations or external dependencies.)
- Was this a misuse pattern? (User behaviour analysis.)
- Was this a model issue? (Model version + recent provider notes.)
- Was this a prompt issue? (Prompt versions + recent changes.)

Root cause leads to the fix.

### 7.7 Step 7 — post-incident review

Standard PIR. Specific to agents:

- What did the trace reveal? Was the trace sufficient?
- Was the observability surface adequate, or were there blind spots?
- Did the alert fire fast enough?
- Did the runbook help?
- What to update?

The PIR's outputs feed code, prompts, tools, eval, observability, runbook.

### 7.8 The "I can't see the trace" failure mode

When the trace is incomplete or missing — sampling dropped it, the observability backend dropped it, the agent ran in a context that wasn't instrumented — debugging is materially harder. The fallback:

- Logs with correlation IDs (aggregate by correlation_id).
- Database state at relevant points (queries to reconstruct).
- The model provider's logs (most providers log calls; available via dashboards or API).

These are slower. The discipline is to keep trace coverage high so this fallback is rarely needed.

---

## 8. Sampling and retention

Trace data is voluminous; sampling and retention strategies are necessary.

### 8.1 Why sample

Capturing every span of every request is often infeasible at scale (storage cost, ingestion cost, query cost). Sampling reduces volume; the discipline ensures useful data is preserved.

### 8.2 Sampling strategies

**Head-based sampling.** Decide at the request boundary whether to trace this request (e.g., 10% of requests fully traced). Simple, predictable cost. Loses unusual cases that fall outside the sample.

**Tail-based sampling.** Capture all spans for every request; at request end, decide whether to keep based on properties (error? long latency? high cost?). Keeps the interesting cases; discards the routine. More expensive to implement; more useful for incident investigation.

**Adaptive sampling.** Sample rate varies by feature, tenant, or attribute. High-priority features (where incidents are expensive) get higher rates.

Recommendation: tail-based for agent features where possible. The agent's unusual cases (loops, breaches, escalations) are the ones worth investigating; they should always be captured.

### 8.3 Per-feature sampling configuration

- Critical features (high-stakes, high-cost): tail-based at 100% (always capture).
- Mainstream features: tail-based or 50% head-based.
- Low-priority / internal-only: 10% head-based.

The configuration is reviewed quarterly. Features moving in importance get updated rates.

### 8.4 Retention

- **Production traces with sensitive content:** 7–30 days (per privacy policy).
- **Production traces without sensitive content:** 30–90 days.
- **Dev / test traces:** 14 days.
- **Aggregated metrics:** 13 months (year-over-year comparisons).

Retention is a privacy decision, not an engineering convenience. Coordinate with legal / compliance.

### 8.5 The cost of observability

A representative breakdown for a moderate-scale AI feature:

- Trace ingestion: $200–$2000 / month (varies with vendor and volume).
- Trace storage: $50–$500 / month.
- Vendor license: $0–$5000 / month.
- Metrics: $50–$500 / month.

For a feature spending $5k–$50k / month on LLM cost, the observability overhead is 2–10% of LLM cost. Worth it for the operational visibility.

### 8.6 The "we'll trace everything later" anti-pattern

Some teams defer observability investment until incidents force it. The cost: the first major incident is the most expensive (no trace data to investigate; root cause is guessed; recurrence is likely). Front-loading observability is operationally cheaper.

---

## 9. Worked Meridian example

Meridian's care-coordinator observability practice.

### 9.1 Stack

- **Tracing.** Phoenix Arize (open-source) on top of OpenTelemetry instrumentation in the runner, gateway, and tool implementations.
- **Metrics.** Prometheus + Grafana for dashboards; Alertmanager for alert routing.
- **Logs.** Loki for structured log aggregation.
- **Eval.** Braintrust for eval runs and production-trace-to-eval-case promotion.

The choice reflects a preference for open-source where viable; Braintrust was paid because the eval integration was strongest.

### 9.2 The trace shape

Per section 3.1. Each request: 1 root span, 1 agent.invocation span, N agent.turn spans, M tool spans inside each turn, K llm.call spans. Average request: ~12 spans for a normal 3-turn invocation; ~40 spans for a complex 10-turn invocation.

Trace data per request: ~50 KB. At ~100k requests / day, ~5 GB / day, ~150 GB / month. Sampled to ~30 GB / month at 100% tail-based for the critical features and 10% head-based for non-critical.

### 9.3 Dashboards

The on-call dashboard for care-coordinator:

- **Top panel:** request rate (per minute), p50/p95/p99 latency, error rate.
- **Cost panel:** cost per minute, top 10 tenants, per-tier breakdown.
- **Loop health panel:** average turn count, P99 turn count, budget breach rate.
- **Tool health panel:** per-tool call rate, per-tool error rate.
- **Failure category panel:** rate of each failure category (transient / permanent / junk / partial / etc.).
- **Eval signal panel:** quality score on continuous production-trace eval (LLM-as-judge over 1% sample).

The dashboard is the first link in every alert's runbook section.

### 9.4 Alerts (selected)

- **Cost spike (per-feature):** care-coordinator cost-per-minute > 2× rolling 7-day avg for 5 minutes. Severity: critical, pager. Runbook: tighten per-feature cap; identify contributing tenant; trace samples.
- **Turn count outlier:** P99 turn count > 20 over 10 minutes (vs baseline 8). Severity: high. Runbook: identify pattern; trace samples; recent prompt / tool changes.
- **Tool error rate spike:** any tool's error rate > 3× baseline. Severity: high. Runbook: identify upstream change; check tool's recent deploys; coordinate with upstream owners.
- **Budget breach rate:** per-request budget breaches > 0.5% over 15 minutes. Severity: high. Runbook: pattern analysis; recent changes; budget calibration review.
- **Eval signal degradation:** continuous LLM-judge score drops > 5% over 24 hours. Severity: medium. Runbook: eval analysis; recent prompt / model changes; possible rollback.
- **Escalation rate spike:** agent escalation rate > 2× baseline. Severity: medium. Runbook: trace samples; identify what's making the agent give up.

The alert family has 14 alerts total; each has a runbook section.

### 9.5 Runbook example — cost spike

```
== Cost Spike (per-feature) ==
Symptom: Care-coordinator cost-per-minute alert firing; > 2x rolling 7-day avg.
User-visible impact: None directly; cost may exceed budget.

Dashboard: <link to care-coordinator cost dashboard>
Triage queries:
- Trace search: care-coordinator requests in last 15min with cost > $0.40
- Per-tenant: cost rate by tenant in last 30min
- Per-tier: cost contribution by tier in last 30min

Common causes:
1. Prompt change at deploy: check deploy log for last 2 hours
2. Tenant misconfiguration: a single tenant with concentrated traffic
3. Tool regression: a tool returning more data than expected
4. Model upgrade: check model-version pin and recent provider notes

Mitigation options:
- Rollback recent deploy [authority: on-call]
- Tighten per-tenant cap [authority: on-call]
- Tighten per-feature cap [authority: on-call manager]
- Disable feature [authority: on-call manager + product]

Escalation: 15min without root cause → on-call manager
            30min without mitigation → engineering leadership

Post-incident actions:
- Update eval-gate's cost-delta check if a prompt-change pattern
- Update tier-routing config if tier mix is the cause
- Add specific alert if a new pattern emerges
```

### 9.6 Incidents over the last 12 months

Five significant incidents; for each, the time-to-diagnosis was 5–10 minutes (the trace surfaced the cause quickly). Time-to-mitigation 12–22 minutes. All five produced runbook updates and one (Q3-25) produced a new alert for the specific cost-regression-from-prompt-change pattern.

### 9.7 What worked

- **Trace coverage of 100% for care-coordinator.** Every request is investigable.
- **Per-tool span detail.** Tool issues are diagnosed quickly because the trace shows tool arguments, results, and errors per call.
- **Runbook integration.** On-call always knows what to do first; no improvising on critical alerts.
- **Vendor integration with eval.** Braintrust's link from trace to eval case makes production-trace promotion fast.

### 9.8 What didn't work initially

- **Auto-instrumentation duplication.** The first deployment auto-instrumented at multiple layers; spans were duplicated. Cleaned up to one source of truth per span type.
- **Over-noisy alerts.** Original alert set had ~30 alerts; about 8 fired routinely with no real signal. Tuned down to 14, the current set.
- **Privacy review surprises.** Early traces captured full clinical content; legal review required redaction. Implemented redaction at the tool boundary.

Each lesson fed a refinement; the current setup is the result of two years of iteration.

---

## 10. Anti-patterns

### 10.1 "Logging only; no trace"

Free-text logs across components; no structured trace. Debugging requires grep-and-correlate.

**Corrective.** Structured trace with consistent attributes. Logs supplement but don't replace.

### 10.2 "Trace exists but no hierarchy"

Spans are flat lists; parent-child relationships are missing. Reading is difficult; the narrative is lost.

**Corrective.** Span hierarchy per section 3.1. Parent references on every span.

### 10.3 "Generic alerts only"

The alert set is "error rate > X%" with no agent-specific signals. Agent pathologies (loops, escalation spikes, junk-output cascades) go undetected.

**Corrective.** Loop-aware alerts per section 5.

### 10.4 "Alerts without runbooks"

Alerts fire; on-call has no script; response is improvised; MTTR is long.

**Corrective.** Every alert has a runbook section per section 6.2.

### 10.5 "Runbook never updated"

The runbook was written once; the team has learned but the runbook is stale.

**Corrective.** Every significant incident updates the runbook per section 6.4.

### 10.6 "Sensitive data captured raw"

Patient PHI, conversation content, customer data — captured raw in the trace. Privacy incident waiting.

**Corrective.** Redaction at capture per section 3.4.

### 10.7 "Sampling drops the interesting cases"

Random head-based sampling means errors and outliers are mostly dropped.

**Corrective.** Tail-based sampling per section 8.2 for incident-relevant features.

### 10.8 "Vendor lock-in"

Instrumentation is bespoke to the vendor; switching costs are prohibitive.

**Corrective.** OpenTelemetry primitives per section 4.4. Collector-based routing.

---

## 11. Findings (sprint-assignable)

### AGT-OBS-001 — Severity: Critical
**Finding.** Agent has no trace coverage; incidents are debugged from logs and guesswork.
**Recommendation.** Trace instrumentation per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md); minimum coverage runner + LLM call + tool dispatch.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-OBS-002 — Severity: Critical
**Finding.** Alerts without runbooks; on-call response improvised; MTTM > 60 minutes.
**Recommendation.** Per-alert runbook sections per section 6.2; trace-link integration per section 6.3.
**Owner.** ops + ai-platform-eng, sprint N+1.

### AGT-OBS-003 — Severity: Critical
**Finding.** Sensitive content (PHI, customer data) captured raw in traces.
**Recommendation.** Redaction at capture per section 3.4; privacy review.
**Owner.** ai-platform-eng + privacy, sprint N+1.

### AGT-OBS-004 — Severity: High
**Finding.** Generic error-rate alerts only; loop pathologies (turn count spikes, escalation spikes, partial-failure spikes) undetected.
**Recommendation.** Loop-aware alert set per section 5.
**Owner.** ai-platform-eng + ops, sprint N+2.

### AGT-OBS-005 — Severity: High
**Finding.** Span hierarchy is flat or missing parent references; trace reads as unrelated events.
**Recommendation.** Hierarchy per section 3.1; parent reference discipline.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-OBS-006 — Severity: High
**Finding.** Random head-based sampling drops interesting cases; incident investigation is gappy.
**Recommendation.** Tail-based sampling per section 8.2 for critical features.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-OBS-007 — Severity: High
**Finding.** Runbook is stale; recent incident patterns not reflected.
**Recommendation.** Runbook update cadence per section 6.4; PIR-driven.
**Owner.** ops, sprint N+2.

### AGT-OBS-008 — Severity: High
**Finding.** On-call lacks authority to mitigate (e.g., rollback) without escalation; MTTM is dominated by approval time.
**Recommendation.** Documented authority per section 6.6; commensurate with response targets.
**Owner.** leadership + ops, sprint N+2.

### AGT-OBS-009 — Severity: Medium
**Finding.** Production-trace samples are not promoted to eval; eval set is stale relative to production.
**Recommendation.** Trace-to-eval pipeline per section 2.5 and [agent-evals.md](./agent-evals.md).
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-OBS-010 — Severity: Medium
**Finding.** Tools' decision attributes (`decision.kind`, `decision.tool_name`, `decision.reasoning_summary`) not captured; trace doesn't tell the story.
**Recommendation.** Decision attributes per section 3.3.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-OBS-011 — Severity: Medium
**Finding.** Per-feature dashboard missing or incomplete; on-call lacks the first-level view.
**Recommendation.** Standard dashboard panels per section 9.3.
**Owner.** ai-platform-eng + ops, sprint N+3.

### AGT-OBS-012 — Severity: Medium
**Finding.** Alert tuning never performed; many alerts fire routinely and are ignored.
**Recommendation.** Quarterly alert quality review per section 5.7.
**Owner.** ops, sprint N+3.

### AGT-OBS-013 — Severity: Medium
**Finding.** Multi-agent system observed only per-agent; no aggregate / hierarchical view.
**Recommendation.** Shared trace_id across agents per [multi-agent-coordination.md](./multi-agent-coordination.md) section 8.1.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-OBS-014 — Severity: Medium
**Finding.** Vendor instrumentation is bespoke; switching costs prohibitive.
**Recommendation.** OpenTelemetry primitives per section 4.4; collector-based routing.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-OBS-015 — Severity: Low
**Finding.** Auto-instrumentation produces duplicate spans; trace is noisier than necessary.
**Recommendation.** Resolve duplicates per section 4.3; one source of truth per span type.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-OBS-016 — Severity: Low
**Finding.** No pre-incident drills; runbooks aren't exercised; staleness undetected until real incident.
**Recommendation.** Monthly drills per section 6.5.
**Owner.** ops, sprint N+4.

### AGT-OBS-017 — Severity: Low
**Finding.** Trace retention not aligned with privacy policy; risk of over-retention.
**Recommendation.** Retention policy per section 8.4; legal review.
**Owner.** ai-platform-eng + privacy, sprint N+5.

### AGT-OBS-018 — Severity: Low
**Finding.** Observability cost not tracked; observability spend grows unchecked.
**Recommendation.** Cost-tracking per section 8.5; quarterly review.
**Owner.** ai-platform-eng + finance, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team launching a new agent:

- [ ] **Sprint 0 — instrumentation plan.** What spans, what attributes, what redaction.
- [ ] **Sprint 0 — vendor choice.** Pick the observability stack; OpenTelemetry-compatible preferred.
- [ ] **Sprint 1 — runner instrumentation.** Loop, turns, decisions, terminations.
- [ ] **Sprint 1 — LLM-call instrumentation.** Per the wrapper.
- [ ] **Sprint 1 — tool-dispatch instrumentation.** Per the registry.
- [ ] **Sprint 1 — redaction.** Sensitive fields at capture.
- [ ] **Sprint 2 — dashboards.** Per-feature view; per-tenant view; loop-health view.
- [ ] **Sprint 2 — alerts.** Loop-aware; runbook-linked.
- [ ] **Sprint 2 — runbook.** Per-alert sections.
- [ ] **Sprint 3 — sampling.** Tail-based for critical; appropriate for others.
- [ ] **Sprint 3 — retention.** Aligned with privacy policy.
- [ ] **Sprint 3 — eval integration.** Trace-to-eval promotion pipeline.
- [ ] **Sprint 4 — drills.** Initial drill; cadence established.
- [ ] **Ongoing — quarterly review.** Alerts, dashboards, runbook, sampling, cost.

For a team retrofitting observability on an existing agent:

- [ ] **Sprint 0 — gap analysis.** What's instrumented; what's missing; what runbook exists.
- [ ] **Sprint 1 — close the worst gap.** Often trace coverage or sensitive-data handling.
- [ ] **Sprint 2 — alerts and runbook.** Get the response surface in place.
- [ ] **Sprint 3 — tail-based sampling.** Preserve the interesting cases.
- [ ] **Sprint 4 — eval integration.** Close the loop with quality measurement.

A team that completes the sequence has incidents diagnosable in minutes and a continuously improving operational practice. A team that doesn't has incidents that compound and a team that resists working on agents because each shift on-call is unpredictable.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 9 (observability).
- [agent-loop-design.md](./agent-loop-design.md) — runner that emits the trace.
- [tool-architecture.md](./tool-architecture.md) — tool registry and dispatch that emit per-tool spans.
- [memory-engineering.md](./memory-engineering.md) — memory state observable in traces.
- [error-and-partial-failure.md](./error-and-partial-failure.md) — failure metadata in spans.
- [agent-cost-control.md](./agent-cost-control.md) — cost observability that feeds dashboards and alerts.
- [agent-evals.md](./agent-evals.md) — eval integration with traces.
- [multi-agent-coordination.md](./multi-agent-coordination.md) — observability for multi-agent (hierarchical trace).
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — span shape depth.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — LLM call span.
- [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — retrieval tool span.
- [observability-and-telemetry/trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) — overall trace design.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alert engineering patterns.
- [observability-and-telemetry/debugging-from-traces.md](../observability-and-telemetry/debugging-from-traces.md) — debugging workflow depth.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — continuous quality eval over traces.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — dashboard patterns.
- [observability-and-telemetry/vendor-tool-integration.md](../observability-and-telemetry/vendor-tool-integration.md) — vendor integration depth.
- LangSmith, Braintrust, Phoenix Arize, Honeycomb, Datadog LLM Observability — vendor tools referenced in section 4.1.
- OpenTelemetry (spec + SDKs) — the foundation for vendor-neutral instrumentation.
- OpenLLMetry (Traceloop) — open-source LLM-specific OTEL instrumentation library.
