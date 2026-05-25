# Retrieval Observability

> **Audience.** RAG engineers and tech leads operating a retrieval pipeline in production. The owner of "how do we know retrieval is healthy" and "how do we tune retrieval based on what we see in production." **Scope.** The *engineering* view of retrieval observability — dashboards, SLIs, debugging workflows, weekly health review, pipeline tuning from observability. Complements (not duplicates) [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) which covers the trace/span emission. **Worked client.** Meridian Health.

---

## 1. Why this document exists

[retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) covers how to emit traces and spans from the retrieval pipeline. This document covers how the RAG engineer *uses* that telemetry to operate the pipeline. The instrumentation produces signal; observability turns signal into action.

Most teams stand up the instrumentation and stop there. Traces are emitted; spans are recorded; nothing actionable happens until a production incident forces investigation. The discipline this document codifies: retrieval observability is its own operational practice — weekly health reviews, per-corpus dashboards, drift detection, debugging playbooks, tuning workflows that consume observability data.

This document is opinionated about three things:

1. **The retrieval pipeline has its own SLIs, separate from the broader system.** Per-retriever latency, per-corpus recall (online sample), per-tenant retrieval cost, empty-retrieval rate. Each is its own production-quality signal.
2. **Debugging workflows are written down.** When retrieval-related quality issues fire, on-call walks through documented diagnostic steps. The trace is the primary evidence; the workflow tells them what to look at.
3. **Observability drives tuning.** HNSW parameter changes, rerank threshold changes, candidate count changes — all informed by observability data. Not by intuition.

Structure: (2) the retrieval SLI taxonomy; (3) per-corpus and per-tenant dashboards; (4) debugging workflows; (5) the weekly health review; (6) tuning from observability; (7) integration with broader alerting; (8) the production replay pattern; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The retrieval SLI taxonomy

The signals that matter for retrieval health.

### 2.1 Latency SLIs

- **End-to-end retrieval latency (p50, p95, p99).** From wrapper invocation to results returned. The user-perceived latency of retrieval.
- **Per-stage latency.** Query embedding latency, BM25 latency, vector latency, rerank latency. Decomposed for tuning.
- **Cold-cache vs hot-cache latency.** If caching is in use, the cache hit/miss latency profiles.

Thresholds: workload-specific. For Meridian: p95 retrieval < 600ms; per-stage thresholds align.

### 2.2 Quality SLIs

- **Online judge recall (sampled).** Sample production retrievals; run judge; measure whether expected sources are returned. Per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md).
- **Empty-retrieval rate.** What fraction of retrievals return zero results? Baseline is workload-specific; spikes are a signal.
- **Top-1-confidence distribution.** The retrieval score of the top result; declining distribution suggests retrieval is failing to find good matches.
- **Rerank-lift on production traffic.** Aggregate measure: does rerank consistently improve precision on production traffic?

### 2.3 Throughput SLIs

- **Queries per second.** Per pipeline instance and aggregate.
- **Per-corpus QPS.** How load distributes across corpora.
- **Per-tenant QPS.** For multi-tenant systems; identifies hot tenants.

### 2.4 Cost SLIs

- **Cost per retrieval.** Reranker + embedding + compute. Trend over time.
- **Cost by corpus.** Which corpora drive cost.
- **Cost by tenant.** For multi-tenant; aligns with per-tenant cost circuits per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).

### 2.5 Failure SLIs

- **Retrieval failure rate.** Errors from the underlying retrievers (DB connection failures, API timeouts).
- **Scope-violation rate.** Should be near zero; any non-zero is a Sev-1 per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
- **Validation failure rate.** Post-retrieval validation failures (NaN embeddings, missing metadata).

### 2.6 The SLI hierarchy

Each SLI has:
- Baseline (historical norm).
- Alert threshold (warning).
- Page threshold (Tier 1 alert).

Per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).

---

## 3. Per-corpus and per-tenant dashboards

Aggregated views for operational use.

### 3.1 The per-corpus dashboard

For each active corpus:

- **Corpus health.** Last ingestion run; pending updates; quarantine count.
- **Corpus size.** Document count; chunk count; index size; growth rate.
- **Retrieval against this corpus.** QPS; latency; recall (online sample); empty rate.
- **Cost.** Retrieval cost attributed to this corpus.
- **Per-document-class breakdown.** If the corpus has internal classes, per-class metrics.

