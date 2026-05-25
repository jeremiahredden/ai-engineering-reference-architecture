# Alerting and Paging Design for AI Systems

> **Audience.** Engineers and SREs designing the alert / page hierarchy for AI features in production. Tech leads who have either too many AI alerts (everyone tunes them out) or too few (incidents are discovered by users). **Scope.** The *engineering* discipline of choosing which AI signals page, which alert without paging, which feed dashboards only. Pair with [trace-and-span-design.md](./trace-and-span-design.md), [llm-call-instrumentation.md](./llm-call-instrumentation.md), [agent-step-instrumentation.md](./agent-step-instrumentation.md), [retrieval-instrumentation.md](./retrieval-instrumentation.md). Not the cost-circuit pattern (sibling [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Alerting for AI systems shares 60% of its discipline with general SRE alerting (page on user impact, calibrate to alert-fatigue thresholds, runbook-per-alert) and adds 40% that is specific (cost-spike alerts, quality-drift alerts, judge-pass-rate alerts, agent-loop-runaway alerts). Teams that adopt the general SRE alerting canon without the AI-specific overlays end up paging on latency / availability but missing the cost incidents and quality regressions that are the dominant AI failure modes.

This document is the discipline for that overlay. It is not a list of every possible alert; it is the framework for deciding which signals to alert on, at what threshold, who to page, and what runbook to attach. The framework is calibrated to the workload's tolerance for noise and the team's response capacity.

This document is opinionated about three things:

1. **Page on user impact or imminent user impact.** Cost spikes that will exhaust the daily budget in an hour are user impact (the feature will become unavailable). Quality drops below SLO are user impact (users get worse answers). Latency rises above tolerance is user impact. Alerts that do not predict user impact go to dashboards or to lower-tier notification channels.
2. **Alert fatigue is a quality issue.** If alerts fire that the team routinely ignores, alerts have stopped being signal. Calibrate ruthlessly; tune thresholds based on observed false-positive rates; retire alerts that do not predict action.
3. **Every alert points to a runbook.** Untitled paging alerts produce confused on-call rotations. The runbook is part of the alert; new alerts cannot land without one.

Structure: (2) the alert hierarchy; (3) the AI-specific signal catalog; (4) threshold calibration; (5) runbook design; (6) alert review discipline; (7) integration with broader SRE; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The alert hierarchy

Three tiers. Each tier has a defined response.

### 2.1 Tier 1: Paging alerts

- The signal indicates user impact in progress or imminent.
- On-call is paged immediately (PagerDuty / Opsgenie / similar).
- Response SLO: acknowledge within 5-15 minutes (per the workload's tier).
- Each alert has a runbook with diagnosis and response steps.

Examples for AI workloads:
- Cost circuit-breaker tripped at per-tenant or per-feature level.
- Production quality SLI breached (judge-pass-rate drop > 8 points).
- Agent-loop runaway rate above baseline.
- Provider outage detected.
- Scope-violation event (cross-tenant data exposure).

### 2.2 Tier 2: Alerting (no page)

- The signal indicates a degraded condition that needs attention but is not user-impacting yet.
- Notification to a channel (Slack channel, email distribution list) without paging.
- Response SLO: triaged within next-business-day; addressed within current sprint.
- Each alert has a documented expected response.

Examples:
- Cost budget at 80% threshold (warning before circuit).
- Quality SLI at warning level (judge-pass-rate drop 3-7 points).
- Empty-retrieval rate elevated.
- Cache hit rate dropped meaningfully.
- Eval-gate failure on a non-blocking PR.

### 2.3 Tier 3: Dashboard-only

- The signal is informational; trends and context matter more than immediate action.
- Visible on dashboards; no notification.
- Reviewed periodically (weekly, monthly) as part of team operational rhythm.

Examples:
- Per-feature daily cost trends.
- Per-tenant usage volume.
- Per-prompt-version production usage share.
- Eval pass rates over time.
- Trace-volume trends.

### 2.4 The decision: which tier?

For a candidate signal, ask:

1. **Does breach predict user-visible impact within an actionable window (minutes to hours)?** If yes → Tier 1.
2. **Does breach predict a problem that needs attention this week?** If yes → Tier 2.
3. **Is the signal useful for trends and context but not immediately actionable?** → Tier 3.

Misclassifying signals is the most common alert-design mistake. Tier 1 alerts that are actually Tier 2 cause alert fatigue. Tier 2 alerts that are actually Tier 1 cause missed incidents.

---

## 3. The AI-specific signal catalog

Signals to consider for each tier. Each has a default tier recommendation; adjust per workload.

### 3.1 Cost signals

| Signal | Default tier | Trigger |
|---|---|---|
| Per-interaction cost circuit-breaker tripped | Tier 1 | Single interaction exceeded per-interaction limit |
| Per-tenant daily cost circuit-breaker tripped | Tier 1 | Tenant exceeded daily limit |
| Per-feature daily cost circuit-breaker tripped | Tier 1 | Feature exceeded daily limit platform-wide |
| Cost-budget at 80% of daily limit | Tier 2 | Early warning before circuit-breaker |
| Cost trend (week-over-week growth > 30%) | Tier 2 | Trending toward circuit |
| Per-feature cost by hour of day | Tier 3 | Trend visibility |

Cost is high-stakes for AI workloads. The Tier 1 alerts here are the difference between "we caught it in an hour" and "we discovered the cost incident at month-end invoice reconciliation."

### 3.2 Quality signals

| Signal | Default tier | Trigger |
|---|---|---|
| Production judge-pass-rate drop > 8 points (4-hour window) | Tier 1 | Quality SLO breach |
| Judge-pass-rate drop 3-7 points | Tier 2 | Early warning |
| Per-feature judge-pass-rate trend | Tier 3 | Trend visibility |
| Eval-gate failure on PR | Tier 2 (the PR check fails) | Quality regression caught pre-prod |
| Eval-gate failure on release candidate | Tier 1 | Release-blocking quality issue |
| Hallucination rate above baseline (sampled judge) | Tier 1 | User-impacting quality failure |
| Citation accuracy below threshold (regulated workloads) | Tier 1 | Audit-relevant quality failure |
| Refusal rate above baseline | Tier 2 | May indicate prompt or model issue |

Quality alerts depend on the eval and online-judge infrastructure being in place per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md). Without that infrastructure, these signals cannot be measured.

### 3.3 Latency signals

| Signal | Default tier | Trigger |
|---|---|---|
| P95 TTFT above SLO target | Tier 1 | User-visible slowness in interactive features |
| P95 total latency above SLO | Tier 1 | User-visible slowness |
| P99 latency drift (week-over-week) | Tier 2 | Trending toward SLO breach |
| Latency per feature trend | Tier 3 | Trend visibility |
| Stream-idle-timeout rate elevated | Tier 2 | Quality of streaming experience |

### 3.4 Availability signals

| Signal | Default tier | Trigger |
|---|---|---|
| Provider error rate above 5% for 5+ minutes | Tier 1 | Provider degradation |
| Fallback rate above baseline | Tier 1 | Indicates upstream issue |
| Per-feature error rate above SLO | Tier 1 | User-impacting failures |
| Circuit-breaker on tools tripping frequently | Tier 2 | Tool reliability issue |

### 3.5 Agent-specific signals

| Signal | Default tier | Trigger |
|---|---|---|
| Agent-loop runaway rate above baseline | Tier 1 | Agents getting stuck in loops |
| Agent average turn count drifting up | Tier 2 | Possible quality regression |
| Agent termination-by-budget rate above baseline | Tier 2 | Agents failing to complete |
| Tool-authorization-rejected rate elevated | Tier 1 if sustained | Possible attack or misconfiguration |
| Per-agent worker quality variance | Tier 3 | Trend / diagnostic |

### 3.6 Retrieval signals

| Signal | Default tier | Trigger |
|---|---|---|
| Empty-retrieval rate above baseline | Tier 1 | Retrieval not finding expected content |
| Scope-violation event (cross-tenant) | Tier 1 | Sev-1 isolation failure |
| Embedding-API error rate elevated | Tier 1 | Upstream provider issue |
| Reranker error rate elevated | Tier 2 | Component issue |
| Per-corpus retrieval latency trend | Tier 3 | Diagnostic |

### 3.7 Security signals

| Signal | Default tier | Trigger |
|---|---|---|
| Prompt-injection detection elevated | Tier 1 if sustained | Possible attack |
| Authorization-failure rate spike | Tier 1 | Possible attack |
| Content-policy-refusal rate spike | Tier 2 | Possible misuse |
| Per-tenant authorization-failure trend | Tier 3 | Diagnostic |

Security signals are jointly owned with security-eng; on-call rotation may include security as a stakeholder.

### 3.8 Infrastructure signals

The standard SRE signals (CPU, memory, disk, network, database latency) apply to AI workloads with no special adaptation. Use the team's existing infrastructure alerting; these AI-specific overlays sit on top.

---

## 4. Threshold calibration

Setting thresholds correctly is harder than setting alerts. The discipline:

### 4.1 Derive thresholds from production data

For any signal, instrument and observe baseline for 2-4 weeks before setting alert thresholds. The baseline tells you:

- The signal's typical value.
- The signal's typical variability (p95, p99 spreads).
- The signal's seasonality (hour-of-day, day-of-week patterns).

Threshold then: typical value + N standard deviations, OR typical value × M (factor), OR absolute floor / ceiling.

The wrong pattern: set the threshold based on intuition before seeing data. The intuition is usually wrong; alerts fire too often or too rarely; calibration happens reactively after alert-fatigue or missed incidents.

### 4.2 Seasonality

Many AI signals are seasonal — daytime higher than night; weekday higher than weekend; quarter-end spikes for some workloads. The threshold logic accommodates this:

- Compare against same-hour-of-week baseline rather than overall baseline.
- Or set thresholds wide enough to accommodate the full seasonality without alerting on normal patterns.

For Meridian Care Coordinator: clinical workloads peak during morning rounds (8-11am local) and afternoon clinics (1-5pm local). Per-tenant cost thresholds use same-hour-of-day comparisons; cost spikes during typical peaks are not alarming, spikes outside typical peaks are.

### 4.3 The fatigue calibration

After deploying an alert, monitor:

- **False positive rate.** What fraction of fires turn out to be non-actionable?
- **Mean time to acknowledge.** Is the team responding promptly, or has fatigue set in?
- **Mean time to resolution.** Are alerts producing actionable response?

If false positive rate is > 30%, the alert is too noisy. If mean time to acknowledge is > 30 minutes consistently, the team is fatigued (likely because of upstream noise from other alerts).

Calibrate quarterly. Retire alerts that consistently fail the calibration.

### 4.4 The two-tier threshold

For many signals, two thresholds are useful:

- **Warning threshold.** Tier 2 notification.
- **Critical threshold.** Tier 1 page.

The warning gives time to investigate before the critical fires. For cost budgets: warning at 80%, critical at 100% of the budget.

### 4.5 Hysteresis

To prevent alerts from flapping (firing repeatedly as the signal oscillates around the threshold), use hysteresis:

- Alert fires when signal > threshold + buffer.
- Alert resolves when signal < threshold - buffer.
- The buffer prevents oscillation noise.

---

## 5. Runbook design

Every paging alert has a runbook. The runbook is what makes the alert actionable rather than informational.

### 5.1 The runbook structure

Each runbook includes:

1. **Trigger.** What this alert means (the underlying signal).
2. **Impact.** What user impact is happening or imminent.
3. **Diagnostic steps.** Specific things to check (specific dashboards, traces, queries).
4. **Mitigation steps.** Actions to take (rollback, throttle, scale, kill switch).
5. **Escalation path.** Who to involve if the on-call cannot resolve.
6. **Post-incident.** What to do after (incident review, follow-up actions).

The runbook is concise (typically 1-2 pages); each step is specific enough to follow under pressure.

### 5.2 Examples

**Cost-circuit-breaker (per-feature) tripped.**

- Trigger: Per-feature daily cost limit exceeded.
- Impact: The feature is throttled / serving degraded mode for all tenants.
- Diagnostic:
  - Open cost dashboard; identify the feature.
  - Inspect cost trend; identify if it's a sudden spike or accumulation.
  - Check recent deploys (prompt-version, model-version changes today?).
  - Check per-tenant breakdown; is one tenant driving the spike?
- Mitigation:
  - If recent deploy is the cause: roll back per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
  - If one tenant is the cause: rate-limit that tenant; contact customer-success.
  - If genuine traffic growth: raise the budget temporarily (per pre-approved limits) and capacity-plan.
  - Lift the throttle once the cause is addressed.
- Escalation: ai-platform-eng lead + sre on-call + (if customer-impacting) customer-success.
- Post-incident: cost-incident review per [cost-incident-runbook.md](../cost-and-finops/) (coming).

**Production judge-pass-rate drop > 8 points.**

- Trigger: Online judge sampling shows pass rate dropped > 8 points from trailing baseline.
- Impact: Users are receiving worse-quality answers right now.
- Diagnostic:
  - Open quality dashboard; identify the feature.
  - Inspect per-case-class breakdown; which class(es) regressed?
  - Check recent deploys (prompt-version, model-version, corpus changes today?).
  - Sample failing interaction traces; identify the failure pattern.
- Mitigation:
  - If recent prompt change: roll back per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md).
  - If recent model change: roll back per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md).
  - If corpus change: investigate per [data-contracts-for-retrieval.md](../../ai-architecture-reference-architecture/data-architecture-for-ai/data-contracts-for-retrieval.md).
