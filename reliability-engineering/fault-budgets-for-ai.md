# Fault Budgets for AI

> **Audience.** SREs adapting the error-budget pattern to AI workloads. Engineering leads whose quarterly review asks "how reliable is our AI?" and whose answer is currently "depends what you mean." Anyone whose SRE practice is mature for HTTP services but new to LLM-shaped reliability. **Scope.** The *engineering* practice of fault budgets (error budgets) for AI: the four SLO dimensions (quality, latency, cost, availability); defining each; the error-budget-burn alert pattern; the stop-shipping discipline when the budget is exhausted; integration with the broader reliability practice. Not the timeout / retry / circuit-breaker primitives (see companions). Not the degraded-mode design (see [degraded-mode-design.md](./degraded-mode-design.md)). Not the broader SRE error-budget framework — this is the AI-specific overlay. **Worked client.** Meridian Health.

---

## 1. Why this document exists

The SRE error-budget framework is mature for conventional services: define an SLO (e.g., 99.9% availability), track actual vs target, the difference is the "error budget," when the budget is exhausted stop shipping changes until reliability recovers.

The framework adapts to AI but with three specific extensions:

- **Quality is an SLO dimension.** Conventional services don't have a "quality" SLO; an HTTP 200 is a success regardless of content. AI services have responses that are "successful" by HTTP but wrong by content; quality SLO captures this.
- **Cost is an SLO dimension.** Conventional services don't track per-request cost as part of reliability; AI services do, because cost is a constraint that can fail in ways that latency / availability can't.
- **Multiple SLOs compose.** A feature may be available, fast, and within cost budget — but producing low-quality responses. All four dimensions must hold.

The "stop shipping when budget exhausted" discipline applies in AI too, with adaptation. A team that has exhausted its quality budget shouldn't ship prompt changes; a team that has exhausted its cost budget shouldn't ship features that increase cost.

Without fault-budget discipline, AI reliability decisions are ad-hoc: someone notices something looks bad; mitigation is improvised; cause isn't analyzed; recurrence is likely.

With fault-budget discipline:

- SLOs are explicit and tracked.
- Budget remaining is visible.
- Approaching exhaustion triggers known actions.
- Exhaustion triggers a "stop shipping" or similar disciplined response.

This document covers the engineering: how to define each SLO dimension; how to track them; how to alert on burn rates; how the stop-shipping discipline integrates with the engineering team's planning.

This document is opinionated about four things:

1. **All four dimensions matter; pick all of them, not one.** Some teams pick "availability" and call it done; the other three (quality, latency, cost) silently drift.
2. **Quality SLO requires a quality signal.** Without a judge model or eval pipeline producing pass/fail signal, there's no quality SLO. Build the signal before the SLO.
3. **Cost SLO needs the right granularity.** Per-feature, per-tenant, or aggregate? The choice affects what the budget protects.
4. **Stop-shipping must mean something.** A "we've exhausted our budget" announcement that doesn't change behavior is just reporting.

Structure: (2) the four SLO dimensions; (3) defining the quality SLO; (4) defining the latency SLO; (5) defining the cost SLO; (6) defining the availability SLO; (7) the error-budget-burn alert; (8) the stop-shipping discipline; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The four SLO dimensions

The dimensions to track. Each is its own SLO; together they characterize the feature's reliability.

### 2.1 Quality SLO

The fraction of responses that pass a quality bar.

**Signal.** Live-judge pass rate; schema-validation pass rate; user-feedback (thumbs-up rate).

**Typical target.** 90-95% of responses pass.

**Why it matters.** A response can be available, fast, and cheap — but wrong. Quality is the substantive measure.

### 2.2 Latency SLO

The fraction of requests that complete within a target time.

**Signal.** P99 or P95 latency over a window.

**Typical target.** P99 < N seconds (workload-specific).

**Why it matters.** Slow responses cost cost (in user dropout) and may indicate underlying issues.

### 2.3 Cost SLO

The fraction of the period's cost budget consumed.

**Signal.** Per-feature spend vs budget.

**Typical target.** Less than 100% of budget consumed by period end.