The dashboard is the RAG engineer's "is this corpus healthy?" view at a glance.

### 3.2 The per-tenant dashboard

For multi-tenant systems:

- **Per-tenant retrieval QPS.**
- **Per-tenant retrieval cost.**
- **Per-tenant retrieval latency** (some tenants may have larger sub-corpora with different latency profiles).
- **Per-tenant scope-verification status** (should always be 100% pass; failures are Sev-1).

The dashboard supports tenant-level operational analysis.

### 3.3 The per-question-class dashboard

When the workload has classifiable question types:

- **Per-class retrieval recall.**
- **Per-class latency.**
- **Per-class cost.**

Per [tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md), workload classification surfaces patterns; the per-class dashboard makes them visible.

### 3.4 The drift dashboard

Long-term trend visualization:

- Latency p95 over the past 90 days.
- Recall over the past 90 days.
- Cost per retrieval over the past 90 days.

Drift in any direction is investigated; persistent drift triggers tuning or migration.

### 3.5 The dashboard ownership

Each dashboard has an owner (ai-platform-eng for retrieval-platform dashboards; specific feature teams for feature-specific). The owner:

- Watches the dashboard weekly.
- Investigates spikes or drift.
- Acts on findings (tuning, escalation, ticket creation).

Dashboards without owners decay; the ownership keeps them current.

---

## 4. Debugging workflows

When retrieval-related issues fire, on-call walks through documented diagnostic steps.

### 4.1 The "why didn't retrieval find X" workflow

User reports: "I asked about X but the system gave me an irrelevant answer."

1. **Pull the trace** by interaction ID.
2. **Find the retrieval span(s).** Per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md), the span includes query, returned doc IDs, scores.
3. **Was X in the returned doc IDs?**
   - **Yes** → retrieval found it; the model failed to use it. Generation issue.
   - **No** → retrieval didn't find it. Continue diagnosis.
4. **Check the per-retriever sub-spans.** Did BM25 or vector retrieve X but it was filtered out by the merge or rerank?
5. **Check the filter** that was applied. Was X excluded by tenant scope or content-type filter?
6. **Re-run the query against the corpus.** Is X in the corpus at all? When was it last updated?
7. **Investigate the chunking.** Was X chunked in a way that the embedding represents it poorly?
8. **Investigate the embedding.** Embedding model version match? Drift?

The workflow ends with a specific diagnosis: retrieval-stage issue, generation-stage issue, corpus-stage issue, or ingestion-stage issue. The fix follows the diagnosis.

### 4.2 The "retrieval is slow" workflow

Latency SLI breached.

1. **Check per-stage latency.** Which stage spiked?
2. **Embedding API latency?** Check the embedding provider's status; check for rate-limit issues.
3. **BM25 latency?** Check the database query plan; check for index bloat.
4. **Vector latency?** Check HNSW efSearch; check vector index size; check for concurrent index rebuild.
5. **Rerank latency?** Check the rerank provider; check candidate count.
6. **Cache miss rate?** If hot-query caching is in use, has the hit rate dropped?

Each stage has its own diagnostic path.

### 4.3 The "empty retrieval spike" workflow

Empty-retrieval rate has spiked above baseline.

1. **Per-corpus breakdown.** Which corpus is producing the empty retrievals?
2. **Per-tenant breakdown.** Specific tenant?
3. **Recent ingestion runs.** Did a recent ingestion succeed? Was content removed?
4. **Recent filter changes.** Has a filter been added or tightened?
5. **Recent embedding change.** Has the embedding model version changed?
6. **Query patterns.** Are users asking new kinds of questions the corpus doesn't cover?

The diagnosis identifies whether the issue is in ingestion, configuration, model, or workload shift.

### 4.4 The "cost spike" workflow

Cost SLI breached.

1. **Per-corpus cost breakdown.** Which corpus is driving the spike?
2. **Per-tenant cost breakdown.** Specific tenant abuse?
3. **Per-stage cost breakdown.** Is it embedding API, rerank API, or compute?
4. **Recent volume changes.** Has QPS spiked?
5. **Recent config changes.** Has candidate count, rerank, or cache configuration changed?