- Escalation: ai-platform-eng lead + prompt-owner team.
- Post-incident: regression case added to eval suite; eval-gate calibration reviewed.

### 5.3 Runbook maintenance

Runbooks decay. Software evolves; the diagnostic and mitigation steps need to keep up. The discipline:

- Quarterly review of every runbook.
- Update on incident if the runbook proved insufficient.
- Owner per runbook (often the team that owns the underlying signal).

---

## 6. Alert review discipline

The alerting practice is itself a system that needs operational discipline.

### 6.1 Weekly review

The on-call rotation's weekly hand-off includes:

- Which alerts fired this week?
- Which were actionable? Which were noise?
- Any incidents that should have alerted but did not?
- Any runbooks that need updating based on this week's incidents?

The hand-off keeps alert quality high.

### 6.2 Quarterly calibration

Every quarter:

- Review every alert; calibrate thresholds against observed baseline.
- Retire alerts with sustained false-positive rates > 30%.
- Promote / demote alerts based on observed user-impact correlation.
- Audit runbooks for currency.

### 6.3 Incident post-mortems

After every Sev-1 / Sev-2 incident, the post-mortem includes alert analysis:

- Did the right alerts fire?
- Did they fire at the right time (early enough to prevent escalation)?
- Did the runbook help, hinder, or get in the way?
- What alerts should have existed but did not?

