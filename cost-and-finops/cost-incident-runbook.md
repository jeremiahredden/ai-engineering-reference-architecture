# Cost Incident Runbook

> **Audience.** SREs and engineering leads who get paged for cost spikes. Engineers triaging "our AI bill jumped 4x yesterday." On-call rotations whose runbook for cost incidents is currently a wiki page nobody updates. **Scope.** The *engineering* practice of responding to AI cost incidents: detection signals; triage workflow (feature / tenant / model / prompt-version / endpoint); mitigation actions (rate-limit, route-down-tier, kill switch); root-cause investigation; resolution and prevention. Not the cost-attribution telemetry (see [cost-attribution.md](./cost-attribution.md)). Not the alerting setup (see [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md)). Not the circuit-breaker primitive (see [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md)). Not the general reliability incident response (see [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Cost incidents are a class of their own. They share some properties with reliability incidents (something is wrong; on-call is paged; mitigation is needed) and some with security incidents (the spend is real; reversing it is hard). But the playbook is distinct.

A cost incident's signature:

- Spend is climbing faster than expected (or has already climbed past a threshold).
- Sometimes user-facing latency / quality looks normal (the system is "working" — just expensive).
- The blast radius is financial: every minute that mitigation lags is more spend.
- Sometimes the cause is internal (prompt bloat, agent loop, retrieval bloat); sometimes external (vendor pricing change, abuse, tenant misconfiguration); sometimes the system itself (a bug in retry logic that creates a retry storm).

The reliability-incident playbook ("triage symptoms; identify failing component; restart / roll back") doesn't quite work. You're not looking for a broken component; you're looking for one that's working too much.

This document covers the engineering-side runbook. The architecture-side context (per-tenant cost control, budgets, fairness) is documented elsewhere; this is what on-call does when the alert fires.

This document is opinionated about four things:

1. **Mitigation precedes investigation.** Once the alert fires, the goal is to stop the bleeding first. Root cause comes after. A cost incident running 15 extra minutes during investigation can cost more than an hour of investigation later.
2. **Mitigation is reversible; investigation isn't urgent.** Apply the brake (rate-limit, kill switch) first; once spend stops climbing, take the time to investigate properly. The brake can be released after investigation.
3. **Every cost incident has a "should-have-caught-earlier" lesson.** Most cost incidents are visible in some signal hours before the page; the post-incident review identifies which signal should have caught it and tightens the alert.
4. **Cost incidents need their own runbook, not a sub-section of the general runbook.** Reliability and cost are different; pretending they're the same produces slow cost response.

Structure: (2) the cost incident classes; (3) detection signals; (4) the triage workflow; (5) mitigation actions; (6) root-cause investigation; (7) resolution and prevention; (8) the post-incident review; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The cost incident classes

Most cost incidents fall into one of seven classes. The class shapes the triage and mitigation.

### 2.1 Runaway cost (general)

**Pattern.** Spend climbs unexpectedly. Cause is unclear at the moment of detection.

**Typical signal.** Burn-rate alert at 2.0+; aggregate AI spend spike; per-feature spike.

**Initial mitigation.** Rate-limit the affected feature; if multiple features, rate-limit the platform; investigate.

### 2.2 Prompt-bloat-induced

**Pattern.** A prompt update increased per-call token count. Cost-per-call rose; total cost grows proportionally to traffic.

**Typical signal.** Per-prompt-version cost-per-call alert; correlation with recent deploy.

**Initial mitigation.** Roll back the deploy (fastest); or hotfix the prompt to trim tokens.

### 2.3 Retrieval-bloat-induced

**Pattern.** Retrieval is returning more or larger chunks; the LLM call receives more tokens; cost grows.

**Typical signal.** Per-feature cost-per-call alert; retrieval result size metric.

**Initial mitigation.** Adjust retrieval parameters (lower k, tighter filter); or roll back if a recent change is the cause.

### 2.4 Agent-loop

**Pattern.** Agent task is recursing or looping; each agent task generates many more LLM calls than expected.

**Typical signal.** Per-feature spike; agent-task average LLM calls metric; per-tenant spike if the bad agent is tenant-deployed.

**Initial mitigation.** Kill running agent tasks (preserve evidence first); add per-task LLM-call cap.

### 2.5 Abuse / external

**Pattern.** A tenant (often free-tier) is generating high cost; possibly malicious, possibly misconfigured.

**Typical signal.** Per-tenant spike; tenant appears in top-spend list for the first time; cost-per-user spike for one user.

**Initial mitigation.** Tighten the tenant's budget; pause the tenant's high-cost workloads; contact the tenant.

### 2.6 Pricing change

**Pattern.** Provider updated pricing; spend increases for unchanged workloads.

**Typical signal.** Per-token-cost change visible in attribution; reconciliation drift; provider announcement.

**Initial mitigation.** Verify the change; update rate tables; communicate impact internally; consider routing changes if cost change is significant.

### 2.7 Retry storm

**Pattern.** Provider is degraded; consumer's retry logic is firing aggressively; each retry incurs cost.

**Typical signal.** Per-feature cost spike correlated with elevated retry metric; provider error rate elevated.

**Initial mitigation.** Pause retries (or apply much stronger backoff); investigate provider status.

### 2.8 The classification matters

Different mitigations apply to different classes. A prompt-bloat incident is fixed by rolling back the deploy; an abuse incident is fixed by tightening tenant controls; a retry storm is fixed by pausing retries. Triaging into the right class is the first investigation step (§4).

---

## 3. Detection signals

What an alert looks like; what it means.

### 3.1 The alert payload

When a cost alert fires, the page should include:

```
ALERT: Per-feature burn rate exceeded 2.0x
Feature: care-coordinator
Current burn rate: 2.4x
Current period spend: $1,650 (limit: $1,500/day)
Time elapsed in period: 13 hours
Top contributing model: claude-sonnet-4-6 ($1,450)
Top contributing tenant: meridian-regional-system ($820)
Runbook: https://runbooks.meridian.example.com/cost/burn-rate-feature
```

On-call has enough to start triaging without opening 5 tabs.

### 3.2 The signal types

- **Burn-rate alert.** Spend rate × period elapsed > budget; will exceed before period ends.
- **Spike alert.** Recent hour's spend > N× baseline.
- **Anomaly alert.** Statistical outlier (z-score; MAD) in some dimension.
- **Per-tenant alert.** One tenant's spend pattern is unusual.
- **Per-prompt-version alert.** Cost-per-call shifted after a deploy.
- **Reconciliation drift alert.** Per-call attribution diverges from vendor invoice.

Cross-link to [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md).

### 3.3 The "is this real" first check

Some alerts are false-positive. The first 30 seconds:

- Spot-check the dashboard. Does the trend match the alert?
- Check the alert's history. Has this alert fired recently? Was it a false positive?
- Check the time of day. Is this a known traffic pattern (Monday morning spike)?

If real → continue triage. If false → silence the alert; investigate why it fired; tune.

### 3.4 The escalation triggers

Some signals justify immediate escalation to engineering manager / leadership:

- Burn rate > 5x (catastrophic).
- Aggregate AI spend > $X/hour (configured per platform).
- Multi-feature simultaneous incident (suggests platform issue).

Escalation isn't always to fix it faster; sometimes it's to inform stakeholders proactively.

### 3.5 The "no alert fired but the bill is high" path

Sometimes a cost incident is discovered after the fact (next-day invoice review, FinOps catches it). The runbook applies retrospectively:

- Identify the time window of the spike.
- Walk the same triage as a real-time incident.
- Determine which alert should have fired and didn't.
- Tune the alert system.

Retrospective response is slower but the lesson (improve detection) is the same.

---

## 4. The triage workflow

Within minutes of the page, on-call must classify the incident.

### 4.1 The triage decision tree

```
Cost spike detected.
    ↓
Is the spike per-feature? Per-tenant? Per-model? Per-prompt-version?
    ↓
Per-feature spike?
  → Which feature? Open per-feature dashboard.
  → Recent deploy on that feature? Check deploy log.
  → Prompt change? → Prompt-bloat (§2.2)
  → Retrieval change? → Retrieval-bloat (§2.3)
  → Agent change? → Agent-loop (§2.4)
  → No recent change? → Continue.
    ↓
Per-tenant spike?
  → Free tier? → Likely abuse / misconfig (§2.5)
  → Premium tier? → Often legitimate; verify with customer success.
    ↓
Per-model spike?
  → New model adoption? Verify cost rates correct.
  → Routing change? Check tier-routing config.
    ↓
Per-prompt-version spike?
  → Recent deploy? → Prompt-bloat (§2.2)
    ↓
None match cleanly?
  → Retry storm (§2.7)? Check provider error rate.
  → Pricing change (§2.6)? Check provider announcements.
  → Generic runaway (§2.1)? Apply broad mitigation; investigate.
```

The triage takes 5-10 minutes; the right classification accelerates mitigation.

### 4.2 The triage dashboards

For each alert, the runbook links to triage dashboards:

- Per-feature dashboard (which feature).
- Per-tenant dashboard (which tenant).
- Per-prompt-version dashboard (which deploy).
- Provider-side dashboard (retry rate, error rate).

The dashboards are pre-built; on-call doesn't construct queries.

### 4.3 The "log into the customer's UI" path

For tenant-specific incidents, sometimes the fastest triage is to log into the tenant's UI (with appropriate access controls) and observe the workflow live. Common for abuse / misconfig where the tenant's own UI behavior is the cause.

### 4.4 The triage time budget

- 5 minutes: classification.
- 10 minutes: targeted investigation (which specific call pattern).
- 15 minutes: mitigation decision.

If triage exceeds 15 minutes without classification, apply broad mitigation (rate-limit the platform; rate-limit the feature) and continue triage.

### 4.5 The "two classes simultaneously" case

Sometimes an incident is multi-class (e.g., a prompt bloat in one feature *and* a tenant runaway in another). The right move: treat them as separate incidents; assign different on-call leads if needed.

A single incident lead can't simultaneously triage two unrelated causes; splitting is faster.

---

## 5. Mitigation actions

What to do once the class is identified.

### 5.1 Rate-limit tightening (broadest)

Tighten existing rate limits on the affected scope:

- Per-feature: cut TPM or cost budget by 50%.
- Per-tenant: cut TPM or cost budget by 80%.
- Platform-wide: cut provider RPM consumed by 50% (forces queueing).

Effects:

- Stops the spend.
- Affects some legitimate traffic (those bumping against tightened limit get 429s).
- Reversible; restore limits once investigated.

### 5.2 Route-down-tier (model fallback)

Force traffic to a cheaper model:

- All Sonnet calls → Haiku for the affected feature.
- All Opus calls → Sonnet for premium tenants.

Effects:

- Reduces per-call cost immediately (10-30x reduction at upper tiers).
- Quality may degrade.
- Reversible.

Suitable for short-term during investigation; not for long-term without eval.

### 5.3 Kill switch (feature disable)

The strongest mitigation: disable the feature entirely:

- API returns "feature temporarily unavailable."
- Spend on the feature stops.
- Customer-visible.

Effects:

- Customers see the outage.
- Required for high-cost incidents where no other mitigation is fast enough.

Cross-link to [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — the kill switch is implemented as a circuit-breaker.

### 5.4 Per-tenant pause

For tenant-specific incidents:

- Tenant's API key revoked or paused.
- Tenant's calls return 429 or specific error.
- Other tenants unaffected.

Faster than rate-limit tightening for the tenant; less customer-visible than full kill switch.

### 5.5 Retry pause

For retry storm:

- Disable retries entirely for the affected provider.
- Calls that fail fail-fast; no retry incurs cost.

Reversible once provider degradation resolves.

### 5.6 Hotfix deploy

For prompt or code bug:

- Identify the bad commit.
- Revert or hotfix.
- Deploy.

Faster than rate-limit if the issue is code-side and confidently identified.

### 5.7 Mitigation decision matrix

| Class | Primary mitigation | Secondary |
| --- | --- | --- |
| Runaway (general) | Rate-limit broad | Kill switch if continues |
| Prompt bloat | Revert deploy | Rate-limit while reverting |
| Retrieval bloat | Tighten retrieval params | Kill switch if continues |
| Agent loop | Per-task LLM call cap | Kill switch agent feature |
| Abuse / external | Per-tenant pause | Kill API key |
| Pricing change | Update rate tables | Route to cheaper model |
| Retry storm | Pause retries | Disable provider temporarily |

### 5.8 The "mitigation might break legitimate traffic" trade-off

Most mitigations have collateral damage. The decision:

- Spend rate: how fast is money being lost?
- Mitigation breadth: how much legitimate traffic is affected?
- Reversal speed: how fast can you reverse if mitigation is wrong?

For fast-burning incidents, accept broader mitigation. For slow-burning, prefer targeted.

---

## 6. Root-cause investigation

Once mitigated, investigate the underlying cause.

### 6.1 The "preserve evidence" first move

Before reverting / killing:

- Snapshot the suspect logs (the LLM calls that contributed to the spike).
- Snapshot the prompt-versions in use.
- Snapshot the tenant / feature configs.
- Note the timeline (when did the spike start; when was it detected; when was it mitigated).

Investigation depends on this evidence.

### 6.2 The investigation tools

- **Per-call trace.** Open the trace for sample calls during the spike. What was in the prompt? What did the model return? How long did it take?
- **Per-feature dashboard.** Compare today's pattern to last week's. Where's the divergence?
- **Recent change log.** Code deploys, prompt updates, config changes, customer onboards in the last 24-48 hours.
- **Provider status / dashboards.** Did provider have issues? Pricing change? New rate limits?
- **Tenant-side investigation.** If tenant-specific, what's happening on their side?

### 6.3 The "5 whys" path

For deeper cause:

1. Why did spend spike? Because feature X cost-per-call doubled.
2. Why did cost-per-call double? Because prompt tokens grew from 4k to 8k.
3. Why did prompt tokens grow? Because the prompt-update PR added 50 new few-shot examples.
4. Why did the PR add 50 examples? Because the engineer was experimenting with prompt engineering and didn't realize the impact.
5. Why didn't the impact get caught in review? Because PR review for prompt changes wasn't required.

The 5-whys often reveals process gaps, not just immediate causes.

### 6.4 The classes of root cause

- **Code/prompt change.** Recent deploy introduced inefficiency.
- **Config change.** Routing, budgets, or feature flags adjusted.
- **Workload change.** Customer behavior shifted (more long-context queries; more tenants onboarded simultaneously).
- **Infrastructure change.** Cache cleared; retrieval index changed; rate-limit budget updated.
- **External.** Provider pricing; provider outage; tenant external behavior.
- **Bug.** Code defect that produces inefficiency under specific conditions.

Identifying the class shapes the prevention.

### 6.5 The "no clear cause" outcome

Sometimes investigation yields no clear cause. Possibilities:

- Many small changes compounded.
- The cause is in a system we don't observe (e.g., provider-side change).
- The cause is in a tenant's behavior we don't fully understand.

Document the investigation; note that cause is unclear; tighten monitoring to catch recurrence; revisit if the pattern recurs.

### 6.6 The "is it still happening" check

Periodically during investigation, verify mitigation is holding:

- Spend rate is back to baseline?
- The affected metric (per-feature cost, per-tenant cost) is stable?
- No new alerts firing?

If mitigation is leaking, tighten further before continuing investigation.

---

## 7. Resolution and prevention

Closing the loop.

### 7.1 The resolution checklist

Once root cause is identified:

- Fix the immediate cause (deploy revert, prompt fix, config rollback, tenant action).
- Verify spend rate returns to baseline.
- Release mitigation (restore rate limits, re-enable feature) gradually; verify spend stays normal.
- Document the timeline in the incident report.
- Communicate resolution to stakeholders (customer success, leadership, affected teams).

### 7.2 The prevention plan

For each incident, the post-incident review produces a prevention plan:

- New alert: an alert that would have caught this earlier.
- Process change: PR review requirements, deploy guardrails.
- Code change: defensive checks, bounded loops, retry caps.
- Architecture change: budget enforcement at a layer that was bypassed.
- Documentation: runbook update so next on-call handles it faster.

### 7.3 The "did we catch the next one earlier" measurement

The previous incident's prevention plan should be measurable. Did the alert fire on the next occurrence? Did the code change prevent recurrence?

Track prevention effectiveness; not every prevention works.

### 7.4 The customer-facing communication

Customer-facing incidents (kill switch, per-tenant pause) require communication:

- Initial: "we detected an issue; investigating."
- During: "mitigation in place; investigating root cause."
- Resolution: "issue resolved; here's what happened."
- Post-incident: "here's what we're doing to prevent recurrence."

Customer trust depends on the communication, not just the engineering.

### 7.5 The internal post-mortem

For significant incidents:

- Blameless post-mortem within a week.
- Timeline reconstruction.
- Root cause analysis.
- Prevention plan with owners.
- Knowledge sharing (relevant teams).

For routine incidents (caught early, mitigated quickly), a shorter retrospective is enough.

### 7.6 The pricing-change-specific prevention

For pricing-change incidents (§2.6):

- Subscribe to provider release notes.
- Automated check of rate table currency.
- Quarterly reconciliation review surfaces price drift.

Pricing changes are predictable (providers usually announce); prevention is process, not engineering.

---

## 8. The post-incident review

Within a week of resolution, the structured review.

### 8.1 The post-incident review template

```
Incident: <descriptive title>
Date: <YYYY-MM-DD>
Duration: <start> → <end>
Severity: <P0/P1/P2/P3>

Summary:
<2-3 sentence executive summary>

Timeline:
<bullet-point timeline; key events; times>

Detection:
- Which alert fired?
- Was it fast enough?
- Was it accurate?

Triage:
- How long to classify?
- Were the dashboards sufficient?

Mitigation:
- What was applied?
- How effective?
- Any collateral damage?

Root cause:
- What was the underlying cause?
- How did it get to production?

Prevention plan:
- New alerts: <owner>
- Process changes: <owner>
- Code changes: <owner>
- Documentation: <owner>

Lessons learned:
- What worked well?
- What could improve?
- What surprised us?

Action items (with owners and due dates):
- ...
```

### 8.2 The review meeting

For significant incidents, a 30-60 min review:

- Incident lead presents timeline.
- Discuss what worked / didn't.
- Discuss prevention.
- Assign action items.
- Document in shared system.

### 8.3 The action-item follow-through

Action items must close:

- Each has an owner.
- Each has a due date.
- Each is tracked in the team's planning.

Unclosed action items from prior reviews are themselves a signal; track and surface.

### 8.4 The cross-team lessons sharing

Lessons relevant to other teams are shared:

- Internal newsletter / blog.
- Tech-talk presentation.
- Update to the platform runbook.

Knowledge spreads; future incidents are caught by other teams' familiarity.

---

## 9. Worked Meridian example

A representative cost incident at Meridian and how the runbook handled it.

### 9.1 The incident: Q4 2025 prompt regression

A deploy of the Care Coordinator agent included an experimental prompt change. The new prompt included more detailed instructions and additional few-shot examples; the token count grew from 4500 to 7800 (74% increase).

The deploy went out at 14:23. Cost-per-call jumped accordingly.

### 9.2 The detection

At 15:51 (88 minutes post-deploy), the cost-per-prompt-version alert fired:

```
ALERT: Cost-per-call shifted after recent deploy
Feature: care-coordinator
Prompt version: v37 (deployed 14:23 today)
Previous cost-per-call: $0.42
Current cost-per-call: $0.71
Shift: +69%
Recent calls on v37: 1,240
Runbook: cost-deploy-induced-shift
```

The alert was specific: it correlated with the deploy.

### 9.3 The triage (4 minutes)

- 15:51 — Alert fires. On-call paged.
- 15:52 — On-call opens runbook + per-feature dashboard.
- 15:53 — Confirms spike correlates with deploy at 14:23. Cost-per-call doubled.
- 15:55 — Classifies as prompt-bloat (§2.2).

### 9.4 The mitigation (8 minutes)

- 15:55 — Decision: revert the deploy.
- 15:56 — On-call contacts the engineer who shipped the change.
- 15:58 — Engineer concurs on revert.
- 15:59 — Revert PR opened.
- 16:00 — Revert deployed.
- 16:03 — Cost-per-call returns to $0.42 baseline (3 minutes for the change to fully propagate).

### 9.5 The investigation (during the 4 hours after)

- The new prompt included 12 additional few-shot examples (originally 3; experimentally 15).
- Each example was ~250 tokens.
- Engineering observation: the examples were copied from a teammate's exploration; not eval-validated; not cost-aware.
- The engineer was experimenting with prompt engineering and didn't realize the cost impact.

### 9.6 The root cause (5 whys)

1. Why did cost spike? Cost-per-call rose from $0.42 to $0.71.
2. Why did cost-per-call rise? Prompt token count rose 74%.
3. Why did the prompt grow? Additional few-shot examples added.
4. Why were examples added without cost review? PR review didn't require cost impact review.
5. Why didn't PR review require it? Prompt changes had been infrequent; cost impact assumed minor.

### 9.7 The prevention plan

- **New alert.** Per-prompt-version cost-per-call alert at +20% shift; previously +50% threshold (caught at 69% but earlier threshold would catch sooner).
- **Process change.** PR review for prompt changes now requires cost impact estimate (token count diff × cost rate × volume).
- **Tooling.** Pre-deploy check estimates cost impact and warns if > 10% change.
- **Documentation.** Prompt engineering guide updated with cost-awareness section.

### 9.8 The financial impact

- Duration of incident: 1h 37m (deploy to revert).
- Extra cost during incident: ~$320 (vs normal baseline).
- Compare to if alert hadn't fired (next-day discovery): ~$5k extra in 24 hours.

The early detection saved ~$4.7k.

### 9.9 The post-incident review

- Held 4 days later.
- 45-minute meeting.
- Engineer who shipped the change presented (blameless framing).
- Action items closed within 2 weeks.

### 9.10 The follow-up validation

3 months later (Q1 2026), a similar prompt change came through:

- Pre-deploy cost estimate: +15% token count.
- Pre-deploy warning fired: review needed.
- Cost-impact discussion in PR.
- Decision: examples are valuable; cost increase acceptable.
- Deploy went out; alert was suppressed for this prompt version (expected change).

The prevention worked: same kind of change, no incident.

---

## 10. Anti-patterns

### 10.1 The "investigate first, mitigate later" reflex

**Pattern.** On-call investigates root cause for 30+ minutes before applying any mitigation. Spend continues during investigation. The financial impact is double or triple what it would have been with fast mitigation.

**Corrective.** Mitigate first; investigate after. The brake is reversible.

### 10.2 The single-page runbook

**Pattern.** One runbook for all cost incidents. On-call has to read 5 pages to find the relevant section. By the time the right path is identified, 10 minutes are lost.

**Corrective.** Multiple runbooks; each alert links to the specific one.

### 10.3 The "we'll figure it out" missing runbook

**Pattern.** Alert fires; runbook is "TBD" or "contact platform team." On-call improvises; delay is significant.

**Corrective.** Runbook for every production alert per [cost-dashboards-and-alerts.md §7](./cost-dashboards-and-alerts.md).

### 10.4 The mitigation that doesn't reverse

**Pattern.** Mitigation applied; spend stops; incident "resolved"; mitigation forgotten. Days later, the feature is still rate-limited; legitimate users complain.

**Corrective.** Track mitigations; close incident only when mitigations are reversed (or explicitly maintained).

### 10.5 The post-incident review that never closes action items

**Pattern.** Action items assigned; nobody owns; sprint passes; action items roll over forever. Next incident has the same root cause.

**Corrective.** Action items have owners and due dates; tracked in planning; surfaced when overdue.

### 10.6 The kill switch that pages everyone

**Pattern.** Kill switch fires; pages 10 people; chaos coordinating. Mitigation is in place but communication is confused.

**Corrective.** Single incident commander; clear roles. Cross-link to standard incident response practices.

### 10.7 The "this is just the bill" acceptance

**Pattern.** Cost incident occurs; spend impact is "only" $5k; nobody investigates; same incident recurs monthly.

**Corrective.** Every cost incident → post-incident review (lightweight for small; heavyweight for large). Recurrence is signal.

### 10.8 The "we never see retry storms because retries are bounded" assumption

**Pattern.** "Our retry budget is 3 per request; storms can't happen." Provider outage causes 30% of requests to retry; 30% × 3 = 90% extra calls; storm.

**Corrective.** Provider-aware backoff per [cost-aware-rate-limiting.md §6.4](./cost-aware-rate-limiting.md). Bounded budget isn't enough at scale.

### 10.9 The "the alert is too sensitive; just silence it" workaround

**Pattern.** Alert fires too often; on-call silences; real incident happens; alert is silenced; missed.

**Corrective.** Tune the alert per [cost-dashboards-and-alerts.md §8.3](./cost-dashboards-and-alerts.md). Silencing is a smell.

### 10.10 The runbook that's never tested

**Pattern.** Runbook exists; never exercised; first real fire reveals broken links, missing dashboards, outdated procedures.

**Corrective.** Quarterly drill: simulate alert; follow runbook; note gaps; fix.

---

## 11. Findings (sprint-assignable)

### COST-INC-001 — Severity: Critical
**Finding.** No documented cost-incident runbook; first incident is improvised.
**Recommendation.** Build runbook per this document; one per alert class per §2.
**Owner.** SRE + AI platform, sprint N+1.

### COST-INC-002 — Severity: Critical
**Finding.** On-call lacks mitigation-first discipline; investigation precedes brake.
**Recommendation.** Document mitigation-first per §1; train on-call.
**Owner.** SRE, sprint N+1.

### COST-INC-003 — Severity: Critical
**Finding.** No kill switch per feature.
**Recommendation.** Feature kill switch primitive per §5.3 and [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md).
**Owner.** AI platform, sprint N+1.

### COST-INC-004 — Severity: High
**Finding.** Alert payload lacks triage context.
**Recommendation.** Include top contributing model/tenant/version in alert payload per §3.1.
**Owner.** observability-eng, sprint N+2.

### COST-INC-005 — Severity: High
**Finding.** Per-tenant pause requires platform admin intervention; slow.
**Recommendation.** Self-service per-tenant pause for on-call per §5.4.
**Owner.** AI platform, sprint N+2.

### COST-INC-006 — Severity: High
**Finding.** No post-incident review process for cost incidents.
**Recommendation.** Template + cadence per §8.
**Owner.** SRE + engineering management, sprint N+2.

### COST-INC-007 — Severity: High
**Finding.** Mitigation reversal not tracked; lingering rate limits affect legitimate traffic.
**Recommendation.** Mitigation log + reversal check per §10.4.
**Owner.** SRE, sprint N+2.

### COST-INC-008 — Severity: High
**Finding.** No pre-deploy cost estimate for prompt changes.
**Recommendation.** Pre-deploy estimate per §9.7; warn if > 10% change.
**Owner.** AI platform + tooling, sprint N+2.

### COST-INC-009 — Severity: Medium
**Finding.** Triage dashboards not pre-built; on-call constructs queries during incident.
**Recommendation.** Per-class dashboards per §4.2.
**Owner.** observability-eng, sprint N+3.

### COST-INC-010 — Severity: Medium
**Finding.** Retry storm protection insufficient.
**Recommendation.** Provider-aware backoff per [cost-aware-rate-limiting.md §6.4](./cost-aware-rate-limiting.md); pause retries on storm detection.
**Owner.** AI platform, sprint N+3.

### COST-INC-011 — Severity: Medium
**Finding.** Quarterly runbook drill not scheduled.
**Recommendation.** Drill per §10.10; identify gaps; fix.
**Owner.** SRE, sprint N+3.

### COST-INC-012 — Severity: Medium
**Finding.** Customer-facing communication for cost incidents undocumented.
**Recommendation.** Template per §7.4.
**Owner.** customer success + SRE, sprint N+3.

### COST-INC-013 — Severity: Medium
**Finding.** Provider release notes not subscribed; pricing changes are surprises.
**Recommendation.** Subscribe + automated rate-table update per §7.6.
**Owner.** AI platform + FinOps, sprint N+4.

### COST-INC-014 — Severity: Medium
**Finding.** PR review for prompts doesn't require cost impact analysis.
**Recommendation.** Cost impact required per §9.7.
**Owner.** AI platform + engineering management, sprint N+4.

### COST-INC-015 — Severity: Low
**Finding.** Action items from past reviews not tracked.
**Recommendation.** Tracker per §8.3; surfaced when overdue.
**Owner.** engineering management, sprint N+5.

### COST-INC-016 — Severity: Low
**Finding.** Cross-team lessons not shared from cost reviews.
**Recommendation.** Newsletter / tech-talk per §8.4.
**Owner.** SRE + engineering management, sprint N+5.

### COST-INC-017 — Severity: Low
**Finding.** Reconciliation drift alert absent.
**Recommendation.** Alert per [cost-dashboards-and-alerts.md §11 finding COST-DASH-017](./cost-dashboards-and-alerts.md).
**Owner.** AI platform + FinOps, sprint N+5.

### COST-INC-018 — Severity: Low
**Finding.** Cost-incident metrics not tracked over time.
**Recommendation.** Per-quarter: incident count, MTTR, avoided cost, prevented recurrence rate.
**Owner.** SRE + FinOps, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Document the seven cost-incident classes (§2).** Familiarize on-call.
- [ ] **Build runbook per class.** Detection signal → triage → mitigation → resolution → prevention.
- [ ] **Build pre-built triage dashboards (§4.2).** Per-feature, per-tenant, per-prompt-version, provider-status.
- [ ] **Implement feature kill switch (§5.3).** Cross-link to circuit-breaker primitive.
- [ ] **Implement per-tenant pause (§5.4).** Self-service for on-call.
- [ ] **Implement mitigation reversal tracking (§10.4).** Incident only closes when mitigation reversed.
- [ ] **Adopt mitigation-first discipline (§1).** Train on-call; reinforce in drills.
- [ ] **Establish post-incident review template (§8.1).** Per-incident; assign action items.
- [ ] **Subscribe to provider release notes (§7.6).** Pricing change advance warning.
- [ ] **Require cost-impact in PR review for prompts (§9.7).** Pre-deploy estimate tool.
- [ ] **Schedule quarterly runbook drills (§10.10).**
- [ ] **Track cost-incident metrics over time.** MTTR; avoided cost; recurrence rate.
- [ ] **Cross-team lessons sharing (§8.4).**

---

## 13. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — telemetry that feeds incident investigation.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) — alerts that page on-call.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — kill switch primitive used in mitigation.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) — per-tenant mitigation context.
- [caching-for-cost.md](./caching-for-cost.md) — caching can be a mitigation lever.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — route-down-tier mitigation.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) — batch as a non-incident cost control.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) — rate limits as mitigation; retry storm protection.
- [finops-process.md](./finops-process.md) — FinOps cadence consumes incident learnings.

**Elsewhere in this repo.**
- [reliability-engineering/timeout-strategy.md](../reliability-engineering/timeout-strategy.md) — timeouts as cost-cap mechanism.
- [reliability-engineering/retry-strategy.md](../reliability-engineering/retry-strategy.md) — retry policy affects cost incidents.
- [reliability-engineering/circuit-breakers.md](../reliability-engineering/circuit-breakers.md) — circuit-breaker primitive used in cost mitigations.
- [reliability-engineering/incident-response-for-ai.md](../reliability-engineering/incident-response-for-ai.md) — general AI incident response framework.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — paging design.
- [observability-and-telemetry/debugging-from-traces.md](../observability-and-telemetry/debugging-from-traces.md) — trace-based investigation.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — rate-limit architecture used in mitigation.
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — tenant-specific incident context.

**External.**
- Google SRE Book — incident response practices; AI overlays apply.
- PagerDuty / Opsgenie incident management documentation.
- Blameless post-mortem literature.
- AWS Well-Architected — incident response practices.