Per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md), the cost circuit may have already fired; the workflow identifies the cause.

### 4.5 The "scope violation" workflow (Sev-1)

A scope violation event has been recorded — a retrieved chunk had the wrong tenant attribution.

This is a Sev-1 isolation failure. Workflow per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md) section 7.3 (drift detection) plus [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) Sev-1 protocol.

### 4.6 The workflow documentation

Each workflow is a runbook document:

- Trigger (what alert or signal initiated the workflow).
- Diagnostic steps (specific dashboards, queries, traces to consult).
- Common causes (the patterns the team has seen before).
- Remediation steps.
- Escalation path.

Runbooks are rehearsed quarterly per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) section 6.4.

---

## 5. The weekly health review

The operational cadence that keeps observability tied to action.

### 5.1 The agenda

Weekly RAG-engineering review (30 minutes):

- **Per-corpus health.** Quick scan of each corpus dashboard; flag concerning trends.
- **SLI trend.** Latency, recall, cost SLIs over the past week.
- **Incident review.** Any retrieval-related incidents from the past week; root causes; preventive actions.
- **Quarantine triage.** Quarantined documents from ingestion; route to upstream owners.
- **Feedback queue.** Production user feedback (per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md)) routed to regression-case promotion.
- **Tuning queue.** Items identified for retrieval tuning (HNSW parameters, rerank thresholds).

### 5.2 The participants

- ai-platform-eng (RAG owner).
- A representative from any feature team that consumes retrieval.
- On-call from the past week (to share incident context).

The participation is light; the review is focused.

### 5.3 The output

Action items:

- Tuning experiments to run.
- Configuration changes to deploy.
- Regression cases to add.
- Escalations to upstream owners.
- Documentation updates.

Each action item has an owner and a target sprint.

### 5.4 The trend tracking

Over months, the review surfaces patterns:

- "Latency is creeping up; we may need to tune HNSW or add replicas."
- "Cost per retrieval has dropped 12% since adopting hot-query caching."
- "Conversational subset recall has stabilized at 86%; no further action."

The trends inform longer-term planning.

### 5.5 The discipline

Without the weekly review, observability accumulates without action. The review is the engineering practice that closes the loop.

---

## 6. Tuning from observability

Observability data is the input to tuning decisions.

### 6.1 The tuning surfaces

- **HNSW parameters** (efSearch, M, efConstruction) for pgvector or similar.
- **Candidate counts** for retrieval and reranking.
- **Rerank threshold** (if used).
- **Cache TTL and size.**
- **Conditional rerank skip thresholds.**
- **Conditional query-rewrite skip thresholds.**

Each surface is tunable based on observed data.

### 6.2 The tuning workflow

1. **Identify a tuning candidate.** From observability: latency above target on a specific corpus; or cost above target; or recall declining.
2. **Hypothesize a change.** E.g., "reduce candidate count from 50 to 30 to speed up rerank."
3. **Shadow-test.** Per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md), run eval against the proposed change.
4. **Evaluate trade-off.** Does the change meet the goal without unacceptable quality drop?
5. **Deploy.** Per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md), the config change is a release artifact.
6. **Monitor.** Watch SLIs post-deploy; verify the change produced the intended effect.

The workflow is structured; tuning is not ad-hoc.

### 6.3 The A/B tuning

For uncertain changes:

- Deploy the change to a percentage of traffic (canary).
- Compare SLIs between canary and baseline.
- Ramp or rollback based on results.

Tuning changes that significantly affect production are A/B-tested.

### 6.4 The tuning history

Each tuning change is documented:

- Date.
- What changed.
- Rationale (which observability signal motivated it).
- Result (post-deploy SLI delta).

The history is useful for:
- Understanding the system's evolution.
- Reversing changes that turned out to be wrong.
- Reasoning about future tuning.

### 6.5 The tuning frequency

Active tuning is more common in early production (the team is still finding the right parameters); becomes less frequent as the system stabilizes.

For Meridian: active tuning weekly in the first 6 months; quarterly tuning review in steady state.

---

## 7. Integration with broader alerting

Retrieval observability feeds the broader AI alerting per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md).

### 7.1 The Tier 1 retrieval alerts

