# Vendor Tool Integration

> **Audience.** Engineers and tech leads choosing which AI observability vendor(s) to use, or refactoring to avoid lock-in. **Scope.** The *engineering* practice of integrating AI observability vendors — LangSmith, Braintrust, Phoenix, Helicone, Datadog, etc. Build-vs-buy decisions, multi-route export, vendor-portability discipline. Pair with [trace-and-span-design.md](./trace-and-span-design.md), [llm-call-instrumentation.md](./llm-call-instrumentation.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The AI observability vendor space in 2026 is active. LangSmith, Braintrust, Phoenix, Helicone, Portkey, Datadog LLM Observability — each has its own emphasis (eval / cost / general-APM-with-LLM-overlays). Most teams pick one vendor early; some end up locked in; some adopt multiple for different purposes without coordination; some build everything themselves at substantial engineering cost.

The discipline this document codifies: the team owns the instrumentation; vendors consume it. The chokepoint pattern from [llm-call-instrumentation.md](./llm-call-instrumentation.md) and [trace-and-span-design.md](./trace-and-span-design.md) means traces are emitted in a standard form (OpenTelemetry-based); the OpenTelemetry collector routes to chosen vendors. Vendor swap is configuration; the application doesn't know which vendor is downstream.

This document is opinionated about three things:

1. **Instrument once, export to many.** OpenTelemetry-based emission; multiple downstream consumers; vendor lock-in avoidance.
2. **Pick vendors per value-add.** Eval tools, cost observability, general APM each have specialized vendors. Most teams use 2-3 for different purposes.
3. **The vendor's value-add is not the chokepoint.** The team's wrapper / gateway is the chokepoint; vendors are consumers of that data.

Structure: (2) the vendor landscape; (3) build-vs-buy decisions; (4) the OpenTelemetry collector pattern; (5) per-vendor integration patterns; (6) avoiding vendor lock-in; (7) cost considerations; (8) vendor evaluation workflow; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The vendor landscape

The categories of AI observability vendors.

### 2.1 AI-eval-focused vendors

**Examples.** LangSmith, Braintrust, Phoenix, Vellum.

**Strengths.**
- Strong eval tooling (golden-set management, judge calibration, regression suites).
- Built for AI workflows (trace-as-debuggable-LLM-call-with-prompt).
- Rich integration with common AI frameworks (LangChain, LlamaIndex).

**When right.** Teams investing heavily in eval engineering (per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md)) — the vendor's eval tools accelerate the practice.

### 2.2 Cost / token observability vendors

**Examples.** Helicone, Portkey.

**Strengths.**
- Per-call cost attribution.
- Provider routing and load balancing.
- Cost dashboards out of the box.

**When right.** Teams primarily concerned with cost observability and operational routing.

### 2.3 General APM with AI extensions

**Examples.** Datadog LLM Observability, New Relic AI Monitoring, Honeycomb AI integrations.

**Strengths.**
- Unified observability with the rest of the platform.
- Familiar operational patterns (alerts, dashboards, runbooks).
- Cross-system correlation (AI feature alongside database, web tier).

**When right.** Teams that already have strong APM and want AI to fit alongside.

### 2.4 Open-source / self-hosted

**Examples.** Phoenix (Arize), Langfuse (open-source tier), OpenInference.

**Strengths.**
- No vendor lock-in (the team owns the deployment).
- Customizable.
- Cost-effective at scale (no per-call vendor fees).

**When right.** Teams with strong infrastructure operations; data-residency requirements that limit hosted options; cost-conscious at high volume.

### 2.5 Built-in-house

**Examples.** Custom dashboards on existing storage (Datadog with custom panels; Grafana on Prometheus).

**Strengths.**
- Fully customized to the team's workload.
- No vendor dependency.

**When right.** Teams with very specific observability needs not met by vendors; teams willing to invest in dashboard engineering.

### 2.6 The category overlap