Findings from post-mortems update the alert hierarchy.

### 6.4 The alert review owner

Someone owns the alert hierarchy. For Meridian: the ai-platform-eng team lead is responsible for the AI-specific alerts; sre owns the broader infrastructure alerting. They coordinate quarterly.

---

## 7. Integration with broader SRE

AI alerts coexist with the broader SRE alerting; the integration matters.

### 7.1 The on-call rotation

For most teams, AI alerts go through the same on-call rotation as the rest of the platform. The on-call engineer responds to anything that pages.

If AI alerts have specialized response needs (clinical-domain expertise, prompt-engineering response), the rotation may include AI-specialized on-call as a secondary tier (the on-call engineer handles routine; specialized engineers handle specific classes).

### 7.2 The shared dashboards

AI signals appear on the same dashboards as general application signals. The investigator can see "the AI feature is slow" alongside "the database is slow" and correlate.

The Meridian Care Coordinator's overview dashboard shows: AI feature latency, AI feature cost rate, AI feature error rate, database connection pool, model provider latency, web server latency, request rate. Cross-system correlation is one dashboard away.

### 7.3 The shared post-mortem framework

AI incidents go through the same post-mortem process as other incidents. Same template; same review cadence; same blameless framing. The AI-specific aspects (cost incidents, quality regressions) get treated as legitimate operational concerns rather than special cases.