- **Scope violation event.** Any → Sev-1.
- **Empty-retrieval rate above threshold.** Sustained → Sev-1.
- **Retrieval latency p95 above SLO.** Sustained → Sev-1.
- **Retrieval failure rate above threshold.** Sustained → Sev-1.

### 7.2 The Tier 2 retrieval alerts

- **Per-corpus quality drift.** Warning.
- **Cost above warning threshold.** Slack notification.
- **Cache hit rate dropped.** Slack notification.

### 7.3 The runbook integration

Each alert has a runbook (per section 4). On-call receives the alert; runbook tells them what to do.

### 7.4 The post-incident review

After every retrieval-related Sev-1 or Sev-2:

- Post-mortem per the broader SRE practice.
- Findings: was the observability adequate? Did the runbook help?
- Updates: add missing instrumentation; update the runbook.

The discipline keeps observability improving.

---

## 8. The production replay pattern

For investigation, the team can replay production retrievals.

### 8.1 The replay setup

A captured production retrieval (with query, context, scope) can be re-executed in a non-production environment:

- The replay reproduces the production retrieval state.
- The team can test configuration changes against the replay.
- The replay produces the same trace as production (with the configuration delta).

### 8.2 The replay use cases

- **Reproducing a quality issue.** "Re-run this failed retrieval with the current pipeline; does it still fail?"
- **Testing a configuration change.** "Re-run last week's traffic with the new HNSW parameters; what's the latency / recall delta?"
- **Eval against production traffic.** Replay a sample of production traffic against the eval suite (per [eval-of-rag.md](../eval-engineering/eval-of-rag.md)).

### 8.3 The replay implementation

- Production traces are stored (per [trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md)) for the retention window.
- The replay harness reads the trace; reconstructs the retrieval call; executes against the test pipeline.
- Results are compared to the original.

### 8.4 The replay observability

Replay execution also emits traces; investigators compare:
- Original production trace.
- Replay trace.

Differences in returned doc IDs, scores, latency reveal the impact of the change.

### 8.5 The cost of replay

Replay consumes the same per-call cost as production (embedding API, rerank API). For investigation: cost is small; for full-traffic replay: budget accordingly.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The dashboards

Meridian's retrieval dashboards:

- **Per-corpus health.** clinical-guidelines, drug-interactions, tenant-protocols (240 tabs), patient-education.
- **Per-class retrieval recall.** Clinical-protocol, drug-interaction, conversational, refusal, side-effect classes.
- **Per-tenant.** Top-20 tenants by volume; aggregate for the rest.
- **Latency.** P50, p95, p99 end-to-end and per-stage.
- **Cost.** Daily cost per corpus and per feature.

### 9.2 The SLIs

| SLI | Current value | Baseline | Threshold |
|---|---|---|---|
| End-to-end retrieval p95 latency | 487ms | ~480ms | warning > 600ms; page > 800ms |
| Online judge recall (clinical) | 96% | 95% | warning < 90%; page < 85% |
| Empty-retrieval rate (clinical) | 1.8% | 2% | warning > 4%; page > 8% |
| Retrieval failure rate | 0.02% | 0.05% | warning > 0.5%; page > 2% |
| Scope-violation rate | 0% | 0% | page on any non-zero |
| Cost per retrieval | $0.0027 | $0.0027 | warning > $0.0035 |

All SLIs within targets; system is healthy.

### 9.3 The weekly review

Recent weekly review (week of 2026-05-19):

- All SLIs within targets.
- One incident: a brief latency spike in vector retrieval (the 2026-Q1 efSearch incident remediation). Already resolved; runbook worked.
- 14 quarantined documents from clinical-guidelines ingestion (AHA released new HTML structure); routed to medical-content-licensing team.
- 8 feedback items triaged; 4 promoted to golden set / regression suite.
- 1 tuning experiment proposed: reduce HNSW efSearch from 40 to 32 for latency improvement.

Action items recorded; sprint assigned.

### 9.4 The debugging in practice

A recent debugging session (2026-Q2 cardiology regression, also referenced in [chunking-engineering.md](./chunking-engineering.md)):