Many vendors offer features across categories:

- LangSmith has cost tracking alongside eval.
- Datadog has eval-like features in LLM Observability.
- Helicone has some eval tooling.

The categories are emphases, not exclusive offerings. The team picks based on the vendor's primary value-add.

---

## 3. Build-vs-buy decisions

What to build internally vs buy from vendors.

### 3.1 The chokepoint must be built

The instrumentation chokepoint (per [llm-call-instrumentation.md](./llm-call-instrumentation.md), [trace-and-span-design.md](./trace-and-span-design.md)) is built. The team owns:

- The LLM-call wrapper.
- The trace span emission.
- The attribute taxonomy.
- The OpenTelemetry collector / export pipeline.

Without the built chokepoint, vendor lock-in is inevitable; the team is dependent on the vendor's instrumentation patterns.

### 3.2 The eval engineering is owned

Per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md):

- The eval cases and golden sets are owned by the team.
- The judge prompt is owned.
- The eval execution can be done by the vendor or built; the engineering discipline is independent of tooling.

Some vendors (LangSmith, Braintrust) provide eval execution; the team can leverage that.

### 3.3 The cost telemetry is computed locally

Per [llm-call-instrumentation.md](./llm-call-instrumentation.md) section 4:

- Cost computed at call time in the wrapper.
- Cost emitted as span attribute.

Vendors that compute cost from their own pricing tables can be inconsistent with the wrapper's computation; the wrapper's computation is the source of truth.

### 3.4 The dashboards: built or bought

Dashboards can be:

- Vendor's UI (LangSmith dashboards, Datadog dashboards, etc.).
- Custom Grafana / Tableau / Looker.

The decision is per-dashboard:

- Operational dashboards (real-time cost, alerts): often vendor-native is fine.
- Diagnostic dashboards (drill-down patterns): often need customization.
- FinOps dashboards: often need custom layouts.

### 3.5 The decision framework

| Component | Built | Bought |
|---|---|---|
| LLM-call wrapper | Always | Never |
| Trace emission (OpenTelemetry) | Always | Never |
| Eval cases / golden sets | Always | Never |
| Judge prompts | Always | Never |
| Cost computation | Always | Never |
| Trace storage | Usually buy | Custom for very specific needs |
| Eval execution | Either | Vendors accelerate; team can build |
| Dashboards | Mixed | Mixed; per-dashboard decision |
| Alerting | Either | Vendors integrate; team can customize |
| Cost dashboards | Custom or vendor | LLM-cost vendors are good |

The decision is per-component; not whole-stack.

---

## 4. The OpenTelemetry collector pattern

The architecture that supports multiple vendors.

### 4.1 The pattern

```
Application (with instrumentation per llm-call-instrumentation.md)
    │
    ▼ OpenTelemetry trace export
OpenTelemetry Collector
    │
    ├─→ Vendor A (e.g., LangSmith) — for AI eval workflows
    ├─→ Vendor B (e.g., Datadog) — for unified APM
    ├─→ Vendor C (e.g., Helicone) — for cost observability
    └─→ Long-term cold storage (e.g., S3 with KMS) — for compliance
```

The application emits once; the collector routes to multiple destinations.

### 4.2 The collector configuration

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

exporters:
  langsmith:
    api_key: ${LANGSMITH_API_KEY}
    endpoint: https://api.smith.langchain.com
  datadog:
    api_key: ${DATADOG_API_KEY}
  helicone:
    api_key: ${HELICONE_API_KEY}
  s3:
    bucket: meridian-trace-archive
    encryption: kms

processors:
  batch:
  attributes/redact:
    actions:
      - key: query_text
        action: hash  # PHI redaction
  sampling:
    sampling_percentage: 10
    # tail-based augmentation for failures, slow tails, etc.

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [attributes/redact, batch]
      exporters: [langsmith, datadog, helicone, s3]
