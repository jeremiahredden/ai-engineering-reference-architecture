# Cost Dashboards and Alerts

> **Audience.** Engineers building cost observability for AI workloads. SREs on-call for cost-spike pages. FinOps partners who use the dashboards in monthly reviews. Anyone whose answer to "how much did Care Coordinator cost yesterday" is "let me run a query." **Scope.** The *engineering* practice of building cost dashboards that support action, alerts that page the right people on the right signal, and the runbook integration that converts an alert into a resolved incident. Per-feature, per-tenant, per-user, per-model breakdowns; burn-rate vs spike-vs-trend signal design; alert thresholds; integration with the broader observability stack. Not the per-call attribution that feeds the dashboards (see [cost-attribution.md](./cost-attribution.md)). Not the circuit-breaker that fires on threshold (see [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md)). Not the runbook itself (see [cost-incident-runbook.md](./cost-incident-runbook.md) *(coming)*). **Worked client.** Meridian Health.

---

## 1. Why this document exists

A cost dashboard that nobody looks at and an alert that pages everyone for everything are equally useless. Cost observability has a specific failure mode: the data is available but the team can't act on it. Either the dashboard is a wall of numbers without a story; or the alerts fire constantly and get suppressed; or the alerts fire silently and nobody notices until the invoice arrives.

The patterns that produce *actionable* cost observability are different from the patterns that produce *correct* cost telemetry. Correct telemetry is the prerequisite (see [cost-attribution.md](./cost-attribution.md)). Actionable observability is what's built on top: dashboards that answer specific questions, alerts that route to specific runbooks, and the discipline that keeps both up to date as the system evolves.

Specifically:

- A "total AI cost" dashboard is not action-bearing. It shows the trend; it doesn't tell anyone what to do.
- A "cost per feature" dashboard surfaces which feature is the dominant contributor and whether the contribution is growing or shrinking. It enables conversations like "why did Care Coordinator's cost double in two weeks?"
- A "cost per tenant" dashboard catches noisy tenants and abuse. It enables conversations like "tenant X just consumed $400 in a day; was it expected?"
- A "cost burn rate" alert (e.g., "this feature is on track to exceed its monthly budget by day 18") gives the team time to react. A "cost spike" alert (e.g., "this feature's hourly spend just tripled") catches incidents in real time.

This document covers the dashboard set, the alert set, the thresholds, and the runbook integration that make cost observable in a way that supports response.

This document is opinionated about four things:

1. **Dashboards are designed for specific questions.** "Show me everything" is not a dashboard; it's a data dump. Each dashboard answers a specific question (which feature is most expensive, which tenant is anomalous, which model is over-routed). The question shapes the visualization.
2. **Alerts must page a specific runbook.** An alert that fires and triggers ad-hoc investigation produces slow response and inconsistent outcomes. Each alert has a documented runbook; the alert references it; on-call follows it.
3. **Burn-rate beats absolute-threshold for budget alerts.** "You will exceed your budget on day 22" is more actionable than "you've used 60% of your budget" — the former implies action; the latter is just status.
4. **The dashboards and alerts must evolve with the system.** New features need their own panels; deprecated features should be hidden; new alert classes emerge from incident reviews. The observability stack is itself a maintained system, not a one-time build.

Structure: (2) the dashboard set; (3) burn-rate alerts; (4) spike alerts; (5) anomaly detection alerts; (6) the alert routing model; (7) the runbook integration; (8) tracking dashboard and alert effectiveness; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The dashboard set

The minimum useful dashboard set has six panels. Each answers a specific question.

### 2.1 Per-feature cost trend (the "where is the money" question)

A stacked-area chart of cost by feature over time (daily for 90 days; hourly for 24 hours).

**What it shows.** Which features are the dominant cost contributors; whether each feature's contribution is growing, stable, or shrinking.

