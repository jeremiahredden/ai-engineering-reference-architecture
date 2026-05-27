# FinOps Process

> **Audience.** Engineering leads accountable for AI cost. FinOps practitioners building the AI overlay on the broader FinOps practice. Engineering managers whose quarterly business review asks "why is AI spend 3x what we projected?" **Scope.** The *engineering* practice of the cross-functional cadence that keeps AI cost in engineering's accountability loop: monthly cost review, per-team chargeback statements, budget-vs-actual report for leadership, launch-readiness cost gate, quarterly forecasting, alignment with finance. Not the per-call attribution (see [cost-attribution.md](./cost-attribution.md)). Not the alerts and dashboards (see [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md)). Not the incident response (see [cost-incident-runbook.md](./cost-incident-runbook.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI cost discipline is not a one-time engineering investment; it's an ongoing process. The dashboards, alerts, budgets, and circuit-breakers are infrastructure. The process is what makes the infrastructure produce business value.

Without process:

- Engineers ship features without knowing what they'll cost.
- The monthly invoice arrives; finance is upset; nobody is accountable.
- Quarterly business reviews include "AI cost" as an unexplained line item.
- Cost surprises become routine.
- Engineering leadership loses the thread on which features are economically viable.

With process:

- Every feature has a budget before it ships.
- Engineers see their team's cost monthly.
- Engineering leadership has a forecasted-vs-actual report each quarter.
- Cost incidents inform process improvements.
- Finance and engineering speak a common language.

The process isn't elaborate; it's predictable. Monthly cost review; monthly chargeback; launch-readiness gate; quarterly forecasting. Each is a recurring meeting or artifact. The discipline is in the recurrence, not the complexity.

This document covers what these processes look like, who participates, what artifacts they produce, and how they integrate with the broader FinOps practice.

This document is opinionated about four things:

1. **AI cost belongs to engineering, not finance.** Finance can report on the spend; only engineering can change the architectural and engineering choices that produced it. The process must reflect engineering accountability.
2. **Per-team chargeback drives behavior.** Aggregate "AI cost" doesn't motivate; "your team's $42k for last month" does. The chargeback is a primary lever.
3. **Launch-readiness cost gate is non-negotiable.** A feature ships with a budget, a kill switch, and a tracked cost-per-call. No exceptions for "small features" or "experimental." Without the gate, runaway features compound.
4. **The process must run even when cost is fine.** Skipping the monthly review when spend is "normal" produces drift. The cadence is the discipline; uneventful months are still reviewed.

Structure: (2) the monthly cost review; (3) per-team chargeback statements; (4) budget-vs-actual report for leadership; (5) launch-readiness cost gate; (6) quarterly forecasting; (7) alignment with finance; (8) the worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption sequencing; (12) references. (One section less than typical; FinOps process is process-heavy not concept-heavy.)

---

## 2. The monthly cost review

The standing meeting that keeps cost visible.

### 2.1 The cadence

Once a month, ~60 minutes. Participants: engineering leads of teams owning AI features; AI platform team; FinOps partner; sometimes engineering management.

### 2.2 The agenda

```
1. Aggregate spend review (10 min)
   - Last month's total AI spend vs budget.
   - Trend over last 6 months.
   - Notable shifts.

2. Per-team breakdown (15 min)
   - Each team's spend.
   - Compared to last month and to budget.
   - Notable outliers discussed.

3. Per-feature focus (15 min)
   - Top 5 features by spend.
   - Top 5 features by growth rate.
   - Discussion of any concerning patterns.

4. Incident review (10 min)
   - Cost incidents from last month.
   - Prevention plan status.

5. Forecasting (5 min)
   - Updated forecast for next month.
   - Adjustments to budgets if needed.

6. Action items (5 min)
   - Who owns what.
```

The agenda is consistent; participants know what to expect.

### 2.3 The data pulled in advance

Pre-meeting, the FinOps partner pulls:

- Total spend (last month, vs forecast).
- Per-team breakdown (cross-link to [cost-attribution.md](./cost-attribution.md)).
- Per-feature breakdown.
- Anomalies / outliers.
- Incident log.

Participants review the data 24 hours in advance; come prepared with context.

### 2.4 The "no surprises" principle

The monthly meeting shouldn't reveal surprises. Surprises mean the alerting / dashboard / on-call layer failed. Most months are uneventful; the review confirms.

When surprises happen:

- Discussion focuses on why the surprise wasn't caught earlier.
- The detection layer is the topic, not the surprise itself.

### 2.5 The "uneventful is fine" posture

A monthly review where everything is normal is still valuable:

- Trends are confirmed.
- Forecasts are validated.
- Forward-looking discussion happens (next month's launches; expected changes).

Don't cancel the meeting because nothing's wrong; that's how drift begins.

### 2.6 The action-item discipline

Items from the meeting are tracked:

- Owners.
- Due dates.
- Status (visible in next month's meeting).

Action items from the previous meeting are reviewed at the start of each new meeting.

### 2.7 The audience-appropriate variant

For larger orgs, the review may have multiple variants:

- Engineering-level monthly: deep-dive; engineering leads.
- Leadership-level monthly: summary; engineering management + finance.
- Annual: forward-looking; broader audience.

The engineering-level is the operational meeting; the leadership-level rolls up.

---

## 3. Per-team chargeback statements

Monthly accountability per team.

### 3.1 The chargeback statement

Each team owning AI features gets a monthly statement:

```
Team: Clinical AI Team
Period: 2026-05-01 to 2026-05-31

Features owned by this team:
  - Care Coordinator agent:     $42,150
  - Patient API chat (clinical): $14,200
                                 ────────
Team total:                      $56,350

Budget:                          $60,000
Variance to budget:              -$3,650 (-6%; under budget)

Compared to last month:          +$2,800 (+5%)
Compared to last quarter avg:    +$8,500 (+18%)

Per-feature breakdown:
  Care Coordinator:
    - Calls: 142,000
    - Tokens: 280M
    - Cost: $42,150
    - Cost-per-call: $0.297
    - Trend: +12% calls, +8% cost-per-call
    
  Patient API chat (clinical):
    - Calls: 380,000
    - Tokens: 95M
    - Cost: $14,200
    - Cost-per-call: $0.037
    - Trend: stable

Top cost drivers (calls × cost-per-call):
  1. Care Coordinator agent task type "referral letter": $18k
  2. Care Coordinator agent task type "care plan": $12k
  3. Patient API clinical chat sessions: $14k

Cost-saving opportunities identified:
  - Cache hit rate for Care Coordinator: 42% (target 50%); $3k/mo gap.
  - 8% of Care Coordinator calls have token counts > 8k; review prompt size.

Action items:
  - Investigate prompt size outliers (owner: clinical-ai-lead, due: 2026-06-15)
```

### 3.2 The statement audience

Primary: the team's engineering lead. Secondary: team members; engineering management.

The statement is shared at the monthly review and persisted (PDF, dashboard view, or both).

### 3.3 The chargeback as conversation

The statement is the basis for conversation, not just reporting. Each month, the team:

- Reviews the trends.
- Discusses the cost-saving opportunities.
- Acknowledges or pushes back on the data.
- Decides what (if anything) to act on.

The conversation is what produces behavior change.

### 3.4 The internal vs customer-facing variant

For customer-facing chargeback (external customers; cross-link to [per-tenant-cost-control.md §7](./per-tenant-cost-control.md)), the same data may be customer-visible.

For internal chargeback, additional internal-only fields:

- Cost vs budget.
- Team comparisons.
- Cost-saving opportunities specific to the team's context.

### 3.5 The "team owns the budget" mental model

Each team's budget is theirs. They:

- Set the budget (with input from FinOps and engineering leadership).
- Manage to the budget.
- Negotiate when business needs require a change.

The platform team doesn't own teams' budgets; they own the infrastructure that enforces them.

### 3.6 The "team comparisons" trap

Avoid:

- "Team A is more expensive than Team B" without context.
- Public per-team rankings that produce internal competition.
- Punitive framing.

Encourage:

- "Team A's cost-per-call is X; Team B's is Y; what could each learn from the other?"
- Best-practice sharing across teams.
- Constructive trends within team's own history.

### 3.7 The action-item closure

Each chargeback includes action items from the previous month's review:

- Status of each action item.
- Closed items removed from next month's statement.
- Overdue items highlighted.

Provides accountability without being punitive.

---

## 4. Budget-vs-actual report for leadership

The summary that reaches executives.

### 4.1 The leadership report

Monthly (or quarterly, depending on org cadence):

```
AI Cost Summary — 2026-Q2

Aggregate spend:
  - Q2 actual:                  $632,000
  - Q2 budget:                  $660,000
  - Variance:                   -$28,000 (-4.2%; under budget)
  - Q1 actual:                  $578,000
  - Q-over-Q growth:            +9.3%

Per-business-line:
  - Clinical:                   $310,000 (49%)
  - Patient-facing:             $185,000 (29%)
  - Internal tools:             $42,000 (7%)
  - Analytics:                  $58,000 (9%)
  - Embedding / infrastructure: $37,000 (6%)

Notable items:
  - Q2 includes Canadian acquisition (Atlantic Maple); $18k added.
  - Care Coordinator's per-call cost dropped 12% from Q1 due to caching improvements.
  - Q1 prompt-bloat incident (resolved); ~$320 impact, prevented recurrence with cost-impact PR gate.

Forecast for Q3:
  - Projected total: $700,000.
  - Drivers: organic growth (5%), expanded Canadian rollout (+$25k), new analytics feature (+$15k).
  - Budget recommendation: $720,000 (3% headroom).

Risks to forecast:
  - New Care Coordinator features in beta; cost-per-call unknown until fully released.
  - Provider pricing announcement expected in Q3 (Anthropic semi-annual cycle); typically minor.

Concentration:
  - Top 3 features account for 78% of spend.
  - Top 1 tenant accounts for 22% of spend.
```

### 4.2 The "what executives want to know"

The report addresses the questions executives actually have:

- Are we over or under budget?
- What's the trend?
- What drove the change?
- What's the forecast?
- What are the risks?
- What's our concentration (one feature / one tenant)?

Numbers without context don't inform decisions; numbers with narrative do.

### 4.3 The cadence

- Monthly for engineering management.
- Quarterly for executive review.
- Annually for board / investor (where applicable).

Different audiences; same underlying data; different summary depth.

### 4.4 The forecast as commitment

The forecast in the report is a commitment, not a guess:

- It includes named drivers.
- It excludes "we hope it stays flat."
- Variance from forecast is the topic of the next quarter's review.

Forecast accuracy improves over time as patterns become familiar.

### 4.5 The "tell the truth" discipline

Reports include bad news clearly:

- "Q2 spend grew 22% beyond budget; primary driver was [specific cause]; plan to recover is [specific]."

Hiding bad news in trend graphs or rolling averages destroys leadership trust.

### 4.6 The "compared to plan" framing

Compare actual to plan; explain variance. Don't just show actual.

Plan was set with assumptions; variance reveals which assumptions were wrong; learning improves next plan.

---

## 5. Launch-readiness cost gate

The pre-launch checklist for new AI features.

### 5.1 The gate

Before any new AI feature ships to production, it must pass the cost gate:

```
Feature: <name>
Launch date: <date>

[ ] Cost-per-call estimate computed (input + output rates × expected tokens).
[ ] Volume forecast (calls per day, week, month).
[ ] Monthly cost forecast (calls × cost-per-call).
[ ] Cost budget assigned (with team owner).
[ ] Cost attribution wired (per-call cost tracked).
[ ] Cost dashboard panel added.
[ ] Cost alert thresholds set (burn-rate + spike).
[ ] Kill switch implemented and tested.
[ ] Pre-launch eval includes cost validation.
[ ] Cost forecast vs actual review scheduled (30 days post-launch).
```

All checkboxes must be ticked. The gate is enforced by the launch process (deploy CI, launch review, etc.).

### 5.2 The cost forecast

The forecast is a structured estimate:

```yaml
feature: care-coordinator-bulk-import
volume_estimate:
  daily_calls_p50: 5000
  daily_calls_p99: 20000
cost_per_call_estimate:
  model: claude-sonnet-4-6
  avg_input_tokens: 4500
  avg_output_tokens: 800
  cost_per_call_p50: 0.025
  cost_per_call_p99: 0.040
monthly_cost_estimate:
  p50: 3750  # 5000 × 30 × $0.025
  p99: 24000  # 20000 × 30 × $0.040
budget_assignment:
  daily_usd: 200
  monthly_usd: 5000
  owner_team: clinical-ai-team
kill_switch:
  flag_name: care-coordinator-bulk-import-enabled
  default: true
  emergency_disable_procedure: documented_at_runbook_url
```

The forecast is reviewed by the FinOps partner; budget is set; gate is passed.

### 5.3 The "no budget = no launch" rule

Launches without budgets create unbounded cost exposure. The rule is firm:

- Feature has a budget OR
- Feature has been classified as "internal experimental" with capped exposure AND a planned graduation date.

There is no "we'll add a budget later" path.

### 5.4 The kill switch verification

Before launch:

- Kill switch is implemented.
- Kill switch is tested (verified to actually disable the feature).
- The runbook for invoking the kill switch is documented and accessible.
- On-call is briefed.

A kill switch that's never been tested is not a kill switch.

### 5.5 The post-launch cost review

30 days after launch:

- Actual cost vs forecast.
- Calls / tokens / cost-per-call.
- Surprises and learnings.
- Forecast updated.

Most forecasts are wrong; the 30-day review calibrates.

### 5.6 The launch-readiness review meeting

For significant launches, a launch-readiness review:

- Feature owner walks the checklist.
- FinOps partner reviews the forecast.
- Engineering management approves (or requests changes).

For smaller launches, async approval is acceptable; checklist still applies.

### 5.7 The "experimental feature" sub-process

For experimental features (proof-of-concept, not yet committed to production):

- Lower bar: forecast + small capped budget + kill switch.
- Time-bound: experimental status expires after N days.
- Graduation: must pass full gate to go to production.

Allows experimentation without bypassing discipline.

---

## 6. Quarterly forecasting

The forward-looking discipline.

### 6.1 The quarterly forecast

Each quarter:

- Aggregate forecast for next quarter.
- Per-team / per-feature breakdown.
- Named drivers.
- Risk register.

The forecast is presented to engineering management; informs budget setting for next quarter.

### 6.2 The forecasting inputs

- Last quarter's actual.
- Growth trends (calls, tenants, features).
- Known changes (new features launching, deprecated features, model upgrades).
- External factors (pricing changes, regulatory).

### 6.3 The forecast confidence bands

Each forecast number has a confidence band:

```
Q3 forecast:
  Base case: $700k
  Optimistic (10th percentile): $650k
  Pessimistic (90th percentile): $780k
  Tail risk (1st percentile): $1M+ (driven by [specific risk scenario])
```

Decision-makers see the band; informs how much headroom to budget.

### 6.4 The "what could surprise us" risk register

Identify known unknowns:

- New model release; might prompt migration.
- Customer onboarding pipeline; volume jump.
- Regulatory change; might shift workloads.
- Vendor pricing change cycle.

Each risk has a magnitude estimate; aggregate provides forecast band.

### 6.5 The "we miss the forecast" learning

When actual differs from forecast:

- Which driver missed? (volume, cost-per-call, new feature impact)
- What's the lesson for next quarter?
- How can the forecasting process improve?

Forecast accuracy is itself a tracked metric.

### 6.6 The forecast as input to budget

The forecast informs budget setting:

- Budget = forecast + headroom (typically 5-15%).
- Headroom accommodates forecast error.

Setting budget below forecast is a red flag; setting it far above forecast wastes capacity allocation.

---

## 7. Alignment with finance

The interface between engineering and the broader finance function.

### 7.1 The shared language

Finance has its own vocabulary (OPEX, CAPEX, run-rate, accrual). Engineering has its own (per-call cost, TPM, model tier).

The bridge:

- AI cost is OPEX (operational expense).
- Per-call cost translates to per-period spend.
- TPM relates to capacity utilization.

The FinOps partner translates; both sides share the vocabulary over time.

### 7.2 The invoice reconciliation

Monthly:

- Engineering attribution (per-call sums by provider).
- Finance receives the vendor invoice.
- Reconciliation: do the numbers match within 2%?
- Drift > 2% investigated; root cause identified; fix.

Cross-link to [cost-attribution.md §7](./cost-attribution.md).

### 7.3 The CFO conversation

Periodically (quarterly; on major changes):

- Engineering presents cost trends.
- CFO asks: forecast confidence, cost-saving opportunities, alignment with business priorities.
- Engineering provides specifics.

A trust-building conversation; both sides need it.

### 7.4 The vendor negotiation

Engineering and finance jointly negotiate with vendors:

- Engineering: technical fit; capability requirements; volume forecast.
- Finance: contract terms; payment structure; SLAs.
- Joint: total cost of ownership; vendor risk; commitment structure.

The negotiation is collaborative; one party alone doesn't represent the full picture.

### 7.5 The annual budget cycle

Each year:

- Engineering proposes AI cost budget for next year.
- Finance reviews; aligns with broader business budget.
- Adjustments negotiated.
- Final budget set; tracked through the year.

The cycle integrates with the company's broader annual planning.

### 7.6 The "we need more budget" conversation

When engineering needs more budget:

- Justification: business value of the additional spend.
- Alternatives considered (cost-saving options).
- Forecast for the increase.
- Trade-offs if denied.

Treating the conversation as a structured proposal (not a request) produces better outcomes.

---

## 8. Worked Meridian example

Meridian's FinOps process supports the engineering organization (~30 engineers across 6 teams) and a leadership tier (engineering management + CFO).

### 8.1 The monthly cost review

- Cadence: first Tuesday of each month, 60 minutes.
- Participants: engineering leads (6), AI platform lead, FinOps partner, occasional engineering manager.
- Standing agenda; data pulled in advance.

Average meeting outcome: 4-6 action items; 1-2 surfaced concerns; 80% of months "uneventful."

### 8.2 The chargeback statements

Each team receives a monthly statement. Notable:

- Clinical AI Team: ~$60k/month; their statement leads the review.
- Patient API Team: ~$30k/month.
- Analytics Team: ~$15k/month.
- Internal Tools Team: ~$8k/month.
- Platform Team: ~$10k/month (shared infrastructure).
- Customer Success Team: small (~$2k/month for internal copilot).

Statements include trends, top drivers, and cost-saving opportunities.

### 8.3 The leadership report

Monthly summary report to engineering management; quarterly to CFO.

Q2 2026 report covered:
- $632k actual vs $660k budget (-4%).
- 9% Q-over-Q growth (driven by Canadian acquisition + new analytics feature).
- Notable: prompt-bloat incident in March handled in 1h 37m; $320 impact; prevention shipped.
- Forecast: Q3 at $700k.

### 8.4 The launch-readiness gate

Every AI feature launched in 2026 went through the gate:

- Care Coordinator agent (Q1): forecast $25k/mo; budget $30k/mo; actual $28k/mo. Forecast within 12%.
- Document classifier batch (Q2): forecast $4k/mo; budget $5k/mo; actual $3.8k/mo. Forecast within 5%.
- Atlantic Maple Canadian deployment (Q2): forecast $18k/mo; budget $22k/mo; actual $16k/mo. Forecast within 11%.
- New analytics warehouse copilot feature (Q2): forecast $12k/mo; budget $15k/mo; actual $14k/mo. Forecast within 17%.

Average forecast accuracy: within 15%. Some features over-forecast, some under-forecast; aggregate calibrated.

### 8.5 The quarterly forecasting

Q3 forecast (made at end of Q2):
- Base: $700k.
- Confidence band: $650k - $780k.
- Risks: provider pricing change cycle (mild); new Care Coordinator features still in beta (moderate); enterprise customer expanded usage (moderate).
- Headroom recommended: $720k budget.

### 8.6 The CFO alignment

Quarterly conversation with CFO covers:

- AI spend trajectory.
- ROI: AI feature revenue impact; customer growth correlation.
- Cost-saving initiatives in flight.
- Forecast confidence.

The conversation has become routine after 18 months; CFO trusts the engineering forecasts within ~15%.

### 8.7 The vendor negotiation outcome

Q1 2026: Meridian negotiated a 12-month commitment with Anthropic for Care Coordinator workload.
- Engineering: 200M tokens/month forecasted.
- Finance: 12-month commit; 15% discount.
- Joint: $130k/year savings.

The deal required engineering's forecast accuracy to be the basis of the commit. Without the forecasting discipline, the commit would have been risky.

### 8.8 The annual budget cycle

Annual budget for FY26 was set in late 2025:

- Engineering proposed $2.8M for AI.
- CFO approved $3.0M (with 7% headroom).
- Quarterly tracking against budget.
- Q1 + Q2 actual: $1.21M; on track for ~$2.5M annual (under budget).

### 8.9 The process improvement loop

Process improvements driven by reviews:

- 2025: launched per-team chargeback (improved accountability).
- Q1 2026: added launch-readiness gate (after one un-budgeted feature shipped).
- Q2 2026: added 30-day post-launch cost review (calibration).
- Q3 2026: added confidence bands to forecasting.

The process evolves; documented in this folder's runbooks.

### 8.10 What the process costs

- FinOps partner: ~0.3 FTE allocated.
- Monthly review meeting: 60 min × 8 attendees = 8 hours of meeting time.
- Quarterly forecasting: ~1 day FinOps + ~2 hours per team lead.
- Launch-readiness gate: ~2 hours per feature launch.

Total: ~5-8% of FinOps capacity + small slice of engineering leads' time.

### 8.11 What the process produces

- Cost predictability: actuals within 15% of forecast.
- Cost incidents: rare; well-mitigated when they occur.
- Vendor negotiations: data-driven; favorable terms.
- Leadership trust: AI cost is "known," not "scary."
- Cross-team learning: cost-saving practices spread.

The process is the difference between AI cost as a wildcard and AI cost as a managed expense.

---

## 9. Anti-patterns

### 9.1 The "FinOps is finance's problem" reflex

**Pattern.** Engineering doesn't own AI cost; finance reports on it; nobody changes architectural decisions. Cost grows uncontrolled.

**Corrective.** Engineering owns AI cost per §1; finance reports and aligns.

### 9.2 The skipped monthly review

**Pattern.** Monthly review is canceled when "nothing's wrong." Drift accumulates; the next "concerning" month produces a chaotic review.

**Corrective.** Cadence is the discipline. Run the meeting even when uneventful.

### 9.3 The chargeback nobody reads

**Pattern.** Statements generated; emailed; ignored. Team behavior doesn't change.

**Corrective.** Chargeback as conversation per §3.3. Review at monthly meeting; engage teams.

### 9.4 The launch with no budget

**Pattern.** A new feature ships "to try it"; no budget; no kill switch; cost is invisible until a surprise invoice.

**Corrective.** No-budget-no-launch per §5.3.

### 9.5 The forecast nobody believes

**Pattern.** Forecast is "we'll keep an eye on it." No specifics; no commitment. Variance from forecast can't be diagnosed.

**Corrective.** Named drivers; confidence bands; specific numbers per §6.

### 9.6 The "I'll add the dashboard panel later" deferral

**Pattern.** Feature launches without a cost-dashboard panel. Cost is invisible until the monthly review surfaces it.

**Corrective.** Dashboard panel required for launch per §5.1.

### 9.7 The action items that never close

**Pattern.** Each review generates action items; few close; new items added to the pile.

**Corrective.** Owners + due dates; surface overdue per §2.6. Closure rate is itself reviewed.

### 9.8 The bad-news-hidden report

**Pattern.** Leadership reports show only good trends; bad trends in detail tabs that nobody opens.

**Corrective.** Honest reporting per §4.5. Lead with reality; don't bury.

### 9.9 The vendor commit without engineering input

**Pattern.** Finance commits to a vendor based on past spend; engineering doesn't know; technical fit isn't validated; commit is too small or too large.

**Corrective.** Joint negotiation per §7.4.

### 9.10 The cost-saving plan that's actually a wish list

**Pattern.** "We'll improve caching" and "we'll do tier routing." No owner; no timeline; no measurement. Cost doesn't improve.

**Corrective.** Cost-saving initiatives have owners, timelines, target metrics. Track in monthly review.

---

## 10. Findings (sprint-assignable)

### COST-FIN-001 — Severity: Critical
**Finding.** No monthly cost review meeting.
**Recommendation.** Schedule recurring monthly review per §2; structured agenda.
**Owner.** engineering management + FinOps, sprint N+1.

### COST-FIN-002 — Severity: Critical
**Finding.** No per-team chargeback statements.
**Recommendation.** Monthly statements per §3; distributed before review meeting.
**Owner.** FinOps + AI platform, sprint N+1.

### COST-FIN-003 — Severity: Critical
**Finding.** Launch-readiness cost gate absent.
**Recommendation.** Gate per §5; required for all new AI feature launches.
**Owner.** engineering management + AI platform, sprint N+1.

### COST-FIN-004 — Severity: Critical
**Finding.** No budget assigned per team / feature.
**Recommendation.** Annual budget cycle per §7.5; per-team and per-feature allocations.
**Owner.** engineering management + FinOps, sprint N+1.

### COST-FIN-005 — Severity: High
**Finding.** No leadership cost report.
**Recommendation.** Monthly summary per §4; quarterly to executive.
**Owner.** FinOps + engineering management, sprint N+2.

### COST-FIN-006 — Severity: High
**Finding.** Quarterly forecasting process absent.
**Recommendation.** Forecast per §6 with confidence bands.
**Owner.** FinOps + engineering leads, sprint N+2.

### COST-FIN-007 — Severity: High
**Finding.** Post-launch cost review not scheduled.
**Recommendation.** 30-day post-launch review per §5.5; calibrate forecasts.
**Owner.** engineering management, sprint N+2.

### COST-FIN-008 — Severity: High
**Finding.** Action items from past reviews not tracked.
**Recommendation.** Tracker per §2.6; surfaced when overdue.
**Owner.** engineering management, sprint N+2.

### COST-FIN-009 — Severity: High
**Finding.** Invoice reconciliation not part of monthly process.
**Recommendation.** Monthly reconciliation per §7.2; drift > 2% triggers investigation.
**Owner.** AI platform + finance, sprint N+2.

### COST-FIN-010 — Severity: Medium
**Finding.** Engineering not present in vendor negotiations.
**Recommendation.** Joint engineering + finance negotiation per §7.4.
**Owner.** engineering management + procurement, sprint N+3.

### COST-FIN-011 — Severity: Medium
**Finding.** CFO interaction is reactive.
**Recommendation.** Quarterly proactive conversation per §7.3.
**Owner.** engineering management, sprint N+3.

### COST-FIN-012 — Severity: Medium
**Finding.** Forecast confidence not communicated.
**Recommendation.** Confidence bands per §6.3.
**Owner.** FinOps, sprint N+3.

### COST-FIN-013 — Severity: Medium
**Finding.** Experimental features ship without budget.
**Recommendation.** Experimental sub-process per §5.7.
**Owner.** engineering management, sprint N+3.

### COST-FIN-014 — Severity: Medium
**Finding.** Annual budget set without engineering forecast input.
**Recommendation.** Engineering proposes budget per §7.5; finance reviews.
**Owner.** engineering management + finance, sprint N+4.

### COST-FIN-015 — Severity: Medium
**Finding.** Cost-saving initiatives lack owners and metrics.
**Recommendation.** Each initiative has owner + target metric + timeline per §9.10.
**Owner.** AI platform + engineering management, sprint N+4.

### COST-FIN-016 — Severity: Low
**Finding.** Cross-team learning from cost-saving practices absent.
**Recommendation.** Quarterly cross-team learning session.
**Owner.** engineering management, sprint N+5.

### COST-FIN-017 — Severity: Low
**Finding.** Forecast accuracy not tracked over time.
**Recommendation.** Track per-quarter forecast vs actual; improve methodology.
**Owner.** FinOps, sprint N+5.

### COST-FIN-018 — Severity: Low
**Finding.** Process documentation not maintained.
**Recommendation.** This document + meeting templates kept current; reviewed quarterly.
**Owner.** FinOps + AI platform, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Schedule monthly cost review meeting (§2).** Standing time; standing agenda.
- [ ] **Build per-team chargeback statement (§3).** First statement before the first review.
- [ ] **Define launch-readiness cost gate (§5).** Required checklist; enforced in launch process.
- [ ] **Assign initial budgets per team / feature (§7.5).** Even if rough; refined over quarters.
- [ ] **Build leadership cost report (§4).** Monthly to engineering management.
- [ ] **Implement invoice reconciliation (§7.2).** Monthly.
- [ ] **Schedule quarterly forecasting (§6).** End-of-quarter forecasting for next quarter.
- [ ] **Schedule 30-day post-launch review (§5.5).** For each launch.
- [ ] **Track action items from reviews (§2.6).** Owners + due dates.
- [ ] **Build cost-saving initiatives tracker (§9.10).** Owner + metric + timeline per initiative.
- [ ] **Annual budget cycle (§7.5).** Engineering proposes; finance aligns.
- [ ] **CFO quarterly conversation (§7.3).** Proactive trust-building.
- [ ] **Quarterly cross-team learning (§9.x analog).**
- [ ] **Annual process review.** Update this document and meeting templates.

---

## 12. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — telemetry that produces chargeback data.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — kill switch infrastructure required by launch gate.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) — dashboards reviewed in monthly meeting.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) — customer chargeback statements.
- [caching-for-cost.md](./caching-for-cost.md) — cost-saving initiative tracked in reviews.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — cost-saving initiative tracked in reviews.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) — cost-saving initiative tracked in reviews.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) — enforcement of budgets set in reviews.
- [cost-incident-runbook.md](./cost-incident-runbook.md) — incidents reported and learned from in reviews.

**Elsewhere in this repo.**
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — broader cost dashboards.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md) — catalogue tracks per-model spend used in reviews.

**External.**
- FinOps Foundation — general FinOps practice; this folder is the AI-specific overlay.
- FinOps for AI / GenAI working group materials.
- AWS Well-Architected — cost optimization pillar.
- Stripe / SaaS metering / billing literature.
- Engineering management literature on budget accountability (e.g., Will Larson, Camille Fournier).