```

Configuration-driven; vendor swaps are configuration changes.

### 4.3 The processor pipeline

The collector supports processing pipelines:

- **Attribute redaction.** PHI / PII fields hashed or stripped before export.
- **Sampling.** Per [trace-and-span-design.md](./trace-and-span-design.md) section 5.
- **Tail-based augmentation.** Keep traces matching specific criteria.
- **Batching.** Reduce export overhead.

Processing happens once; all exporters see processed traces.

### 4.4 The per-vendor format adaptation

Different vendors expect different trace formats:

- LangSmith: their proprietary format with specific span attributes.
- Datadog: APM trace format.
- Helicone: token-call format.

The OpenTelemetry collector has per-vendor exporters that handle the format conversion. The application doesn't need to know.

### 4.5 The independent export

Each exporter operates independently:

- One vendor outage doesn't affect others.
- One vendor's rate limit doesn't block another's export.
- Buffering per exporter for resilience.

The collector's design supports this.

### 4.6 The cost of multi-export

Each export consumes:
- Network bandwidth (per-span data sent to each vendor).
- Vendor cost (per-vendor pricing model).

For 10K traces/day × 3 vendors: ~30K trace ingest events. Most vendors handle this volume in their entry-level tiers.

---

## 5. Per-vendor integration patterns

How each vendor category integrates.

### 5.1 The AI-eval vendor integration

For LangSmith / Braintrust / Phoenix:

- Traces flow via OpenTelemetry (with vendor-specific attribute extensions where needed).
- The team uses the vendor's eval UI for case management and result visualization.
- The team's eval scripts can be deployed in the vendor's runtime (or run locally; vendor consumes results).

The vendor accelerates the eval engineering practice without owning it.

### 5.2 The cost vendor integration

For Helicone / Portkey:

- The vendor sits in the request path (gateway role) or consumes traces.
- Per-call cost is captured by the vendor.
- The vendor's dashboard shows cost in their UI.

For pure-observability integration (not gateway): traces with cost attributes flow to the vendor's collector.

For gateway integration: the wrapper routes through the vendor; the vendor handles routing + cost capture.

### 5.3 The general APM integration

For Datadog / New Relic / Honeycomb:

- Standard OpenTelemetry export.
- AI-specific attributes are recognized (or appear as custom attributes).
- Dashboards built using the vendor's tooling.
- Cross-system correlation works (AI spans alongside database spans).

### 5.4 The self-hosted integration

For Phoenix / Langfuse:

- The team deploys the open-source product.
- Trace export to the local deployment.
- Configuration, scaling, backups all team-owned.

The team gains control at the cost of operational ownership.

### 5.5 The vendor sidecar pattern

Some vendors operate as sidecars in the request path:

- Portkey routes requests through their service; they capture cost and behavior.
- Pros: zero application-side instrumentation; the vendor handles everything.
- Cons: deepest vendor lock-in; the application's request path depends on the vendor's availability.

Less common in mature production; popular in early-stage teams.

---

## 6. Avoiding vendor lock-in

The discipline that keeps vendor swaps tractable.

### 6.1 The instrumentation independence

The wrapper produces traces in OpenTelemetry format with `gen_ai.*` and `ai.*` attribute conventions per [trace-and-span-design.md](./trace-and-span-design.md). Standard format; no vendor-specific structures.

### 6.2 The configuration-driven vendor selection

Vendor choices are configuration:

```yaml
trace_exporters:
  - vendor: langsmith
    enabled: true
    api_key_secret: langsmith_api_key
  - vendor: datadog
    enabled: true
    api_key_secret: datadog_api_key
  - vendor: helicone
    enabled: false  # disabled; can be re-enabled