### 7.4 Where AI is different

AI workloads have failure modes the broader SRE practice may not be prepared for:

- **Quality degradation without availability impact.** The system is up; responses are technically returning; the quality is just bad. SRE alerting often misses this.
- **Cost runaway from a single bad call.** A misbehaving agent can burn meaningful budget in minutes. SRE alerting on infrastructure cost is monthly; this needs per-minute response.
- **Silent provider-side changes.** A model alias resolving to a new version changes behavior without any platform deploy. SRE alerting on deploys won't catch this.
- **Cross-tenant data exposure.** A bug in scope enforcement leaks data without crashing anything. SRE alerting on errors won't catch this.

The AI-specific alerts (per section 3) fill these gaps.

---

## 8. Worked Meridian Care Coordinator example

### 8.1 The Tier 1 alerts in production

| Alert | Threshold | Runbook |
|---|---|---|
| Care Coordinator chat cost-circuit-breaker (per-feature) | $1,500/day breached | cost-incident-runbook |
| Per-tenant cost-circuit-breaker | $50/day (standard) or $200/day (premium) breached | tenant-cost-runbook |
| Production judge-pass-rate drop > 8 points | 4-hour window | quality-regression-runbook |
| Production judge-pass-rate drop > 5 points on drug-interaction subset | 4-hour window | clinical-quality-runbook |
| Agent-loop runaway rate > 1% | 1-hour window | agent-runaway-runbook |
| Provider error rate > 5% for 5+ minutes | sustained | provider-outage-runbook |
| Scope-violation event | any | sev-1-isolation-runbook |
| Tool-authorization-failure spike (> 3 sigma above baseline) | 15-minute window | possible-attack-runbook |