1. Online judge alert fires; recall on cardiology subset dropped from 89% to 78%.
2. On-call pulls the dashboard: per-class retrieval recall confirms the drop is in cardiology.
3. Per-corpus dashboard: clinical-guidelines retrieval recall for cardiology specifically dropped; other classes stable.
4. Recent ingestion runs: cardiology guidelines were updated 3 days ago (AHA Q2 release).
5. Per-document inspection: the updated docs have SVG figures; conversion to text was lossy.
6. Fix: converter updated; affected docs re-ingested.
7. Recall restored within the same business day.

The debugging followed the documented workflow (section 4.1); the trace was the primary evidence.

### 9.5 The tuning workflow in practice

The 2026-Q1 HNSW tuning incident:

1. Latency spike observed; per-stage breakdown identified vector retrieval as the cause.
2. Hypothesis: bump efSearch to 100 for better recall.
3. Shadow test: recall up 1 point; latency up 200ms.
4. Decision: revert; recall gain not worth latency hit.
5. Tuning history updated.

The team didn't ship the change; the workflow caught the regression in shadow.

### 9.6 The replay use

Production replay used in the cardiology incident:

- Pulled traces of recent failed cardiology retrievals.
- Re-ran them against the test pipeline with the updated converter.
- Verified the fix worked before promoting to production.

The replay supported confident deployment.

### 9.7 The platform discipline

- Weekly health review on the calendar (Wednesdays 10am).
- Per-corpus dashboards owned by ai-platform-eng.
- Runbooks documented for each Tier 1 alert.
- Tuning experiments shadow-tested.
- Production replay used for fix verification.

---

## 10. Anti-patterns

### 10.1 "Instrumented but not observed"

Traces are emitted; nobody looks at them until an incident. Investigation requires building dashboards from scratch.

**Corrective.** Per-corpus and per-tenant dashboards per section 3; weekly health review per section 5.

### 10.2 "Alerts without runbooks"

Alerts fire; on-call doesn't know what to do; investigation is ad-hoc and slow.

**Corrective.** Runbooks per section 4.6; rehearsed quarterly.

### 10.3 "Dashboards without owners"

Dashboards exist but nobody is responsible for them. They become stale; metrics drift; the team stops trusting them.

**Corrective.** Dashboard ownership per section 3.5.

### 10.4 "Tuning by intuition"

Engineers tune parameters based on guesses; no shadow testing; no observability validation.

**Corrective.** Tuning workflow per section 6.2.

### 10.5 "No weekly review"

Observability is per-incident only; trends and patterns missed; preventive action absent.

**Corrective.** Weekly review per section 5.

### 10.6 "No production replay capability"

Investigations re-run new queries instead of replaying production cases; the team cannot reproduce specific failures.

**Corrective.** Replay pattern per section 8.

### 10.7 "Per-tenant signal missing"

Multi-tenant system, but observability is aggregate only. Per-tenant issues hide.

**Corrective.** Per-tenant dashboards per section 3.2.

### 10.8 "Drift trend not surfaced"

Slow degradation accumulates; nobody notices because daily metrics still look okay; eventually a Sev-1 fires.

**Corrective.** Long-term trend dashboard per section 3.4.

---

## 11. Findings (sprint-assignable)

### ROBS-001 — Severity: Critical
**Finding.** Retrieval traces emitted but never observed; no dashboards.
**Recommendation.** Per-corpus and per-tenant dashboards per section 3.
**Owner.** ai-platform-eng + observability-eng, sprint N+1.

### ROBS-002 — Severity: Critical
**Finding.** Alerts on retrieval SLIs fire without runbooks; on-call confused.
**Recommendation.** Runbooks per section 4.6.
**Owner.** ai-platform-eng + sre, sprint N+1.

### ROBS-003 — Severity: High
**Finding.** Weekly retrieval health review not scheduled.
**Recommendation.** Schedule per section 5.
**Owner.** ai-platform-eng team lead, sprint N+2.

### ROBS-004 — Severity: High
**Finding.** Tuning changes shipped without shadow testing.
**Recommendation.** Tuning workflow per section 6.2.
**Owner.** ai-platform-eng, sprint N+2.

### ROBS-005 — Severity: High
**Finding.** Per-tenant retrieval signal missing for multi-tenant system.
**Recommendation.** Per-tenant dashboards per section 3.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### ROBS-006 — Severity: High
**Finding.** Long-term trend dashboards absent; slow drift missed.
**Recommendation.** 90-day trend dashboards per section 3.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ROBS-007 — Severity: High
**Finding.** Production replay capability absent; investigations rerun fresh queries.
**Recommendation.** Replay harness per section 8.
**Owner.** ai-platform-eng, sprint N+3.