**Why it matters.** Cost overruns are a reliability dimension; budget protection is engineering's responsibility.

### 2.4 Availability SLO

The fraction of requests that complete successfully (no error).

**Signal.** Error rate.

**Typical target.** 99.9% successful.

**Why it matters.** The classic uptime SLO. Still applies; just doesn't capture quality, cost, or latency on its own.

### 2.5 The SLOs as a vector

A feature's reliability is a 4-vector:

```
care-coordinator-reliability:
  quality_slo: 95% pass
  quality_actual: 94.2%
  latency_slo: P99 < 8s
  latency_actual: P99 = 7.4s
  cost_slo: monthly_budget = $35k
  cost_actual_eom_projected: $33.5k
  availability_slo: 99.9% success
  availability_actual: 99.94%

Status: all SLOs met
```

### 2.6 The composite SLO is misleading

A "combined SLO" that averages the four is misleading; each is independent. Track each separately.

### 2.7 The SLO hierarchy

Some SLOs are stricter than others depending on workload:

- Clinical decision support: quality is the strictest; latency moderate; cost moderate; availability strict.
- Patient chat: latency strict; quality moderate; cost moderate; availability moderate.
- Bulk classification: cost strict; latency loose; quality moderate; availability moderate.

Per-workload SLO weighting reflects business priority.

---

## 3. Defining the quality SLO

The novel dimension; requires a quality signal.

### 3.1 What "quality" means

Quality is workload-specific:

- Patient chat: clinical accuracy + tone appropriateness + completeness.
- Care Coordinator: agent task succeeded + clinician approved.
- Document classification: correctly classified.
- Code generation: code passes tests.

Per workload, define what "pass" means.

### 3.2 The judge model signal

For most workloads, a judge model evaluates sampled responses:

```python
def judge_response(query, response):
    judgment = judge_model.evaluate(query, response, criteria=...)
    return judgment.score  # 0.0 to 1.0
```

Cross-link to [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md).

### 3.3 The schema-validation signal

For structured outputs, schema validation is a quality signal:

- Pass: output parses correctly; all required fields present; types correct.
- Fail: parse error, missing field, type mismatch.

Cross-link to [prompt-engineering/structured-output-engineering.md](../prompt-engineering/structured-output-engineering.md).

### 3.4 The user-feedback signal

Thumbs-up / thumbs-down on responses:

- Pass: user clicked thumbs-up or proceeded normally.
- Fail: user clicked thumbs-down or abandoned.

Noisy; supplemented by judge signal.

### 3.5 The quality SLO threshold

Define what fraction of responses must pass:

- 99%: very strict; appropriate for clinical / high-stakes.
- 95%: standard; appropriate for most production AI.
- 90%: relaxed; appropriate for experimental or low-stakes.

The threshold corresponds to risk tolerance.

### 3.6 The sampling rate

For judge-based quality:

- 100% sampling: judge every response. Expensive (judge model cost); thorough.
- 5-10% sampling: judge a fraction. Affordable; statistically representative.
- Sampling-with-stratification: heavier sampling for high-risk workloads.

Choose based on volume + budget.

### 3.7 The "judge isn't always available" reality

The judge model may have its own outages. Plan:

- Backup judge (different model).
- Quality SLO with explicit "judge availability" sub-SLO.
- Periodic batch judge if real-time isn't available.

### 3.8 The quality SLO over a window

Quality is measured over a window (typically 1-7 days):

```
quality_pass_rate_7d = pass_count_7d / total_count_7d
```

Window long enough to be statistically meaningful; short enough to detect issues.

---

## 4. Defining the latency SLO

The classical dimension; AI calibration applies.

### 4.1 The percentile choice

- P50: median; doesn't capture tail.
- P95: 95% of requests; reasonable for most workloads.
- P99: 99% of requests; strict.
- P99.9: 99.9% of requests; extreme.

P99 is typical for AI workloads; the tail is real.

### 4.2 The target choice

Workload-specific:

- Interactive chat: P99 < 5s.
- Streaming TTFT: P99 < 2s.
- Agent task: P99 < 60s.
- Batch: P99 < 24h.

Cross-link to [timeout-strategy.md](./timeout-strategy.md) §2 for typical distributions.