**What it surfaces.** Cost shifts (a feature that suddenly costs more), launch impact (a new feature's cost trajectory), seasonality (weekday vs weekend patterns).

**Typical row count.** 5-15 features. More than 15 becomes hard to read; consolidate small features into "other."

**Action.** When the chart shows a feature growing unexpectedly, that's the feature to investigate.

### 2.2 Per-tenant cost ranking (the "noisy neighbor" question)

A horizontal bar chart of cost per tenant over a window (last 24 hours, last 7 days), sorted by cost descending.

**What it shows.** Which tenants are the dominant cost contributors; whether the distribution is even or long-tailed.

**What it surfaces.** A free-tier tenant that suddenly appears in the top 10 (potential runaway or abuse); a premium tenant whose cost grew (organic growth or misconfiguration); a tenant whose cost is below expected (potential outage or churn).

**Typical row count.** Top 20 tenants displayed; full list available via filter.

**Action.** Top tenants outside their expected range warrant investigation.

### 2.3 Per-user cost distribution (the "outlier user" question)

A histogram of cost per user over a window (last 30 days).

**What it shows.** The distribution shape; how concentrated cost is among power users; whether outlier users exist.

**What it surfaces.** A single user generating $500/day in a system designed for $5/day per user.

**Typical layout.** Histogram with log-scale x-axis (cost) and linear y-axis (user count).

**Action.** Outliers in the long tail warrant per-user investigation.

### 2.4 Per-model cost breakdown (the "tier routing" question)

A pie or stacked chart of cost by model (Sonnet, Haiku, Opus, self-hosted, embedding) over a window.

**What it shows.** Cost distribution across the model portfolio; whether the routing strategy is working.

**What it surfaces.** Too much Sonnet usage (tier routing not working); too much Opus usage (premium escalation too aggressive); embedding cost dominance (retrieval pattern review needed).

**Action.** Skew toward expensive models triggers tier-routing review (see [tier-routing-for-cost.md](./tier-routing-for-cost.md)).

### 2.5 Per-prompt-version cost breakdown (the "prompt change impact" question)

A line chart of cost-per-call by prompt version over a window.

**What it shows.** Whether a recent prompt change increased or decreased per-call cost.

**What it surfaces.** A prompt update that ballooned token usage (bloated prompt; missing trim; verbose new instructions); a prompt update that successfully tightened cost.

**Action.** Prompt changes that increase cost-per-call are not necessarily wrong (quality may have improved), but they should be intentional. Unintentional cost increases are bugs.

### 2.6 Cost vs. usage cohort (the "is this growth normal" question)

A scatter or line chart of cost per call vs call volume.

**What it shows.** Whether cost is growing in proportion to usage (expected) or faster than usage (concerning).

**What it surfaces.** A feature whose call volume grew 10% but cost grew 80% — disproportionate growth suggests a per-call cost increase (prompt bloat, model upgrade, context growth).

**Action.** Disproportionate growth triggers per-call investigation.

### 2.7 The "build for action, not display" discipline

Each panel must have:

- A clear question it answers.
- A normal-range annotation (what's "expected" looks like).
- A click-through to investigate (drill into the data).
- A link to the relevant runbook for anomalies.

Panels that don't meet these criteria are deleted. A dashboard with 30 unloved panels is worse than one with 6 well-maintained panels.

### 2.8 The customer-facing variant

For multi-tenant platforms, customers expect to see their own cost. The customer-facing dashboard is a subset:

- Their cost over time.
- Their cost by feature (within their account).
- Their cost by user (within their account).
- Their budget consumption vs limit.

PII / cross-tenant data is excluded. Cross-link to [per-tenant-cost-control.md](./per-tenant-cost-control.md).

---

## 3. Burn-rate alerts

Burn-rate alerts fire when a budget is on track to be exceeded before the period ends. They give the team time to react.

### 3.1 The burn-rate formula

```
burn_rate = (spend_so_far / period_elapsed) / (budget / period_total)
```

A burn rate of 1.0 means "exactly on track." Greater than 1.0 means "spending faster than budget."

For a monthly budget:

- Day 10 of 30. Spent: 40% of budget. Burn rate = (0.40 / 10/30) = 1.20 → 20% over track.
- Day 20 of 30. Spent: 70% of budget. Burn rate = (0.70 / 20/30) = 1.05 → 5% over track.

### 3.2 The two-threshold pattern

Two alert thresholds:

**Warning (burn rate > 1.5):** spending 50% faster than budget. Notification (Slack, email) to the team that owns the budget. No paging.

**Critical (burn rate > 2.0):** spending 100% faster than budget. Pages on-call. Triggers the cost-incident runbook.

The thresholds are tunable; 1.5 / 2.0 is a typical default.

### 3.3 The "fast-burn" variant

Burn-rate over short windows catches sudden spikes:

- 1-hour burn rate: compares hourly spend to expected hourly average.
- 6-hour burn rate: smoother; catches sustained anomalies.

A 1-hour burn rate of 10.0 is more concerning than the same monthly burn rate; the underlying anomaly is more severe.

**Pattern.** Multi-window burn-rate alerts (from the Google SRE book applied to cost):
- 1-hour burn at 14x → page (sudden severe spike).
- 6-hour burn at 6x → page (sustained anomaly).
- 1-day burn at 3x → ticket (drift).
- 3-day burn at 2x → ticket (slower drift).

### 3.4 The cool-down behavior

After an alert fires and is acknowledged:

- Pause re-firing of the same alert for N hours (typical: 2-6 hours; depends on alert criticality).
- Do not pause silently; show "paused, expires at X" in the alert state.
- Re-fire if condition persists past pause.

Without cool-down, a sustained anomaly produces alert storms.

### 3.5 The burn-rate alert per scope

Burn-rate alerts at multiple scopes:

- Per-feature: each feature has its monthly budget; burn rate per feature.
- Per-tenant: each tenant has its budget; burn rate per tenant.
- Per-model: each model has its monthly budget (if you set one); burn rate per model.
- Aggregate: overall AI spend has a budget; burn rate against the aggregate.

Aggregate alerts catch what per-scope alerts miss (the case where many small drifts add up). Per-scope alerts catch what aggregate misses (the case where one scope is the cause).

---

## 4. Spike alerts

Spike alerts fire when spend rate increases suddenly, even if absolute spend is within budget.

### 4.1 The spike-vs-trend distinction

**Spike.** A sudden change in the rate. Example: hourly spend goes from $50 to $500.

**Trend.** A gradual change over time. Example: weekly spend grows 5% per week for 6 weeks.

Spikes need fast response (page); trends warrant review (ticket).

### 4.2 The spike-detection signal

A spike is defined as:

```
current_rate > baseline_rate × spike_factor
```

Where `spike_factor` is typically 3-10x.

The `baseline_rate` is the recent moving average (1-hour, 6-hour, or 24-hour window). Use the longer window to reduce false-positive sensitivity; use the shorter for fast catch.

### 4.3 The "weekend effect" trap

Many workloads have weekly patterns: Saturday is 30% of weekday traffic. Naive baselining produces false-positive spikes Monday morning ("traffic increased 3x! spike!") and false-positive lulls Saturday ("traffic decreased 70%! is there an outage?").

**Mitigation.** Baseline using "same time-of-day, same day-of-week" averaging. The current Monday morning's spend compares to last Monday morning's spend, not to Sunday afternoon's.

### 4.4 The "launch day" trap

A feature launch produces a legitimate cost spike. The alert fires; the team acknowledges; the alert fires again next time the launch hits a bigger market. Each acknowledgment is fatigue.

**Mitigation.** Launch announcements suppress alerts for the launched feature for a configured window. Alert system has a "feature launching" state.

### 4.5 The "we deployed a new prompt" spike

A prompt update can produce a legitimate cost-per-call increase. The alert fires; the team investigates; the answer is "the prompt change was intentional."

**Mitigation.** Cost-per-call alerts tagged with the deploying prompt version. If the spike correlates with a recent deploy, the alert message includes "this began at deploy X of feature Y at time Z." Investigation starts pre-informed.

---

## 5. Anomaly detection alerts

Beyond simple threshold and spike alerts, anomaly detection catches patterns that humans wouldn't write rules for.

### 5.1 What anomaly detection adds

Threshold alerts catch known patterns ("cost > X"). Spike alerts catch step changes. Anomaly detection catches:

- Distribution shifts (cost-per-call distribution skewed in an unusual direction).
- Correlated changes (cost up + latency up + error rate up — likely a model issue).
- Composite patterns (a normally-small tenant whose usage looks like a normally-large tenant).
- Outlier classification (this hour's cost looks like a 99th-percentile-anomalous hour).

### 5.2 The simple anomaly metric

A useful first-pass anomaly metric:

```
z_score = (current_value - mean_value) / stddev_value
```

Computed over a window (e.g., last 30 days, same hour). |z_score| > 3 is "very unusual."

**Pros.** Simple; explainable; no ML required.

**Cons.** Assumes Gaussian distribution; doesn't handle non-stationary patterns well.

### 5.3 The MAD-based anomaly metric

Median Absolute Deviation is more robust to outliers:

```
mad = median(|x_i - median(x)|)
robust_z = (current - median) / (1.4826 × mad)
```

Less sensitive to occasional large values affecting baseline.

### 5.4 The ML-based anomaly metric

For higher-fidelity detection, ML-based methods (isolation forest, autoencoders, Prophet for time-series) can catch subtler patterns.

**Caveat.** ML-based anomaly detection requires training data; produces false positives until tuned; adds operational complexity. Most teams don't need it.

**When justified.** High-volume, high-variability workloads where simple methods have unacceptable false-positive rate.

### 5.5 The "anomaly detected; now what" runbook

Anomaly alerts often don't tell the team what's wrong, only that something is unusual. The runbook must accept that and triage:

1. Confirm the anomaly is real (not a data quality issue).
2. Identify the dimension (per-feature? per-tenant? per-user?).
3. Investigate the dimension's recent context (deploys, incidents, customer onboards).
4. Decide whether to mitigate or accept.

Anomaly detection's value is in catching things the team wouldn't have looked for; it's not in giving answers.

---

## 6. The alert routing model

Alerts go to people; people respond. The routing must match alert to person.

### 6.1 The routing dimensions

For each alert:

- **Owner team.** Who owns the budget / feature / tenant affected.
- **Severity.** Page / ticket / notification.
- **Time of day.** Business hours vs after hours.
- **Customer-visible.** Does the customer need to be notified?

The routing decision uses these dimensions.

### 6.2 The per-feature routing

Each feature in production has an owner team. Cost alerts for that feature route to:

- The feature's on-call rotation (PagerDuty / Opsgenie / similar).
- The feature's Slack channel.
- The feature's email distribution list (for non-page alerts).

The feature → team mapping is maintained in a service catalogue (or the model catalogue from [model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md)).

### 6.3 The per-tenant routing

For tenant-level alerts:

- Customer-affecting: route to customer-success team + on-call.
- Customer-not-affecting (caught early): route to platform team only.
- Premium tenants: tighter SLO; faster response.

### 6.4 The escalation path

If the primary on-call doesn't acknowledge within N minutes:

- Escalate to secondary on-call.
- Escalate to engineering manager.
- Escalate to platform team lead.

Cost incidents rarely need this level of escalation, but the path must exist; cost-affecting incidents that go unacknowledged become bigger incidents.

### 6.5 The alert fatigue management

The architecture's responsibility is to keep alerts firing only when action is needed.

- Track per-alert false-positive rate.
- Alerts with > 50% false-positive rate are reviewed; tuned or removed.
- Alerts that never fire are reviewed; removed if no longer relevant.
- New alerts get a 2-week observation period before being made paging.

### 6.6 The "do not page on business hours noise" rule

Some patterns produce business-hours-only "noise" (e.g., morning startup spikes that are normal). Tag these alerts as ticket-not-page; let business hours handle them.

After-hours alerts are stricter; they should fire only on genuine emergencies.

---

## 7. The runbook integration

Every alert references a runbook. The runbook turns the alert into a resolved incident.

### 7.1 The runbook structure

A useful cost runbook has:

- **What this alert means.** The signal the alert is based on; what it suggests.
- **First-line check.** Quick verification the alert is real (not data quality).
- **Triage steps.** Which dashboard to open; which dimension to investigate first.
- **Common causes.** Top 3-5 root causes for this alert.
- **Mitigation steps.** What to do to stop the bleeding (rate-limit, route down-tier, kill switch).
- **Investigation.** How to find the root cause.
- **Resolution.** Once cause is known, how to fix.
- **Post-incident.** Update budgets, add new alerts, document.

### 7.2 The runbook as code

Runbooks live in the same repo as the alert configuration. When the alert is defined, the runbook is referenced; the alert's notification includes the runbook URL.

Changes to runbooks go through PR review.

### 7.3 The "no runbook = no production alert" rule

Alerts in production must have a runbook. Alerts being prototyped go to staging only.

### 7.4 The runbook validation

Periodically (quarterly), run an alert-runbook validation:

- Simulate the alert condition.
- Follow the runbook.
- Note any steps that are unclear, missing, or wrong.
- Update the runbook.

Drill discipline keeps runbooks current.

### 7.5 The "alert fires; runbook runs; incident is resolved" cycle

The cycle should be:

1. Alert fires.
2. On-call acknowledges (within minutes).
3. On-call opens the linked runbook.
4. Runbook leads to dashboards.
5. Dashboards reveal the cause.
6. Runbook prescribes mitigation.
7. Mitigation applied.
8. Incident review (next business day).

When this cycle breaks (alerts fire and nobody knows what to do), the runbook is failing its purpose. Improvement is in the runbook, not in adding more alerts.

---

## 8. Tracking dashboard and alert effectiveness

The observability stack is itself a system that needs observability.

### 8.1 The dashboard usage metric

Track which dashboards are accessed and by whom. Dashboards that get no traffic are deletion candidates (or the question they answer is no longer relevant).

Most observability platforms have built-in usage analytics.

### 8.2 The alert effectiveness metric

For each alert:

- **Fire rate.** How often does it fire per month?
- **Acknowledge rate.** Of fires, how many are acknowledged?
- **False-positive rate.** Of fires, how many were not real incidents?
- **Mean time to acknowledge (MTTA).** How fast does on-call respond?
- **Mean time to resolve (MTTR).** How long from fire to incident close?

Alerts with poor metrics get tuned or removed.

### 8.3 The "noisy alert" cleanup cycle

Monthly review:

- Alerts that fired > 20 times last month: tune up (reduce sensitivity) or split (separate paging from notification).
- Alerts that have > 70% false-positive rate: tune up or remove.
- Alerts that didn't fire all year: review if still needed.

### 8.4 The incident review of alert effectiveness

Post-incident, review:

- Did an alert fire? (If not, what would have caught this earlier?)
- Was the alert helpful? (Did it page the right person at the right time?)
- Was the runbook helpful? (Did it lead to resolution quickly?)

Each incident is an opportunity to improve the observability stack.

### 8.5 The "evolve over time" expectation

The dashboards and alerts you have today are not the ones you want next year. Add panels for new features; remove panels for deprecated features; refine alerts as patterns emerge.

Treat the observability stack as a maintained system, not a one-time configuration.

---

## 9. Worked Meridian example

Meridian's cost observability stack has six dashboards, twenty-two alerts, and an on-call rotation that's reduced cost-incident MTTR from 4 hours (pre-observability) to 23 minutes (median, after 12 months of refinement).

### 9.1 The dashboard set

```
Dashboard               Question                                Refresh
─────────────────────────────────────────────────────────────────────────
Cost-Per-Feature        Where is the money going?               5 min
Cost-Per-Tenant         Who is using the most?                  5 min
Cost-Per-User           Are any users outliers?                 hourly
Cost-Per-Model          Is tier-routing working?                5 min
Cost-Per-Prompt-Version Did a deploy increase cost?             on-deploy
Cost-vs-Usage-Cohort    Is cost growing faster than usage?      daily
```

Each is owned by the platform team; each has a documented question, normal range, and click-through.

### 9.2 The customer-facing dashboard

Built into the Meridian customer portal:

- Their cost over time (graph).
- Their cost by feature within their account.
- Their cost by user.
- Their budget consumption with warning indicators.

PII excluded. Updated every 15 minutes.

### 9.3 The alert set

22 alerts in three classes:

**Burn-rate alerts (8):**
- Per-feature monthly burn > 1.5x (warning) and > 2.0x (page).
- Per-tenant monthly burn > 1.5x and > 2.0x.
- Aggregate AI spend burn > 1.3x and > 1.8x.
- 1-hour fast burn > 14x (page); 6-hour > 6x (page).

**Spike alerts (6):**
- Per-feature 1-hour spend > 3x baseline.
- Per-tenant 1-hour spend > 5x baseline.
- Per-user 1-hour spend > 10x baseline.
- Per-model 1-hour spend > 3x baseline.
- Per-prompt-version cost-per-call > 2x baseline (post-deploy).
- Per-region 1-hour spend > 3x baseline (catches per-region anomalies).

**Anomaly alerts (8):**
- Per-feature MAD z-score > 4.
- Per-tenant MAD z-score > 4.
- Cost vs usage divergence (cost growing > 2x faster than usage).
- Free-tier tenant appearing in top-10 by spend.
- Cost-per-call distribution shift.
- New cost-bearing endpoint (un-attributed cost detected).
- Vendor invoice reconciliation > 5% drift.
- Cost paid for retries > 10% of base cost (signals retry storms).

### 9.4 The alert routing

- Platform team owns burn-rate, anomaly, and reconciliation alerts.
- Per-feature owners own per-feature spike and burn alerts.
- Customer-success owns tenant-level alerts that affect customer experience.
- On-call rotation 24/7; escalation chain to engineering manager → platform lead.

### 9.5 The runbook integration

Each alert links to a runbook in `meridian-platform/runbooks/cost/`:

- `cost-spike-per-feature.md` — triage and mitigation.
- `cost-burn-rate-warning.md` — slow-response procedure.
- `cost-spike-per-tenant.md` — tenant-specific investigation.
- `cost-anomaly-zscore.md` — anomaly triage.
- `cost-reconciliation-drift.md` — invoice vs attribution reconciliation.

### 9.6 The metrics on the observability stack itself

Tracked monthly:

- Dashboard usage: top 3 used at > 80% of weekday business hours.
- Alert fire rate: 5-10 alerts/week across all 22.
- False-positive rate: target < 20%, currently ~12%.
- MTTA: target < 15 min for pages, currently 8 min median.
- MTTR: target < 1 hour, currently 23 min median.

### 9.7 The Q4 2025 cost incident the dashboards caught

A new code change inadvertently produced a feature loop. Cost-per-call doubled overnight. The cost-per-prompt-version alert fired within 90 minutes of the deploy. On-call opened the runbook; the per-feature dashboard showed the affected feature; the per-prompt-version panel showed the spike correlated with the deploy.

Mitigation: roll back the prompt change. Resolution time: 41 minutes from alert to resolved.

Pre-observability, this would have been discovered when the monthly invoice arrived; estimated cost impact $40k. With observability, impact was ~$200.

### 9.8 The Q1 2026 noisy-neighbor incident

A free-tier tenant's misconfigured agent generated $4 of spend per hour for 3 hours before the cost circuit-breaker fired. The per-tenant spike alert fired at hour 1; the on-call triaged using the per-tenant dashboard; investigation showed the agent loop; tenant was notified.

Resolution: tenant fixed their agent. Cost: $12 (vs uncapped exposure of hundreds).

### 9.9 What the dashboards / alerts cost to maintain

- ~0.25 FTE platform engineer for ongoing maintenance (dashboards, alert tuning, runbook updates).
- Datadog / Grafana infrastructure: ~$1500/month.
- Initial setup: ~6 weeks (3 weeks for telemetry pipeline + 3 weeks for dashboards/alerts).

Avoided cost (estimated): $80-150k/year in caught incidents.

---

## 10. Anti-patterns

### 10.1 The "single global cost number" dashboard

**Pattern.** The cost dashboard shows total AI spend. No breakdown. When cost spikes, the dashboard tells you nothing about what to do.

**Corrective.** Per-feature, per-tenant, per-user, per-model breakdowns (§2). The aggregate is the headline; the breakdowns are the action.

### 10.2 The dashboard with 47 panels

**Pattern.** Every metric ever collected has a panel. Nobody looks at most of them. The dashboard is overwhelming; finding the right panel takes minutes.

**Corrective.** 6-10 panels per dashboard, each answering a specific question (§2.7). Unused panels are deleted.

### 10.3 The alert that always fires

**Pattern.** An alert that fires daily because the threshold is too low. On-call ignores it; it's silenced; real incidents get missed.

**Corrective.** Alert effectiveness review (§8.2). Tune up or remove.

### 10.4 The alert that never fires

**Pattern.** An alert with a threshold so high it's never tripped. Existing but useless.

**Corrective.** Alert effectiveness review. If the alert hasn't fired in a year, decide if the threshold is wrong or the alert is no longer needed.

### 10.5 The alert with no runbook

**Pattern.** Alert fires; on-call gets paged; on-call has no idea what to do.

**Corrective.** Runbook for every production alert (§7). No runbook → no production alert.

### 10.6 The runbook that's three years old

**Pattern.** Runbook exists; references dashboards that were renamed; references services that were retired; references people who left.

**Corrective.** Quarterly runbook validation (§7.4). Drill exercises.

### 10.7 The spike alert that fires on every Monday morning

**Pattern.** Naive spike alert with no day-of-week baseline. Fires on every traffic increase.

**Corrective.** Baseline using "same day-of-week, same hour-of-day" (§4.3).

### 10.8 The anomaly detection that fires on nothing important

**Pattern.** ML-based anomaly detection set up; fires daily on minor variations; team learns to ignore.

**Corrective.** Tune sensitivity; require multi-signal correlation; or remove the ML and use simpler signals.

### 10.9 The "dashboards in production, alerts in staging" gap

**Pattern.** Dashboards work in production; alerts were prototyped in staging and never moved to production. Production fires no alerts; production has no observability.

**Corrective.** Alert promotion process to production; pre-production observation period required.

### 10.10 The on-call who can't access the dashboard

**Pattern.** Alert fires for on-call from one team; the linked dashboard is in another team's tenant; on-call doesn't have access.

**Corrective.** Shared dashboard tenancy or per-team copies; verify on-call access during onboarding.

---

## 11. Findings (sprint-assignable)

### COST-DASH-001 — Severity: Critical
**Finding.** No per-feature cost dashboard.
**Recommendation.** Build the six-dashboard set per §2; start with per-feature trend.
**Owner.** observability-eng, sprint N+1.

### COST-DASH-002 — Severity: Critical
**Finding.** No burn-rate alerts.
**Recommendation.** Implement burn-rate alerts per §3; per-feature and aggregate; two thresholds (warning + critical).
**Owner.** observability-eng + SRE, sprint N+1.

### COST-DASH-003 — Severity: Critical
**Finding.** Alerts have no runbooks.
**Recommendation.** Runbook per alert; runbook-as-code in same repo as alert config; no runbook → no production alert (§7.3).
**Owner.** SRE + ops, sprint N+1.

### COST-DASH-004 — Severity: Critical
**Finding.** No per-tenant cost dashboard.
**Recommendation.** Per-tenant ranking dashboard per §2.2.
**Owner.** observability-eng, sprint N+1.

### COST-DASH-005 — Severity: High
**Finding.** Spike alerts not aware of day-of-week patterns.
**Recommendation.** Baseline using same-day-of-week, same-hour-of-day per §4.3.
**Owner.** observability-eng, sprint N+2.

### COST-DASH-006 — Severity: High
**Finding.** No per-prompt-version cost panel.
**Recommendation.** Track cost-per-call by prompt version per §2.5; alert on deploy-correlated spike.
**Owner.** AI platform + observability-eng, sprint N+2.

### COST-DASH-007 — Severity: High
**Finding.** No alert escalation chain.
**Recommendation.** Document escalation per §6.4; validate via fire-drill exercise.
**Owner.** SRE, sprint N+2.

### COST-DASH-008 — Severity: High
**Finding.** Customer-facing cost dashboard absent.
**Recommendation.** Customer dashboard per §2.8; cost, by feature, by user, vs budget.
**Owner.** product + AI platform, sprint N+2.

### COST-DASH-009 — Severity: High
**Finding.** Cost-vs-usage cohort dashboard absent.
**Recommendation.** Cohort dashboard per §2.6; alert on disproportionate growth.
**Owner.** observability-eng, sprint N+2.

### COST-DASH-010 — Severity: Medium
**Finding.** Alerts have no false-positive tracking.
**Recommendation.** Track per-alert metrics per §8.2; monthly review.
**Owner.** SRE, sprint N+3.

### COST-DASH-011 — Severity: Medium
**Finding.** No anomaly detection beyond simple thresholds.
**Recommendation.** Add MAD-based z-score anomaly detection per §5.3; tune to < 20% FP.
**Owner.** observability-eng, sprint N+3.

### COST-DASH-012 — Severity: Medium
**Finding.** Launch announcements don't suppress alerts.
**Recommendation.** "Feature launching" state suppresses related alerts for configured window (§4.4).
**Owner.** observability-eng, sprint N+3.

### COST-DASH-013 — Severity: Medium
**Finding.** Runbook validation not on schedule.
**Recommendation.** Quarterly runbook drill per §7.4; document gaps; update.
**Owner.** SRE, sprint N+3.

### COST-DASH-014 — Severity: Medium
**Finding.** No alert cool-down configured.
**Recommendation.** Cool-down per §3.4; alert storms prevented.
**Owner.** SRE, sprint N+3.

### COST-DASH-015 — Severity: Medium
**Finding.** Dashboard usage not tracked.
**Recommendation.** Usage metrics per §8.1; unused dashboards deleted.
**Owner.** observability-eng, sprint N+4.

### COST-DASH-016 — Severity: Medium
**Finding.** Alert routing not per-feature.
**Recommendation.** Per-feature ownership in routing layer per §6.2.
**Owner.** SRE, sprint N+4.

### COST-DASH-017 — Severity: Low
**Finding.** No reconciliation-drift alert.
**Recommendation.** Alert on vendor invoice vs attribution drift > 5% (catches attribution gaps).
**Owner.** AI platform + finance, sprint N+5.

### COST-DASH-018 — Severity: Low
**Finding.** No "retries pay too much" alert.
**Recommendation.** Track retry-cost share; alert when > 10% of base cost (signals retry storms or non-idempotent calls).
**Owner.** AI platform, sprint N+5.

---

## 12. Adoption sequencing checklist

- [ ] **Build per-feature cost trend dashboard (§2.1).** First panel; supports basic "where is the money" question.
- [ ] **Build per-tenant cost ranking (§2.2).** Catches noisy neighbors.
- [ ] **Build per-user distribution (§2.3).** Catches outlier users.
- [ ] **Build per-model breakdown (§2.4).** Validates tier-routing.
- [ ] **Build per-prompt-version panel (§2.5).** Catches deploy-induced cost changes.
- [ ] **Build cost-vs-usage cohort (§2.6).** Catches disproportionate growth.
- [ ] **Implement burn-rate alerts (§3).** Two thresholds per scope; multi-window for fast-burn.
- [ ] **Implement spike alerts (§4).** Baseline aware of day-of-week.
- [ ] **Implement anomaly alerts (§5).** Start with simple MAD; add ML only if needed.
- [ ] **Build runbook per alert (§7).** No production alert without a runbook.
- [ ] **Document alert routing (§6).** Per-feature owners; escalation chain.
- [ ] **Build customer-facing cost dashboard (§2.8).** PII-excluded subset.
- [ ] **Track alert effectiveness (§8).** Monthly review; tune or remove.
- [ ] **Quarterly runbook validation drill (§7.4).**
- [ ] **Pre-production alert observation period.** 2 weeks before promotion to paging.

---

## 13. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — telemetry that feeds these dashboards.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — the circuit-breaker that fires on threshold; alerts here are the early-warning.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) — per-tenant budgets surfaced in the per-tenant dashboard.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — the routing strategy validated by the per-model dashboard.
- [caching-for-cost.md](./caching-for-cost.md) *(companion)* — cache effectiveness surfaces in cost trends.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) *(companion)* — batch usage tracked.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) *(companion)* — rate-limit signals in dashboards.
- [cost-incident-runbook.md](./cost-incident-runbook.md) *(coming)* — the runbook referenced by alerts.
- [finops-process.md](./finops-process.md) *(coming)* — process that consumes the dashboards.

**Elsewhere in this repo.**
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — general alert design patterns.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — broader cost dashboards perspective.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — quality drift that often correlates with cost drift.

**Sibling repos.**
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — multi-tenant detection patterns that compose with per-tenant dashboards.
- [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md) — per-model cost feeds catalogue.

**External.**
- Google SRE Book — Multi-window, multi-burn-rate alerts (chapter 5 of the second SRE book).
- Datadog / Grafana / New Relic / Splunk dashboarding documentation.
- PagerDuty / Opsgenie incident management documentation.
- FinOps Foundation — KPI library, alert practices.