Each runbook is ~1 page; on-call rehearsal includes walking through 2-3 per quarter.

### 8.2 The Tier 2 alerts

| Alert | Threshold | Action |
|---|---|---|
| Cost-budget at 80% | warning | Slack notification; investigate within day |
| Judge-pass-rate drop 3-7 points | warning | Slack notification; investigate within day |
| Empty-retrieval rate > 5% (vs ~2% baseline) | sustained 1 hour | Slack; investigate cause |
| Cache hit rate dropped > 15 points | warning | Slack; check if recent prompt change |
| Agent average turn count drift up | week-over-week | Slack; review |
| Eval-gate failure on non-blocking PR | always | Slack; PR author responds |

### 8.3 The Tier 3 dashboards

Daily-reviewed dashboards:
- Per-feature cost trend (with budgets overlaid).
- Per-feature quality SLI (judge-pass-rate trend).
- Per-feature latency (TTFT, total).
- Per-tenant cost ranking.
- Cross-tenant retrieval-empty-rate trends.
- Prompt-version usage share (canary visibility).

### 8.4 The on-call rotation

ai-platform-eng has a 5-engineer rotation. Each shift is 12 hours weekday / 24 hours weekend. The shift's primary responsibility is responding to Tier 1 AI alerts; secondary is shadowing the general SRE on-call for cross-system incidents.

For specialized incidents (clinical quality, security-flagged), the rotation includes call-out paths to clinical-knowledge-eng or security-eng.

### 8.5 An incident walkthrough

The 2026-04-29 cost incident (referenced in [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)):

- 4:00 PM UTC: cost-spike alert (Tier 2 warning) fires — care-coordinator cost at 80% of daily budget unusually early in the day.
- 4:02 PM: on-call acknowledges; opens cost dashboard.
- 4:05 PM: identifies cost trajectory will hit 100% by ~5 PM.
- 4:08 PM: cost-circuit-breaker (Tier 1) fires; feature throttles to degraded mode.
- 4:09 PM: on-call's investigation in parallel — checks recent deploys (none today); checks per-tenant breakdown (cost is uniform across tenants, not single-tenant abuse); checks prompt-version (no change today); checks model-version (a new claude-opus-4-7 version was auto-resolved at 3:30 PM today via the `claude-opus-latest` alias).
- 4:15 PM: root cause identified — alias resolved to new version with different pricing.
- 4:20 PM: model pin updated to the previous specific version; feature un-throttled.
- 4:30 PM: cost normalized.

The combination of Tier 2 (warning) and Tier 1 (circuit) alerts produced the right response pattern: warning gave time to start investigation; circuit prevented runaway while investigation completed.

### 8.6 The platform discipline

- Every alert has a runbook; no alert lands without one.
- Quarterly calibration; alerts retired if false-positive > 30%.
- Weekly on-call hand-off includes alert review.
- Post-mortems include alert analysis.

---

## 9. Anti-patterns

### 9.1 "Page on everything"

The team set up alerts liberally; everything that looks like a problem pages on-call. Alert fatigue sets in within weeks; real incidents are missed because the on-call has stopped reading the alerts.

**Corrective.** Tier discipline. Most signals are Tier 2 or Tier 3; only user-impacting / imminently-user-impacting signals page.

### 9.2 "No runbook"

Pages fire without runbooks. On-call engineers diagnose from scratch every time. Time to resolution is high; quality of response varies by engineer.

**Corrective.** Runbook is part of the alert. New alerts without runbooks are not deployed.

### 9.3 "Threshold from intuition"