```

Adding or removing a vendor is a configuration change; no application code change.

### 6.3 The exit strategy

Per vendor, the team documents:

- How to disable the export.
- How to migrate data out (if the vendor stores trace history).
- How to maintain equivalent functionality without the vendor.

The exit strategy is rehearsed periodically (vendor outage drills).

### 6.4 The eval portability

Eval cases live in the team's repository (per [golden-set-design.md](../eval-engineering/golden-set-design.md) section 4.4). Even if the eval vendor changes, the cases come with the team.

### 6.5 The cost-telemetry portability

Cost is computed locally per [llm-call-instrumentation.md](./llm-call-instrumentation.md). The team's cost telemetry is independent of any vendor's cost computation.

### 6.6 The vendor swap workflow

When swapping vendors:

1. Enable the new vendor's exporter in the collector.
2. Run both vendors in parallel for a calibration period (~2 weeks).
3. Verify data parity (the new vendor's view matches the old).
4. Switch dashboards / alerts / runbooks to the new vendor.
5. Disable the old vendor's exporter.

Total swap time: 4-6 weeks typically; the dual-run period is the longest.

---

## 7. Cost considerations

Vendors have costs.

### 7.1 The per-vendor pricing models

- **Per-trace pricing.** Cost scales with trace volume. Common for AI eval vendors.
- **Per-call pricing.** Cost scales with LLM call volume. Common for cost-observability vendors.
- **Tiered subscription.** Flat cost up to a tier limit. Common for general APM.
- **Self-hosted (open-source).** Compute and storage costs only; no vendor fee.

The pricing model affects the cost-vs-scale trajectory.

### 7.2 The cumulative cost

For Meridian (~10K traces/day from ai-platform):
- LangSmith: ~$300-500/month (per-trace tier).
- Datadog LLM Observability: ~$200-400/month (subscription tier).
- Helicone: ~$0 (cost observability via traces; entry tier).
- Self-hosted Phoenix: ~$50/month compute.

Total: ~$500-1000/month for 3-vendor stack. Bounded; small fraction of overall AI spend.

### 7.3 The cost-attribution back-pressure

Some vendor costs grow with workload growth. The team:

- Monitors per-vendor cost.
- If a vendor's cost grows faster than value: re-evaluate.
- Negotiate enterprise pricing at scale.

### 7.4 The cost-vs-value trade-off

Per vendor:

- What does this vendor provide that we couldn't otherwise achieve?
- Is the cost justified by the value?

Periodically (annually): re-evaluate the trade-off; some vendors may be retired in favor of consolidated tooling.

---

## 8. Vendor evaluation workflow

The structured process for evaluating new vendors.

### 8.1 The evaluation criteria

- **Capability fit.** Does the vendor solve a specific need?
- **Integration cost.** Effort to integrate and operate.
- **Cost.** Subscription / per-trace pricing.
- **Vendor stability.** Funding, team, roadmap.
- **Data security.** SOC 2, BAA for regulated workloads.
- **Lock-in posture.** Can we export data? Can we leave?

### 8.2 The trial pattern

Most vendors offer a trial:

1. Set up trial account.
2. Configure the collector to export to the trial endpoint (in parallel with existing vendors).
3. Use the vendor's UI for 2-4 weeks.
4. Evaluate against the criteria.
5. Decision: adopt, defer, or reject.

The parallel evaluation doesn't disrupt production.

### 8.3 The proof-of-concept

For larger evaluations:

- Define specific use cases the vendor should accelerate.
- Time-box the POC (4-6 weeks).
- Measure: did the vendor accelerate the use cases? How much?
- Decision based on measurement.

### 8.4 The adoption decision

If adopting:

- Negotiate contracting (especially for regulated workloads requiring BAA).
- Define the integration approach.
- Schedule the rollout (often with a sunset of an existing vendor).
- Document.

### 8.5 The retirement decision

For existing vendors no longer earning their cost:

- Identify replacement (if needed).
- Migrate data / workflows.
- Decommission per the exit strategy.

Retirement is a quarterly discipline; vendors are not adopted forever by default.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The vendor stack

Meridian's current vendor stack:

- **Datadog LLM Observability.** Primary unified observability. Per-feature alerts, dashboards, cross-system correlation.
- **LangSmith.** AI-specific debugging and eval. Trace-as-debuggable-LLM-call view; eval workflows.
- **S3 + KMS** (custom). Long-term trace archive for compliance.

Three integration points; the OpenTelemetry collector routes to all three.

### 9.2 The integration pattern

```
meridian-ai-platform (with wrapper from llm-call-instrumentation.md)
    │
    ▼ OpenTelemetry export