### ROBS-008 — Severity: High
**Finding.** Dashboards without owners; metrics decay.
**Recommendation.** Ownership per section 3.5; quarterly review.
**Owner.** ai-platform-eng team lead, sprint N+2.

### ROBS-009 — Severity: Medium
**Finding.** Per-stage latency not separately tracked; bottleneck investigation requires correlation.
**Recommendation.** Per-stage SLIs per section 2.1.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ROBS-010 — Severity: Medium
**Finding.** Quality SLI (online judge recall) not implemented.
**Recommendation.** Per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) section 7.
**Owner.** ai-platform-eng, sprint N+3.

### ROBS-011 — Severity: Medium
**Finding.** Per-corpus cost attribution missing; tuning decisions made without cost signal.
**Recommendation.** Per-corpus cost in dashboards per section 3.1.
**Owner.** ai-platform-eng + finops, sprint N+3.

### ROBS-012 — Severity: Medium
**Finding.** Post-incident reviews don't include observability gap analysis.
**Recommendation.** Add observability adequacy to post-mortem template per section 7.4.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ROBS-013 — Severity: Medium
**Finding.** Tuning history not maintained; reversals lose context.
**Recommendation.** Tuning history log per section 6.4.
**Owner.** ai-platform-eng, sprint N+4.

### ROBS-014 — Severity: Medium
**Finding.** A/B tuning not framework-supported; experiments are manual.
**Recommendation.** A/B canary capability per section 6.3.
**Owner.** ai-platform-eng + sre, sprint N+4.

### ROBS-015 — Severity: Medium
**Finding.** Per-question-class dashboards absent.
**Recommendation.** Per-class dashboards per section 3.3.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ROBS-016 — Severity: Low
**Finding.** Replay cost not tracked separately from production cost.
**Recommendation.** Replay cost attribution per section 8.5.
**Owner.** ai-platform-eng + finops, sprint N+5.

### ROBS-017 — Severity: Low
**Finding.** Dashboard documentation thin; new engineers don't understand the panels.
**Recommendation.** Dashboard documentation; tooltips on each panel.
**Owner.** ai-platform-eng, sprint N+5.

### ROBS-018 — Severity: Low
**Finding.** Weekly review minutes not retained; action items lost.
**Recommendation.** Meeting notes; action item tracking.
**Owner.** ai-platform-eng team lead, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team with retrieval instrumentation but no operational observability:

- [ ] **Sprint 0 — inventory.** What signals are emitted; what dashboards exist; what runbooks exist.
- [ ] **Sprint 1 — core dashboards.** Per-corpus health; per-stage latency; cost.
- [ ] **Sprint 1 — core alerts.** Per [alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md); runbooks per section 4.
- [ ] **Sprint 2 — quality SLI.** Online judge per [online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md).
- [ ] **Sprint 2 — per-tenant dashboards.** For multi-tenant systems.
- [ ] **Sprint 3 — weekly health review.** Cadence established; participants identified.
- [ ] **Sprint 3 — tuning workflow.** Shadow testing; A/B canary.
- [ ] **Sprint 4 — production replay.** Replay harness; investigation use cases.
- [ ] **Sprint 5 — drift dashboards.** Long-term trend visualization.
- [ ] **Ongoing — discipline.** Weekly review; tuning history; post-incident gap analysis.

A team that completes this sequence operates retrieval as a known-quality production system. A team that ships instrumentation without the operational practice is collecting data without acting on it.

---

## 13. References

- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — span/attribute emission this document consumes.
- This repo: [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alert hierarchy.
- This repo: [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/) (coming) — cost dashboard patterns.
- This repo: [eval-engineering/online-eval-and-feedback.md](../eval-engineering/online-eval-and-feedback.md) — quality SLI source.
- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — what's being observed.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](./rag-failure-modes-and-debugging.md) — diagnostic patterns this enables.
- This repo: [rag-engineering/rag-eval-integration.md](./rag-eval-integration.md) — eval discipline complement.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost circuit integration.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — operational context.