Thresholds are set by guess. Some are too tight (alert noise); some are too loose (missed incidents). Calibration is reactive.

**Corrective.** Thresholds derived from 2-4 weeks of observed data. Re-calibrate quarterly.

### 9.4 "Quality alerts not implemented"

The team has latency / availability alerts but no quality SLI alerts. Quality regressions ship; users notice; the team learns from user complaints.

**Corrective.** Judge-pass-rate as an SLI; quality alerts per section 3.2.

### 9.5 "Cost alerts arrive at month-end"

Cost monitoring is the monthly cloud invoice. Cost incidents are detected after the fact.

**Corrective.** Per-call cost telemetry per [llm-call-instrumentation.md](./llm-call-instrumentation.md); per-feature / per-tenant cost circuit breakers per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md); Tier 1 alerts on circuit-breaker trips.

### 9.6 "No alert review discipline"

Alerts accumulate; nobody reviews their effectiveness; stale alerts persist long after their relevance.

**Corrective.** Weekly hand-off review; quarterly calibration; ownership per alert.

### 9.7 "AI on-call isolated from broader on-call"

AI alerts go to a separate rotation that does not coordinate with the general SRE on-call. Cross-system incidents have communication gaps.

**Corrective.** Shared rotation or strong coordination patterns. Shared dashboards. Cross-system incidents have unified response.

### 9.8 "Severity inflation"

Every alert is Sev-1 because everything feels important. Sev-1 loses meaning; response prioritization breaks down.

**Corrective.** Sev-1 is reserved for true user-impact emergencies. Tier 2 alerts are not Sev-1.

---

## 10. Findings (sprint-assignable)

### ALERT-001 — Severity: Critical
**Finding.** AI quality SLI alerts are not implemented; quality regressions ship and are discovered by users.
**Recommendation.** Implement online judge sampling per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) section 7.3; surface as SLI; Tier 1 alerts on breach.
**Owner.** ai-platform-eng + observability-eng, sprint N+1.

### ALERT-002 — Severity: Critical
**Finding.** Cost alerts arrive at month-end via cloud invoice; in-the-moment cost incidents are not detected.
**Recommendation.** Per-call cost telemetry per [llm-call-instrumentation.md](./llm-call-instrumentation.md); cost-circuit-breaker alerts per Tier 1.
**Owner.** ai-platform-eng + finops, sprint N+1.

### ALERT-003 — Severity: Critical
**Finding.** Paging alerts do not have runbooks; on-call diagnosis is ad-hoc.
**Recommendation.** Every Tier 1 alert has a runbook per section 5; no alert deploys without one.
**Owner.** ai-platform-eng + sre, sprint N+1.

### ALERT-004 — Severity: High
**Finding.** Thresholds were set by intuition; false-positive rate is high or unknown.
**Recommendation.** Derive from 2-4 weeks of observed data per section 4.1; calibrate quarterly.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### ALERT-005 — Severity: High
**Finding.** Scope-violation events (cross-tenant data exposure) do not have Tier 1 alerts; isolation failures may be undetected.
**Recommendation.** Tier 1 alert on every scope-violation event per [retrieval-scope-enforcement.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/retrieval-scope-enforcement.md).
**Owner.** ai-platform-eng + security-eng, sprint N+1.

### ALERT-006 — Severity: High
**Finding.** Agent-loop runaway alerts do not exist; agent cost overruns are detected via cost alerts only.
**Recommendation.** Agent-loop runaway as separate Tier 1 alert per section 3.5.
**Owner.** ai-platform-eng, sprint N+2.

### ALERT-007 — Severity: High
**Finding.** Alert review discipline is informal; alert effectiveness is not tracked.
**Recommendation.** Weekly hand-off includes alert review per section 6.1; quarterly calibration per section 6.2.
**Owner.** ai-platform-eng team lead + sre, sprint N+2.