OpenTelemetry Collector (deployed alongside the platform)
    │
    ├─→ Datadog (95% sampled traces + all metrics)
    ├─→ LangSmith (100% of clinical traces for eval; 10% otherwise)
    └─→ S3 (100% with PHI redaction; 7-year retention)
```

Each vendor gets the right slice of data for their role.

### 9.3 The vendor evaluation history

Vendors evaluated in 2025-2026:

- **Helicone:** trialed in 2025-Q4 for cost observability. Decision: not adopted; Datadog's cost dashboards plus the custom dashboard per [cost-dashboards.md](./cost-dashboards.md) covered the need.
- **Phoenix (Arize):** trialed in 2026-Q1 as an alternative to LangSmith. Decision: stayed with LangSmith; team's familiarity and existing workflows.
- **Braintrust:** evaluated for the eval workflow. Decision: defer; the team's golden-set discipline is satisfied by LangSmith.

Each evaluation documented; not all vendors adopted.

### 9.4 The lock-in posture

Per section 6:
- All instrumentation is OpenTelemetry-based.
- Vendor configuration is in YAML; no application code dependencies.
- Eval cases in `meridian-eval` repository; portable.
- Cost computation in the wrapper; vendor-independent.

The exit strategy for LangSmith: re-route the collector to a different eval vendor; eval cases ship with the team; dashboards (currently in LangSmith UI) would be rebuilt in the new vendor or in Datadog.

### 9.5 The cost

- Datadog: ~$420/month for LLM observability tier.
- LangSmith: ~$340/month for the team's eval volume.
- S3 storage: ~$15/month (compressed traces).
- OTel collector compute: ~$80/month (k8s pods).

Total: ~$855/month. Below 1% of overall AI spend.

### 9.6 The dual-vendor period

When LangSmith was first adopted (late 2025), the team ran in parallel with Datadog only for ~3 weeks to verify trace consistency. After verification, LangSmith handled AI-specific workflows; Datadog continued for cross-system.

### 9.7 The annual review

Each January:
- Per-vendor cost-vs-value review.
- Retirement decisions if any vendor isn't earning its cost.
- New vendor evaluations queued.

The 2026 review didn't retire any vendors; all three are earning.

### 9.8 The platform discipline

- Instrumentation is OpenTelemetry; vendor-portable.
- Collector configuration-driven.
- Per-vendor exporters can be enabled / disabled.
- Eval cases in version control (not vendor-locked).
- Annual vendor review.

---

## 10. Anti-patterns

### 10.1 "Vendor SDK as the instrumentation"

The team uses LangChain (or another framework) with auto-instrumentation that emits vendor-specific traces. Switching vendors means re-instrumenting.

**Corrective.** Own the instrumentation per [llm-call-instrumentation.md](./llm-call-instrumentation.md); vendor consumes.

### 10.2 "Single vendor for everything"

One vendor for eval, cost, APM. The vendor doesn't excel at all; some functions are mediocre.

**Corrective.** Per-value-add vendor selection per section 2.6.

### 10.3 "Vendor sidecar in the request path"

Vendor sits in the request path; vendor outage takes down the AI feature.

**Corrective.** Vendor consumes telemetry (post-call), doesn't gate calls. Exception: if using a vendor's routing / load-balancing feature, that's a deliberate gateway role.

### 10.4 "Vendor cost-pipeline as source of truth"

The team relies on the vendor's cost computation; vendor pricing-table drift causes inaccurate dashboards.

**Corrective.** Cost computed locally per [llm-call-instrumentation.md](./llm-call-instrumentation.md); vendor is a visualization layer.

### 10.5 "Eval cases locked in vendor format"

Golden-set cases live in the vendor's proprietary format; switching vendors requires re-authoring.

**Corrective.** Cases in version control per [golden-set-design.md](../eval-engineering/golden-set-design.md) section 4.4.

### 10.6 "No vendor evaluation cadence"

Vendors adopted years ago are still in place; better alternatives have emerged; team doesn't notice.

**Corrective.** Annual vendor review per section 8.5.

### 10.7 "Vendor adoption without exit strategy"

Vendor adopted; no plan for how to leave; lock-in deepens with each year.

**Corrective.** Exit strategy documented per section 6.3.

### 10.8 "Vendor cost-vs-value not measured"

The team can't articulate what each vendor provides; cost is just paid.

**Corrective.** Annual cost-vs-value review per section 7.4.

---

## 11. Findings (sprint-assignable)

### VENDOR-001 — Severity: Critical
**Finding.** Vendor SDK auto-instrumentation is the source of traces; vendor lock-in is deep.
**Recommendation.** Own the instrumentation per [llm-call-instrumentation.md](./llm-call-instrumentation.md); use OpenTelemetry export.
**Owner.** ai-platform-eng + observability-eng, sprint N+1.

### VENDOR-002 — Severity: Critical
**Finding.** Vendor sidecar in the request path; vendor outage cascades to AI feature.
**Recommendation.** Vendor consumes post-call telemetry; don't gate the request path.
**Owner.** ai-platform-eng, sprint N+1.

### VENDOR-003 — Severity: High
**Finding.** Eval cases in vendor-proprietary format; lock-in.
**Recommendation.** Cases in version control per [golden-set-design.md](../eval-engineering/golden-set-design.md).
**Owner.** ai-platform-eng, sprint N+2.

### VENDOR-004 — Severity: High
**Finding.** No exit strategy per vendor; lock-in deepening.
**Recommendation.** Document exit strategy per section 6.3.
**Owner.** ai-platform-eng team lead, sprint N+2.

### VENDOR-005 — Severity: High
**Finding.** Vendor's cost computation used as source of truth; pricing-table drift causes errors.
**Recommendation.** Cost computed locally per [llm-call-instrumentation.md](./llm-call-instrumentation.md).
**Owner.** ai-platform-eng + finops, sprint N+2.

### VENDOR-006 — Severity: High
**Finding.** Annual vendor review not scheduled; vendors accumulate without re-evaluation.
**Recommendation.** Annual review per section 8.5.
**Owner.** ai-platform-eng team lead, sprint N+2.

### VENDOR-007 — Severity: High
**Finding.** OpenTelemetry collector not deployed; single-vendor export.
**Recommendation.** Collector pattern per section 4.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### VENDOR-008 — Severity: Medium
**Finding.** Cost-vs-value review not done; vendor adoption decisions are ad-hoc.
**Recommendation.** Annual cost-vs-value review per section 7.4.
**Owner.** ai-platform-eng + finops, sprint N+3.

### VENDOR-009 — Severity: Medium
**Finding.** Single vendor for all observability needs; some functions are mediocre.
**Recommendation.** Per-value-add vendor selection per section 2.6.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### VENDOR-010 — Severity: Medium
**Finding.** Vendor evaluation workflow ad-hoc; trial decisions are gut-feel.
**Recommendation.** Structured evaluation per section 8.
**Owner.** ai-platform-eng team lead, sprint N+3.

### VENDOR-011 — Severity: Medium
**Finding.** Vendor-specific span attributes used in primary platform code; vendor swap requires application changes.
**Recommendation.** Standard OpenTelemetry attributes per [trace-and-span-design.md](./trace-and-span-design.md); vendor-specific are sidelined.
**Owner.** ai-platform-eng, sprint N+3.

### VENDOR-012 — Severity: Medium
**Finding.** Per-vendor cost not tracked; total vendor spend invisible.
**Recommendation.** Per-vendor cost tracking per section 7.2.
**Owner.** ai-platform-eng + finops, sprint N+3.

### VENDOR-013 — Severity: Medium
**Finding.** Vendor outages not handled per-exporter; one vendor's issue blocks others.
**Recommendation.** Independent export per section 4.5.
**Owner.** ai-platform-eng + sre, sprint N+4.

### VENDOR-014 — Severity: Medium
**Finding.** Trial pattern not used; vendor evaluations require full integration.
**Recommendation.** Trial in parallel per section 8.2.
**Owner.** ai-platform-eng, sprint N+4.

### VENDOR-015 — Severity: Medium
**Finding.** PHI redaction not applied before vendor export; sensitive data leaving the boundary.
**Recommendation.** Redaction in the collector per section 4.3.
**Owner.** ai-platform-eng + security-eng, sprint N+2.

### VENDOR-016 — Severity: Low
**Finding.** Vendor decision documentation thin; team can't articulate why each vendor was chosen.
**Recommendation.** Per-vendor decision document.
**Owner.** ai-platform-eng team lead, sprint N+5.

### VENDOR-017 — Severity: Low
**Finding.** Self-hosted alternatives not evaluated.
**Recommendation.** Periodic self-hosted evaluation per section 2.4.
**Owner.** ai-platform-eng, sprint N+5.

### VENDOR-018 — Severity: Low
**Finding.** Vendor outage drill not rehearsed; team doesn't know how to recover.
**Recommendation.** Annual drill per the exit strategy.
**Owner.** ai-platform-eng + sre, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team choosing AI observability vendors:

- [ ] **Sprint 0 — own the instrumentation.** Per [llm-call-instrumentation.md](./llm-call-instrumentation.md) and [trace-and-span-design.md](./trace-and-span-design.md); OpenTelemetry-based.
- [ ] **Sprint 1 — collector pattern.** OpenTelemetry collector deployed; configuration-driven exports.
- [ ] **Sprint 1 — first vendor.** Pick the vendor that addresses the most-immediate need.
- [ ] **Sprint 2 — secondary vendors.** Add additional vendors for complementary value-adds.
- [ ] **Sprint 2 — exit strategies.** Document per vendor.
- [ ] **Sprint 3 — PHI / sensitive-data redaction.** In the collector before export.
- [ ] **Sprint 3 — per-vendor cost tracking.** Visibility into vendor spend.
- [ ] **Sprint 4 — vendor evaluation workflow.** Trial pattern; structured criteria.
- [ ] **Sprint 4 — annual review cadence.** Cost-vs-value; retirement decisions.
- [ ] **Sprint 5 — outage drill.** Rehearse vendor unavailability.
- [ ] **Ongoing — discipline.** Cases in version control; cost computed locally; vendor swaps as configuration.

A team that completes this sequence has a vendor strategy that supports current needs and accommodates future changes. A team that locks in early pays the lock-in tax for the lifetime of the system.

---

## 13. References

- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — instrumentation foundation.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — per-call wrapper.
- This repo: [observability-and-telemetry/quality-drift-detection.md](./quality-drift-detection.md) — drift discipline.
- This repo: [observability-and-telemetry/cost-dashboards.md](./cost-dashboards.md) — cost dashboards.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](./alerting-and-paging-design.md) — alerts.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — eval owned by team.
- This repo: [eval-engineering/golden-set-design.md](../eval-engineering/golden-set-design.md) — cases in version control.
- OpenTelemetry collector documentation.
- OpenTelemetry `gen_ai.*` semantic conventions.
- LangSmith, Braintrust, Phoenix, Helicone, Portkey, Datadog LLM Observability product documentation.