### 4.3 The SLO over a window

Measured over a rolling window (typically 7-30 days):

```
slo_compliance_30d = fraction_of_requests_within_P99_target
target: 99.5% (i.e., 99.5% of windows have P99 below target)
```

### 4.4 The latency SLO and streaming

For streaming workloads, multiple latency SLOs:

- TTFT (time to first token): P99 < 2s.
- Inter-token: P95 < 200ms.
- Total streaming duration: P99 < 60s.

Each has its own SLO.

### 4.5 The per-call-class latency SLO

Different call classes have different latency expectations:

- Classification: P99 < 1s.
- Chat: P99 < 5s.
- Long-context analysis: P99 < 30s.

Per-class SLO per the call class taxonomy.

### 4.6 The "we're over P99 budget but only 4% of the time" interpretation

If P99 target is 5s and the actual P99 is 5.3s, the SLO is "burned" for that window. The error budget tracks how often this happens.

---

## 5. Defining the cost SLO

The financial dimension; AI-specific.

### 5.1 The per-period cost budget

```
cost_slo:
  monthly_budget_usd: 35000
  daily_warning_threshold: 80% of pace
  daily_hard_threshold: 100%
```

The SLO is "not exceed monthly budget."

### 5.2 The cost budget granularity

- Per-feature: each AI feature has a monthly budget.
- Per-tenant: each tenant's monthly cost budget.
- Aggregate: platform-wide AI spend.

Each is its own SLO. Per-feature is the engineering accountability layer; per-tenant is the commercial layer; aggregate is the leadership layer.

### 5.3 The cost SLO and forecast

Cost SLO depends on a forecast:

- Forecast: $30k/month based on volume × cost-per-call.
- Budget: $35k (with 17% headroom).
- SLO: actual ≤ budget.

The forecast is part of the SLO's design.

### 5.4 The burn-rate cost alert

Cost SLO burn-rate (cross-link to [cost-and-finops/cost-dashboards-and-alerts.md §3](../cost-and-finops/cost-dashboards-and-alerts.md)):

```
burn_rate = (spend_so_far / period_elapsed) / (budget / period_total)
```

If burn rate > 1.5, alert at warning; if > 2.0, page; if > 3.0, escalate.

### 5.5 The cost budget exhaustion

When cost is at 100% of monthly budget mid-month:

- Feature degrades (cross-link to [degraded-mode-design.md](./degraded-mode-design.md)).
- New requests may be refused.
- Cost-incident runbook activates (cross-link to [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md)).

### 5.6 The cost SLO as a constraint on shipping

If cost is approaching budget exhaustion, new features that increase cost should not ship until budget is reset. Cross-link to §8.

---

## 6. Defining the availability SLO

The classical dimension.

### 6.1 The success rate

```
availability_slo:
  target: 99.9% successful
  measurement_window: 30 days
```

The fraction of requests that complete without an error (not a timeout, not a failure, not a refusal).

### 6.2 What counts as "successful"

For AI workloads:

- HTTP 200 with parseable response: success.
- HTTP 200 with unparseable response: fail (or "quality-failed" depending on classification).
- HTTP 4xx (caller's fault): fail (but not counted against availability — caller's responsibility).
- HTTP 5xx: fail; counts against availability.
- Timeout: fail; counts against availability.

Customize based on workload.

### 6.3 The error budget

For 99.9% target over 30 days:

- Total requests in 30 days: ~3M (example).
- Allowed errors: 3M × 0.001 = 3,000.
- Budget: 3,000 errors per 30 days.

When > 3,000 errors in 30 days, SLO violated; budget exhausted.

### 6.4 The availability and degraded mode

When degraded mode fires, is it a "successful" response?

- If the response is delivered (even if degraded): success.
- If the request is refused: count as success if explicit (structured error with helpful context) or failure if generic.

Define per-workload.

### 6.5 The "we have 99.9% availability" but quality is 80%

If availability is met but quality is low, the feature is delivering wrong answers fast. Availability SLO alone is insufficient.

### 6.6 The window aggregation

Availability over different windows:

- 1 hour: catches incidents.
- 24 hours: catches sustained issues.
- 30 days: monthly compliance.

Each window has its own threshold; alerts on burn.

---

## 7. The error-budget-burn alert

When the SLO is being burned faster than allowed.

### 7.1 The burn rate

```
burn_rate = (errors_so_far / period_elapsed) / (budget / period_total)
```

Burn rate of 1.0 = on track; 2.0 = burning 2x; 10.0 = burning 10x.

### 7.2 The multi-window alert

Per Google SRE Book practices:

- 1-hour burn rate > 14: fast-burn page (severe; immediate).
- 6-hour burn rate > 6: slow-burn page (sustained; soon).
- 1-day burn rate > 3: ticket (drift; investigate).
- 3-day burn rate > 2: ticket (slow drift).

Each window catches a different failure mode.

### 7.3 The alert payload

Useful alert content:

```
ALERT: Quality SLO burning fast
Feature: care-coordinator
SLO target: 95% quality pass rate
Last hour: 78% pass rate (burn rate 4.4x)
Last 6 hours: 87% (burn rate 1.6x)
Likely cause: recent deploy (14:23)
Runbook: cost-incident-runbook (quality regression class)
```

### 7.4 The "burning slowly is still burning" mindset

Slow burns don't fire as pages but do indicate the SLO will be missed if uncorrected. Tickets / dashboard awareness; address before they become incidents.

### 7.5 The fast-burn protection

For severe burns (10x+), the architecture may automatically:

- Trigger circuit-breaker.
- Switch to degraded mode.
- Page on-call.

Doesn't wait for human reaction; automated protection.

### 7.6 The per-SLO burn handling

Different SLOs trigger different responses:

- Quality burn: investigate prompt / model changes; possibly revert.
- Latency burn: investigate retrieval, model performance, capacity.
- Cost burn: cost-incident runbook (§5 cross-link).
- Availability burn: provider degradation; reliability response.

Each has its own response.

---

## 8. The stop-shipping discipline

What happens when the budget is exhausted.

### 8.1 The disciplinary action

Standard SRE practice: when error budget is exhausted, stop shipping changes to that service. Focus engineering capacity on reliability.

For AI features:

- Quality budget exhausted: stop shipping prompt changes; stop shipping model changes.
- Latency budget exhausted: stop shipping context-size changes; stop shipping model upgrades.
- Cost budget exhausted: stop shipping cost-increasing features.
- Availability budget exhausted: stop shipping risky deploys.

The "stop" is per-budget; selective.

### 8.2 The discipline as a guide, not a hammer

Strict "stop everything" is rarely the right move. Negotiated:

- Critical bug fixes still ship.
- Reliability work prioritized.
- Riskier changes deferred.

The discipline is the framework for the conversation, not the answer.

### 8.3 The "budget exhausted; how do we recover" plan

When budget is exhausted:

- Engineering team's planning shifts toward reliability work.
- New SLO target may be reset (don't immediately recover).
- Customer-facing communication if degradation is visible.

### 8.4 The communication to leadership

Budget exhaustion is reportable:

- Monthly cost review (cross-link to [cost-and-finops/finops-process.md §2](../cost-and-finops/finops-process.md)).
- Leadership report (cross-link to §4 finops-process).
- Trust-building: leadership knows the team is managing reliability, not chasing features.

### 8.5 The cross-team incentive

If the discipline only applies to one team, others continue shipping changes that affect the team's SLOs. Negotiated cross-team:

- Platform team's changes that affect a feature can also be paused.
- Provider-side issues are documented as "outside our control" and don't count.

### 8.6 The "we never exhausted the budget" pattern

If the team never exhausts its budget, the SLO may be too loose. Tighten:

- Better latency target.
- Higher quality threshold.
- Lower cost ceiling.

The SLO should be aspirational, not trivially achievable.

### 8.7 The "we always exhaust the budget" pattern

If the team always exhausts its budget, the SLO may be too strict. Loosen:

- Realistic latency target.
- Achievable quality threshold.
- Adequate cost budget.

The SLO should be challenging, not impossible.

---

## 9. Worked Meridian example

Meridian's SLO framework for AI features.

### 9.1 The SLO catalog

```yaml
features:
  care-coordinator:
    quality_slo:
      target: 95% judge-pass-rate
      sampling: 10% of responses
      window: 7d
    latency_slo:
      target: P99 < 8s (agent task)
      window: 30d
    cost_slo:
      monthly_budget: $35,000
      target: not exceed
    availability_slo:
      target: 99.9% successful task completion
      window: 30d

  patient-api-chat:
    quality_slo:
      target: 92% judge-pass-rate
      sampling: 5%
      window: 7d
    latency_slo:
      target: P99 < 3s (chat response)
      window: 30d
    cost_slo:
      monthly_budget: $10,000
    availability_slo:
      target: 99.9%

  clinical-decision-support:
    quality_slo:
      target: 99% judge-pass-rate (clinical safety)
      sampling: 100%
      window: 7d
    latency_slo:
      target: P99 < 5s
    cost_slo:
      monthly_budget: $3,500
    availability_slo:
      target: 99.95% (safety-critical)
```

### 9.2 The dashboard

For each feature, a SLO dashboard panel:

```
care-coordinator (last 30 days):
  Quality:      94.2% pass rate (target 95%) ⚠️ in deficit
  Latency:      P99 7.2s (target 8s)         ✓ within
  Cost:         $34,500 (target $35,000)     ⚠️ approaching
  Availability: 99.93% (target 99.9%)         ✓ within
```

### 9.3 The burn-rate alerts

Per-SLO burn alerts:

- Quality 1-hour burn > 5x: page.
- Latency 6-hour burn > 3x: page.
- Cost 1-hour burn > 5x (correlated with cost-and-finops alerts): page.
- Availability 1-hour burn > 14x: page.

### 9.4 The Q1 2026 quality SLO incident

Quality dipped to 78% (burn rate 4.4x) for ~2 hours after a prompt deploy (cross-link to [cost-incident-runbook.md §9](../cost-and-finops/cost-incident-runbook.md)).

- Quality budget burned ~30% in 2 hours.
- Deploy reverted; quality recovered to 94%.
- Budget remained but degraded.
- Engineering team committed to no major prompt changes for next 2 weeks (recovery discipline).

### 9.5 The Q2 2026 latency SLO drift

Latency P99 drifted from 7.4s to 8.7s over 3 weeks. Burn rate 1.3x.

- Slow burn; ticket created.
- Investigation: retrieval index grew; cache hit rate dropped; per-call latency rose.
- Mitigation: optimized retrieval; cache hit rate restored.
- P99 returned to 7.5s.

No page (slow burn); ticket-level response sufficed.

### 9.6 The cost SLO over months

Care Coordinator monthly cost vs budget:

```
Jan 2026: $28k (80%)
Feb 2026: $30k (86%)
Mar 2026: $32k (91%)  ← growing
Apr 2026: $34k (97%)  ← warning fired
May 2026: $34.5k (99%) ← burn rate 1.5x; engineering paused new cost-increasing features
Jun 2026: $32k (91%) ← recovered after caching improvements
```

The discipline produced a sustainable cost trajectory.

### 9.7 The stop-shipping during quality regression

The Q1 quality regression triggered selective stop-shipping:

- Prompt changes: paused 2 weeks.
- Model changes: paused 2 weeks.
- Tool changes: continued (not affected).
- Bug fixes: continued.

Selective; not full stop.

### 9.8 The SLO review cadence

- Monthly: review each feature's SLO compliance; discuss in cost review (cross-link).
- Quarterly: re-tune SLO targets based on observed achievability and business priority.
- Annually: SLO framework itself is reviewed.

### 9.9 What the SLO framework costs

- Initial setup: ~3 weeks for the platform team (define SLOs; build dashboards; build alerts).
- Ongoing: ~5% of platform team's time for monitoring + tuning.
- Eval infrastructure (judge model for quality SLO): ~$2k/month.

### 9.10 What the SLO framework produces

- Predictable reliability: SLOs meet targets ~85% of months.
- Early detection of issues via burn alerts.
- Engineering planning informed by reliability data.
- Leadership trust: AI reliability is "known."

---

## 10. Anti-patterns

### 10.1 The "we have an SLA but no SLO" gap

**Pattern.** Customer contract says 99.9% availability. Engineering doesn't track. Customer complains; engineering scrambles.

**Corrective.** SLO precedes SLA; SLO is internal target; SLA is customer commitment.

### 10.2 The "availability only" SLO

**Pattern.** SLO = 99.9% availability. Quality drifts (HTTP 200 with wrong content); cost drifts; nobody notices.

**Corrective.** All four dimensions per §2.

### 10.3 The "we'll add quality SLO when we have eval" deferral

**Pattern.** Quality SLO requires judge / eval; team defers indefinitely. Quality drift catches them.

**Corrective.** Build judge per [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md); then add SLO.

### 10.4 The SLO that no one looks at

**Pattern.** SLO defined; dashboard exists; nobody reviews; budget burns silently.

**Corrective.** Monthly review; alerts on burn; visible to engineering management.

### 10.5 The cost SLO without a forecast

**Pattern.** Cost SLO is "don't exceed N." But N was chosen arbitrarily. Either way too generous or way too tight.

**Corrective.** Forecast per [finops-process.md §5](../cost-and-finops/finops-process.md); budget = forecast + headroom.

### 10.6 The stop-shipping that doesn't stop

**Pattern.** Budget exhausted; team announces "we'll focus on reliability"; ships new features the next week.

**Corrective.** Selective stop per §8; documented; enforced.

### 10.7 The SLO target that's never achieved

**Pattern.** SLO at 99.99% availability for a small startup feature. Always burning; team disregards.

**Corrective.** SLO should be challenging but achievable per §8.7; tune.

### 10.8 The SLO that's trivially achieved

**Pattern.** SLO at 99% availability when actual is 99.99%. Never burns; not informative.

**Corrective.** Tighten per §8.6.

### 10.9 The composite SLO that averages

**Pattern.** "Overall reliability SLO = (quality + latency + cost + availability) / 4." A high availability hides a low quality.

**Corrective.** Each SLO separate per §2.6.

### 10.10 The "cost burned but not stopped"

**Pattern.** Cost budget burning fast; alert fires; team acknowledges; no action; budget exhausts.

**Corrective.** Alert → action mapping per §7.6.

---

## 11. Findings (sprint-assignable)

### REL-BUDGET-001 — Severity: Critical
**Finding.** No fault-budget framework for AI features.
**Recommendation.** Four-dimensional SLO per §2 for each feature.
**Owner.** SRE + AI platform, sprint N+1.

### REL-BUDGET-002 — Severity: Critical
**Finding.** Quality SLO absent.
**Recommendation.** Judge model + quality SLO per §3.
**Owner.** AI platform + eval, sprint N+1.

### REL-BUDGET-003 — Severity: Critical
**Finding.** Cost SLO absent.
**Recommendation.** Per-feature cost SLO per §5.
**Owner.** FinOps + AI platform, sprint N+1.

### REL-BUDGET-004 — Severity: High
**Finding.** Latency SLO not per-call-class.
**Recommendation.** Per-class SLO per §4.5.
**Owner.** AI platform, sprint N+2.

### REL-BUDGET-005 — Severity: High
**Finding.** Burn-rate alerts not multi-window.
**Recommendation.** Multi-window per §7.2.
**Owner.** SRE, sprint N+2.

### REL-BUDGET-006 — Severity: High
**Finding.** Stop-shipping discipline undefined.
**Recommendation.** Per-budget selective stop per §8.
**Owner.** engineering management, sprint N+2.

### REL-BUDGET-007 — Severity: High
**Finding.** SLO dashboard absent.
**Recommendation.** Per-feature SLO panel per §9.2.
**Owner.** observability-eng, sprint N+2.

### REL-BUDGET-008 — Severity: High
**Finding.** Streaming latency not separated from total latency in SLO.
**Recommendation.** Per-streaming-dimension SLO per §4.4.
**Owner.** AI platform, sprint N+2.

### REL-BUDGET-009 — Severity: Medium
**Finding.** Per-tenant cost SLO not tracked separately from per-feature.
**Recommendation.** Per-tenant per §5.2.
**Owner.** AI platform, sprint N+3.

### REL-BUDGET-010 — Severity: Medium
**Finding.** Quality sampling rate not tuned to volume.
**Recommendation.** Sampling per §3.6.
**Owner.** AI platform + eval, sprint N+3.

### REL-BUDGET-011 — Severity: Medium
**Finding.** SLO targets not tuned based on observed achievability.
**Recommendation.** Quarterly tuning per §8.7.
**Owner.** SRE, sprint N+3.

### REL-BUDGET-012 — Severity: Medium
**Finding.** SLO compliance not in monthly review.
**Recommendation.** Add to monthly review per [finops-process.md](../cost-and-finops/finops-process.md).
**Owner.** SRE + engineering management, sprint N+3.

### REL-BUDGET-013 — Severity: Medium
**Finding.** Burn alerts don't route to action.
**Recommendation.** Alert → action mapping per §7.6.
**Owner.** SRE, sprint N+3.

### REL-BUDGET-014 — Severity: Medium
**Finding.** Provider-side issues counted against availability SLO.
**Recommendation.** Document exclusions per §8.5.
**Owner.** SRE, sprint N+4.

### REL-BUDGET-015 — Severity: Low
**Finding.** SLA without SLO underneath.
**Recommendation.** SLO precedes SLA per §10.1.
**Owner.** engineering management + customer success, sprint N+5.

### REL-BUDGET-016 — Severity: Low
**Finding.** "Composite SLO" used; hides individual SLO state.
**Recommendation.** Each separate per §2.6.
**Owner.** SRE, sprint N+5.

### REL-BUDGET-017 — Severity: Low
**Finding.** Annual SLO framework review absent.
**Recommendation.** Review per §8.6-§8.7; tune.
**Owner.** SRE, sprint N+6.

### REL-BUDGET-018 — Severity: Low
**Finding.** Customer-facing SLA derived without engineering input.
**Recommendation.** Joint SLA → SLO derivation.
**Owner.** customer success + engineering management, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Build quality signal (judge model) per §3.**
- [ ] **Define quality SLO per feature.**
- [ ] **Define latency SLO per feature / per call class.**
- [ ] **Define cost SLO per feature.**
- [ ] **Define availability SLO per feature.**
- [ ] **Build SLO dashboard.**
- [ ] **Implement multi-window burn-rate alerts per §7.2.**
- [ ] **Define stop-shipping discipline per §8.**
- [ ] **Integrate SLO review with monthly cost review.**
- [ ] **Quarterly SLO tuning.**
- [ ] **Document SLO framework; new features adopt at launch.**

---

## 13. References

**In this folder.**
- [timeout-strategy.md](./timeout-strategy.md) — latency that informs latency SLO.
- [retry-strategy.md](./retry-strategy.md) — retry policy that affects availability SLO.
- [fallback-patterns.md](./fallback-patterns.md) — degraded delivery may affect SLO accounting.
- [circuit-breakers.md](./circuit-breakers.md) — breakers that protect SLO.
- [degraded-mode-design.md](./degraded-mode-design.md) — degraded mode invoked when SLO is at risk.
- [capacity-planning.md](./capacity-planning.md) *(companion)* — capacity that enables SLO.
- [multi-provider-failover.md](./multi-provider-failover.md) *(companion)* — failover that protects availability SLO.
- [incident-response-for-ai.md](./incident-response-for-ai.md) *(companion)* — incidents that affect SLO.

**Elsewhere in this repo.**
- [eval-engineering/llm-as-judge-patterns.md](../eval-engineering/llm-as-judge-patterns.md) — judge for quality SLO.
- [cost-and-finops/finops-process.md](../cost-and-finops/finops-process.md) — cost SLO and monthly review.
- [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md) — alert infrastructure.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — quality drift signal.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alert design.

**Sibling repos.**
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — per-tenant SLO context.

**External.**
- Google SRE Book — chapter on SLO design and error budgets.
- Google SRE Workbook — chapter on multi-burn-rate alerts.
- "SRE Workbook" by Beyer et al. — practical guidance.
- FinOps Foundation — cost SLO framing.