### ALERT-008 — Severity: High
**Finding.** Two-tier thresholds (warning + critical) are not used; alerts either fire too late or too early.
**Recommendation.** Warning + critical thresholds per section 4.4; warning gives investigation time.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### ALERT-009 — Severity: Medium
**Finding.** Seasonality is not accommodated in thresholds; daytime peaks trigger false alerts.
**Recommendation.** Same-hour-of-week baseline comparison for seasonal signals per section 4.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ALERT-010 — Severity: Medium
**Finding.** AI on-call rotation is isolated from broader SRE; cross-system incidents have communication gaps.
**Recommendation.** Shared rotation or strong coordination per section 7.1; shared dashboards.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ALERT-011 — Severity: Medium
**Finding.** Runbooks are not maintained; software has evolved but diagnostic / mitigation steps are stale.
**Recommendation.** Quarterly runbook review per section 5.3; owner per runbook.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ALERT-012 — Severity: Medium
**Finding.** Post-mortems do not include alert analysis; alert-design improvements are missed.
**Recommendation.** Add alert analysis to post-mortem template per section 6.3.
**Owner.** ai-platform-eng + sre, sprint N+3.

### ALERT-013 — Severity: Medium
**Finding.** Per-tenant cost alerts are uniform; premium-tier tenants generate same-threshold alerts as standard tenants.
**Recommendation.** Per-tier alert thresholds calibrated to the contracted usage.
**Owner.** ai-platform-eng + customer-success, sprint N+3.

### ALERT-014 — Severity: Medium
**Finding.** Empty-retrieval rate is not alerted; quality issues from retrieval gaps are undetected.
**Recommendation.** Empty-retrieval-rate alert per section 3.6.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### ALERT-015 — Severity: Medium
**Finding.** Provider-outage detection is reactive (the team learns from user complaints).
**Recommendation.** Provider error-rate alerting per section 3.4; fallback rate as a derived signal.
**Owner.** ai-platform-eng, sprint N+3.

### ALERT-016 — Severity: Low
**Finding.** Hysteresis is not configured; some alerts oscillate around the threshold.
**Recommendation.** Buffer for fire / resolve thresholds per section 4.5.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### ALERT-017 — Severity: Low
**Finding.** Tier-3 dashboards are not regularly reviewed; trends that should inform alert thresholds are missed.
**Recommendation.** Weekly dashboard review as part of operational rhythm.
**Owner.** ai-platform-eng team lead, sprint N+4.

### ALERT-018 — Severity: Low
**Finding.** Alert configuration is hardcoded; threshold changes require deploys.
**Recommendation.** Alert configuration as runtime config; threshold changes apply without deploy.
**Owner.** ai-platform-eng + sre, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team building AI alerting from scratch:

- [ ] **Sprint 0 — design.** Review the signal catalog (section 3); decide which signals are needed for the workload. Define tier per signal.
- [ ] **Sprint 1 — instrumentation foundation.** Confirm signals are emitted (per the per-call / per-trace docs); if not, instrument first.
- [ ] **Sprint 1 — baseline observation.** Deploy the signals to dashboards (Tier 3). Observe baseline for 2-4 weeks.
- [ ] **Sprint 2 — first Tier 1 alerts.** Implement cost-circuit-breaker alerts and quality-SLI alerts first (most-leveraged). Calibrate thresholds from baseline data.
- [ ] **Sprint 2 — runbooks.** Every Tier 1 alert has a runbook per section 5.
- [ ] **Sprint 3 — additional Tier 1 alerts.** Add agent-runaway, scope-violation, provider-outage alerts.
- [ ] **Sprint 3 — Tier 2 warnings.** Two-tier thresholds for high-stakes signals.
- [ ] **Sprint 4 — process.** Weekly hand-off review; quarterly calibration; post-mortem template.
- [ ] **Sprint 5 — refinement.** Hysteresis; seasonality; per-tenant calibration.
- [ ] **Ongoing — discipline.** Quarterly calibration; runbook maintenance; post-mortem integration.

A team that completes this sequence has an AI alerting discipline that produces signal proportional to action. A team that defaults to "alert on everything" or "alert on nothing" pays in either fatigue or missed incidents.

---

## 12. References

- Google SRE Book chapters on monitoring distributed systems, alerting philosophy.
- PagerDuty / Opsgenie / VictorOps documentation on alert routing.
- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — the signal source.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — LLM-call signals.
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](./agent-step-instrumentation.md) — agent-loop signals.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](./retrieval-instrumentation.md) — retrieval signals.
- This repo: [observability-and-telemetry/quality-drift-detection.md](./) (coming) — quality SLI source.
- This repo: [observability-and-telemetry/cost-dashboards.md](./) (coming) — cost dashboard patterns.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost alerts integration.
- This repo: [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/) (coming) — cost-incident runbook.
- This repo: [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/) (coming) — broader incident response.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
