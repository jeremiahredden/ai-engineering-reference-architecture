# Incident Response for AI

> **Audience.** SREs and engineering leads on-call for AI features. Incident commanders coordinating responses where the AI is the broken thing. Anyone whose general-purpose incident response runbook doesn't quite fit the AI-specific failure modes. **Scope.** The *engineering* practice of incident response for AI workloads: the AI-specific incident classes (cost incident, quality incident, provider outage, model deprecation surprise, multi-tenant incident); response procedures per class; post-incident review templates; integration with the broader incident-response framework. Not the cost-incident runbook specifically (see [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md), companion). Not the general SRE incident response — this is the AI-specific overlay. **Worked client.** Meridian Health.

---

## 1. Why this document exists

General incident response is mature: page on-call, classify, mitigate, investigate, learn. The SRE practice is well-documented in books and standards.

AI incident response has additional classes and challenges:

- **Cost incidents.** Spend climbing fast; user-facing service may still be working. Detection and mitigation different from "service is down."
- **Quality incidents.** HTTP 200 responses; users seeing wrong answers. Detection is hard; mitigation requires investigation; root cause may be a prompt change, model upgrade, or data drift.
- **Provider outages.** External dependency; engineering doesn't control resolution; response is "manage through it."
- **Model deprecation surprise.** Provider silently changed a model behind an alias; quality changes; nobody noticed until a customer complained.
- **Multi-tenant incidents.** One tenant's burst affects others; isolation must work.

Each class requires its own response pattern. A general "incident response" runbook adapted from web-service practice doesn't fully cover these.

This document covers the engineering: the AI-specific incident classes; the response procedures; the integration with the broader response framework.

This document is opinionated about four things:

1. **AI-specific incident classes need their own runbooks.** The general SRE runbook gets you started; AI overlays specialize.
2. **Cost incidents are real incidents.** Treat them with the same rigor as latency or availability incidents. Pages on-call; structured response.
3. **Quality incidents are the hardest to detect.** Without live-judge or quality monitoring, quality regressions can persist for days. Detection infrastructure is the prerequisite.
4. **The post-incident review for AI incidents must include "would we know if this happened again?"** Many AI incidents persist because detection was missing; the review must address detection as much as resolution.

Structure: (2) the AI-specific incident classes; (3) cost incident response; (4) quality incident response; (5) provider outage response; (6) model deprecation surprise response; (7) multi-tenant incident response; (8) post-incident review templates; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The AI-specific incident classes

The classes that warrant their own runbook.

### 2.1 Cost incident

Per [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md): seven sub-classes (runaway, prompt-bloat, retrieval-bloat, agent-loop, abuse, pricing change, retry storm).

**Detection.** Burn-rate alert; spike alert; anomaly alert.

**Severity.** Variable; from $50 to $10k+ impact.

**Response.** Mitigation-first; investigation after. Per cost-incident-runbook §1.

### 2.2 Quality incident

The AI is producing wrong, low-quality, or unsafe responses.

**Detection.** Live-judge quality drop; user-feedback spike; structured-output failures; downstream complaints.

**Severity.** From "users notice" to "user harm."

**Response.** Investigation-first (understand what's wrong before mitigation could make it worse). Per §4.

### 2.3 Provider outage

External provider (Anthropic, OpenAI, Bedrock) is degraded or unavailable.

**Detection.** Provider circuit-breaker opens; provider status page; elevated error rate.

**Severity.** Variable; depends on workload and degree of degradation.

**Response.** Failover (if multi-provider); degraded mode (cross-link); customer communication. Per §5.

### 2.4 Model deprecation surprise

The provider quietly changed a model behind an alias; behavior shifted; nobody knew.

**Detection.** Quality drift correlated with no internal change; provider release notes if subscribed.

**Severity.** Variable; depends on the change.

**Response.** Identify the change; eval against current behavior; decide whether to migrate or pin. Per §6.

### 2.5 Multi-tenant incident

One tenant's behavior is affecting others.

**Detection.** Cross-tenant correlation in dashboards; one tenant's spike + another tenant's degradation.

**Severity.** Variable; SLA implications for affected tenants.

**Response.** Isolate the noisy tenant; restore service for affected. Per §7.

### 2.6 Composite incidents

Sometimes multiple classes simultaneously:

- Provider outage triggers retry storm → cost incident + reliability incident.
- Quality regression triggers user feedback → quality incident + possibly cost (retries).

Handle each class in parallel; coordinate.

### 2.7 The incident-classification at page time

The alert payload should hint at the class:

```
ALERT: Cost burn rate 4.0x (last 1 hour)
Suspected class: cost incident (sub-class: TBD)
Runbook: cost-incident-runbook
```

Or:

```
ALERT: Quality SLO burn rate 3.5x (last 1 hour)
Suspected class: quality incident
Runbook: quality-incident-runbook
```

On-call sees the class before opening the runbook.

---

## 3. Cost incident response

The cost-specific incident response is documented in detail at [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md). Brief summary here for completeness.

### 3.1 Triage

- Per-feature dashboard: which feature?
- Per-tenant dashboard: which tenant?
- Per-prompt-version: recent deploy?
- Per-model: routing change?

### 3.2 Mitigation

- Rate-limit tightening.
- Route-down-tier (cheaper model).
- Kill switch (feature disable).
- Per-tenant pause.
- Retry pause (for retry storm).

### 3.3 Investigation

- Preserve evidence.
- Per-call trace inspection.
- Recent change review.
- 5 whys.

### 3.4 Resolution

- Fix immediate cause.
- Verify mitigation reversal.
- Communicate.
- Post-incident review.

Cross-link to cost-incident-runbook for full procedure.

---

## 4. Quality incident response

The AI is producing wrong responses.

### 4.1 The signal

- Live-judge quality drop (e.g., pass rate dropped from 95% to 78%).
- User feedback spike (thumbs-down rate increasing).
- Schema validation failure rate climbing.
- Downstream consumer complaints ("the AI gave us bad data").
- Specific incident reports (clinician noticed wrong answer).

### 4.2 The classification

Sub-classes:

- **Deploy-induced.** Quality changed after a deploy (cross-link to [cost-incident-runbook.md §9](../cost-and-finops/cost-incident-runbook.md) for the prompt-bloat parallel).
- **Model-version change.** Provider's model behavior shifted (cross-link to §6 deprecation).
- **Data drift.** Tenant's input distribution shifted; previous quality no longer applies.
- **External dependency degraded.** A tool / retrieval source is degraded; the LLM's input is worse.
- **Provider-side issue.** The provider's model is degraded (rare; usually correlated with other signals).

### 4.3 The investigation-first approach

Quality incidents differ from cost: mitigation that's wrong can make quality worse. Investigate before mitigating broadly:

- Pull sample responses from the bad window.
- Compare to baseline responses.
- Identify the failure pattern.

### 4.4 The targeted mitigation

Based on the class:

- Deploy-induced: revert the deploy.
- Model-version change: pin to old version or re-eval and accept.
- Data drift: investigate input source; adjust prompt or retrieval.
- External dependency: failover the dependency; restore the upstream.
- Provider-side: failover or wait.

### 4.5 The "feature disable while investigating" pattern

If quality is severely degraded and users are affected:

- Disable the feature (feature circuit-breaker; cross-link to [circuit-breakers.md §5](./circuit-breakers.md)).
- Serve degraded mode (cross-link to [degraded-mode-design.md](./degraded-mode-design.md)).
- Investigate without time pressure.

### 4.6 The user-facing communication

Quality incidents may require customer-facing communication:

- "We're investigating reports of AI quality issues."
- "Issue identified and resolved."
- "Post-incident: here's what happened and what we're doing."

Honest; sets expectations.

### 4.7 The investigation tools

- Per-feature dashboard.
- Sample response review (random sample from the bad window).
- Eval suite re-run against current production.
- Deploy log review.
- Per-prompt-version comparison.

### 4.8 The recovery validation

After mitigation, verify recovery:

- Live-judge passes resume.
- User feedback returns to baseline.
- Eval suite passes.

Don't close the incident until recovery is verified.

---

## 5. Provider outage response

The external provider is degraded or unavailable.

### 5.1 The detection signals

- Provider's status page.
- Elevated error rate (4xx, 5xx) from the provider.
- Circuit-breaker open (cross-link to [circuit-breakers.md §3](./circuit-breakers.md)).
- Provider's own announcement (Twitter, status, support email).

### 5.2 The triage

- Confirm the outage with provider's status page.
- Identify scope (which models; which regions).
- Identify expected duration.

### 5.3 The response

Per the multi-provider posture (cross-link to [multi-provider-failover.md](./multi-provider-failover.md)):

**If multi-provider failover configured:**
- Automatic or manual failover.
- Verify secondary provider is handling.
- Monitor quality of fallback.

**If single-provider with internal fallback:**
- Provider circuit-breaker engages.
- Internal fallback ladder (smaller model, cached, templated) activates.
- Monitor.

**If single-provider with no fallback:**
- Workload affected; users see degraded service.
- Communicate to customers.

### 5.4 The customer-facing communication

```
Status update:
"Anthropic is experiencing elevated latency on Sonnet model.
We are using our Haiku fallback model for affected requests.
Some responses may be shorter or less detailed than usual.
Updates posted at status.example.com/incident/12345"
```

Standard template; customizable per incident.

### 5.5 The "wait it out" decision

Some incidents resolve before mitigation is fully effective. Decision point:

- Estimated duration < 30 min: ride it out with degraded mode.
- Estimated duration > 30 min: consider provider-specific actions.

The data dictates; on-call decides.

### 5.6 The post-incident reconciliation

After resolution:

- Verify no residual issues.
- Check cost during incident (retries may have inflated).
- Review affected requests.
- Customer follow-up if SLA implications.

---

## 6. Model deprecation surprise response

The provider changed a model; behavior shifted.

### 6.1 The detection (often after the fact)

- Quality drift with no internal change.
- Live-judge metric drop.
- User reports of "the AI seems different."
- Provider release note (if subscribed).

The "surprise" is detection lag; the change happened days/weeks ago.

### 6.2 The investigation

- Subscribe to provider release notes (subscribe ahead of time, not during the incident).
- Check the model alias's underlying version.
- Compare current behavior to baseline.

### 6.3 The classification

- **Major model change.** Provider released a new version of the alias; behavior is materially different.
- **Provider's silent tuning.** Behind-the-scenes adjustments without announcement.
- **Model deprecation pending.** Old model being phased out; alias may be redirecting to newer.

### 6.4 The response

For major model change:
- Eval against current behavior.
- Decide: accept new behavior (new baseline) or pin to old version (if provider supports).
- Communicate to stakeholders.

For provider's silent tuning:
- Adjust prompts to compensate.
- Communicate via release notes if customer-visible.

For deprecation pending:
- Plan migration to the new version.
- Run eval; adjust workflows.

### 6.5 The "pin to specific version" decision

Some providers offer model-version pinning (Anthropic: `claude-sonnet-4-6` vs `claude-3-sonnet`, both specific; vs alias `claude-sonnet-latest`).

Use specific version where stability matters:

- Production workloads: pin to specific version.
- Experimental workloads: alias acceptable.

### 6.6 The provider-side communication

Communicate to provider:
- "We discovered this change; here's the workload impact."
- "We'd like more advance notice for future changes."

Most providers respond to feedback; the squeaky wheel gets the notice.

### 6.7 The prevention

- Subscribe to provider release notes.
- Live-judge monitoring detects drift.
- Catalogue tracks model versions (cross-link to [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md)).
- Pin to specific versions where stability matters.

---

## 7. Multi-tenant incident response

One tenant's behavior affecting others.

### 7.1 The signal

- Cross-tenant correlation: tenant A spikes; tenant B degrades.
- Per-tenant latency dashboard.
- Customer complaints from tenants who shouldn't be affected.

### 7.2 The triage

- Identify the noisy tenant (top spend; top RPM; top TPM).
- Identify the affected tenants.
- Estimate the impact.

### 7.3 The mitigation

- Tighten the noisy tenant's quota.
- Pause the noisy tenant's specific feature.
- Per-tenant kill switch.

### 7.4 The customer communication

Affected tenants:
- "We're investigating elevated latency."
- After mitigation: "Resolved. Caused by another tenant's traffic; isolation now enforced."

Noisy tenant:
- "Your traffic exceeded expected pattern; we've tightened your limits."
- Engagement to understand cause.

### 7.5 The architecture review post-incident

The incident reveals isolation gaps:

- Was the architecture supposed to prevent this?
- Why didn't it?
- What's the fix?

Cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md).

### 7.6 The premium tenant SLA impact

If a premium tenant was affected by another tenant:

- SLA impact assessment.
- Customer-facing communication.
- Credit or remediation if SLA breached.

### 7.7 The "isolation didn't work" post-mortem

Why did the architecture not isolate?

- Per-tenant budget not configured?
- Per-tenant budget too generous?
- Shared resource not isolated (vector store hot key, etc.)?
- Per-tenant budget set in dev but not promoted?

Specific investigation; tighten the controls.

---

## 8. Post-incident review templates

Per incident class, the review template.

### 8.1 The standard structure

```
Incident: <descriptive title>
Date: <YYYY-MM-DD>
Duration: <start> → <end>
Severity: <P0/P1/P2/P3>
Class: <cost / quality / provider / deprecation / multi-tenant / composite>

Summary:
<2-3 sentence executive summary>

Timeline:
<bullet-point timeline; key events; times>

Detection:
- Which alert(s) fired?
- Was detection fast enough?
- Was the signal specific enough?

Triage:
- How long to classify?
- Were dashboards sufficient?
- Did the runbook cover this?

Mitigation:
- What was applied?
- How effective?
- Any collateral damage?

Root cause:
- What was the underlying cause?
- How did it get to production / escape detection?

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

### 8.2 The class-specific extensions

For cost incidents:
- Financial impact (specific dollars).
- Cost during incident vs baseline.
- Was the circuit-breaker effective?

For quality incidents:
- Quality metric trend.
- Sample of affected responses.
- Customer impact estimate.

For provider outages:
- Did failover work?
- Was customer communication adequate?
- What did we learn about provider?

For deprecation surprises:
- When did the provider change happen?
- When did we detect?
- Why didn't we catch earlier?

For multi-tenant incidents:
- Was the architecture supposed to prevent this?
- Per-tenant data; affected vs unaffected.
- Architecture changes needed.

### 8.3 The "would we know if this happened again" question

For every incident:

- Is detection sufficient?
- Would the alert fire on a similar issue?
- What signal is currently missing?

Often the answer suggests a new alert; track it.

### 8.4 The "what was the customer impact" assessment

- How many customers affected?
- How visible was the impact?
- Did anyone churn or complain?
- Support tickets generated?

The customer impact is the bottom-line measure of incident severity.

### 8.5 The cross-team learning

Lessons relevant to other teams:

- Newsletter post.
- Tech-talk presentation.
- Update to general runbook.

The team that handled the incident becomes a resource for future similar incidents.

### 8.6 The action item closure

Every review produces action items:

- Owner.
- Due date.
- Status (tracked).

Overdue items reviewed; not allowed to languish.

### 8.7 The "we've had this incident before" pattern recognition

If the same incident keeps recurring:

- Root cause may be deeper than the immediate fix suggests.
- Process change rather than just engineering fix.

Pattern recognition across incidents.

---

## 9. Worked Meridian example

Meridian's incident response evolved over multiple production cycles.

### 9.1 The incident catalog (Q1-Q2 2026)

```
Date         Class           Duration  Severity  Impact
─────────────────────────────────────────────────────────────────
Q1-2026-01   Cost            1h 37m   P2        $320 / 0 customers visible (caught early)
Q1-2026-02   Provider outage  12 min   P2        Patient API chat degraded; no customer report
Q1-2026-03   Quality (deploy) 1h 18m   P3        Care Coordinator quality regression; caught by judge
Q2-2026-01   Multi-tenant     2h 30m   P2        External eligibility tool down; degraded
Q2-2026-02   Cost (free tier) 18 min   P3        Free-tier runaway agent capped at $4.30
Q2-2026-03   Quality (refactor) 4h     P2        Patient API chat refactor; semantic cache miss-config; caught by drift
```

### 9.2 The Q1 cost incident in detail

(Cross-link to [cost-incident-runbook.md §9](../cost-and-finops/cost-incident-runbook.md) for the full story.)

- 14:23: deploy of Care Coordinator prompt v37.
- 15:51: alert fires.
- 15:59: revert PR opened.
- 16:00: revert deployed.
- 16:03: baseline restored.

Total duration: 1h 37m. Cost impact $320.

Post-incident: pre-deploy cost-impact check added (cross-link to [finops-process.md §5](../cost-and-finops/finops-process.md)).

### 9.3 The Q1 quality incident in detail

A separate quality regression (different from the cost one):

- Day 0: Care Coordinator prompt change ships at 14:00.
- 14:00-15:30: live-judge sees gradual quality drop (95% → 78%).
- 15:55: quality circuit-breaker fires.
- 16:00: degraded mode (cached responses, structured "limited mode" indicator).
- 16:20: deploy identified.
- 16:30: revert deployed.
- 17:30: quality recovered.

Total: 3h 30m from deploy to recovery.

Customer impact: ~12 clinicians saw "limited mode" indicator; no support tickets (the degraded mode handled gracefully).

Prevention: prompt-change review now requires eval suite pass before deploy.

### 9.4 The Q1 provider outage in detail

Anthropic 12-min degradation:

- Provider circuit-breaker engaged at 09:14.
- Patient API chat fell back to Haiku.
- Care Coordinator queued in-flight tasks.
- Document classification queued.
- At 09:26, provider recovered; breaker closed.

Customer impact: minimal. Patient API customers saw normal responses (opaque degraded mode); Care Coordinator's clinician users saw "Quick mode" indicator briefly.

Lessons:
- Internal fallback was sufficient.
- Multi-provider would have added complexity without proportionate benefit.

### 9.5 The Q2 multi-tenant incident

External eligibility-check API was down for ~2.5 hours:

- Tool circuit-breaker engaged at 09:16.
- Care Coordinator agent continued with "eligibility TBD" markers.
- 140 tasks completed during the outage with marked eligibility.

Customer impact: limited. Atlantic Maple's clinicians worked normally; eligibility verification deferred.

After recovery: batch eligibility check on the marked tasks.

Lessons:
- Tool circuit-breaker + degraded mode handled gracefully.
- The architecture is right; no changes needed.

### 9.6 The Q1 free-tier runaway

Free-tier tenant's misconfigured agent recursively loop'd:

- 4-hour duration before tenant noticed.
- Per-tenant cost cap kept impact at $4.30.
- Operator alerted at 80% of budget.
- Cap enforced at 100%.

Tenant fixed their agent; per-tenant isolation worked.

### 9.7 The Q2 quality refactor incident

A platform refactor changed how semantic cache was used; cache keys didn't match new pattern; cache miss rate jumped from 30% to 95%.

- Patient API chat cost rose ~3x for ~4 hours.
- Live-judge quality also drifted (cache hit was masking model quality issue that surfaced when cache missed).
- Alert fired at 4 hours when cost burn was severe.

Mitigation: hotfix to cache key configuration. Recovery in ~10 minutes.

Lessons:
- Refactors should include test of cache configuration.
- Faster cache-hit-rate alert (caught at 4 hours; should have caught at 1 hour).

### 9.8 The composite Q1 incident

The Q1 cost incident and Q1 quality incident were related:

- Same deploy caused both.
- Different alerts fired (cost burn + quality SLO).
- Both were tracked as separate incidents (different runbooks, different post-mortems).

Lessons:
- Cost incident response was faster (well-tuned dashboards).
- Quality incident response was slower (judge cycle had latency).

### 9.9 The post-incident reviews

For each incident:

- Lightweight (P3 or contained): 1-hour review; documented.
- Heavy (P2 or higher): 60-90 min review; cross-team; action items.

Meridian's standard review template applied; action items tracked; closure rate ~90%.

### 9.10 What the discipline produces

- MTTR median: 25 minutes (provider outages); 35 minutes (cost); 60 minutes (quality).
- Customer-visible incidents: rare (~1 per quarter).
- Recurrence rate: low (action items prevent repeat).
- Engineering knowledge of AI incidents: high; new engineers ramp via incident retrospectives.

### 9.11 The runbook library

Meridian maintains runbooks per class:
- cost-incident-runbook.md
- quality-incident-runbook.md
- provider-outage-runbook.md
- model-deprecation-runbook.md
- multi-tenant-incident-runbook.md
- general-ai-incident-runbook.md (umbrella; routes to specific)

Each updated after relevant incidents; quarterly drill exercises.

### 9.12 The lessons learned over time

- Detection is the biggest lever; alerts that fire too late cost most.
- Cost incidents and quality incidents are different; different runbooks.
- Provider outages: internal fallback usually sufficient.
- Multi-tenant incidents: architecture works when configured; verify config.
- Post-mortem culture: blameless; action-item driven; repeats are the signal.

---

## 10. Anti-patterns

### 10.1 The single "AI incident" runbook

**Pattern.** One runbook for all AI incidents. On-call has to read 5 pages to find the right path. Slow response.

**Corrective.** Per-class runbooks per §2.

### 10.2 The "investigate first" cost incident

**Pattern.** Cost incident; investigation precedes mitigation; spend continues during 30-minute investigation. 2x-3x the financial impact.

**Corrective.** Mitigation-first per [cost-incident-runbook.md §1](../cost-and-finops/cost-incident-runbook.md).

### 10.3 The quality incident with no detection

**Pattern.** Live-judge isn't implemented; quality incidents are detected via customer reports. Days to first signal.

**Corrective.** Live-judge per [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md) + quality SLO per [fault-budgets-for-ai.md](./fault-budgets-for-ai.md).

### 10.4 The provider outage without communication

**Pattern.** Provider is down; engineering is mitigating; customers aren't told. Support tickets pile up.

**Corrective.** Customer communication per §5.4.

### 10.5 The model deprecation surprise

**Pattern.** Provider changes model; no subscription to release notes; no live-judge; quality drifts undetected for weeks.

**Corrective.** Subscribe to release notes; live-judge; pin to specific versions where stability matters.

### 10.6 The multi-tenant incident hidden by aggregate metrics

**Pattern.** Aggregate latency / cost looks normal; one tenant's experience is bad; aggregate hides it.

**Corrective.** Per-tenant metrics per [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md).

### 10.7 The post-mortem without action items

**Pattern.** Post-incident review held; lessons discussed; no action items; same incident recurs.

**Corrective.** Action items with owners + due dates per §8.6.

### 10.8 The "incident commander" without clear authority

**Pattern.** Multiple engineers on the incident; no clear lead; coordination chaos.

**Corrective.** Incident commander role per standard SRE practice; clear roles.

### 10.9 The runbook that's three years old

**Pattern.** Runbook exists; references retired dashboards; never used; ignored when needed.

**Corrective.** Quarterly review of runbooks; update with each significant incident.

### 10.10 The "we've had this before" without root-cause investigation

**Pattern.** Same incident pattern recurs; each instance handled as new; root cause is never identified.

**Corrective.** Pattern recognition across incidents per §8.7; root cause as separate investigation.

---

## 11. Findings (sprint-assignable)

### REL-INC-001 — Severity: Critical
**Finding.** No per-class runbooks for AI incidents.
**Recommendation.** Per §2; runbook per class.
**Owner.** SRE + AI platform, sprint N+1.

### REL-INC-002 — Severity: Critical
**Finding.** Quality incident detection absent.
**Recommendation.** Live-judge + quality SLO per [fault-budgets-for-ai.md](./fault-budgets-for-ai.md).
**Owner.** AI platform + eval, sprint N+1.

### REL-INC-003 — Severity: Critical
**Finding.** Cost incident response doesn't prioritize mitigation.
**Recommendation.** Mitigation-first per [cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md).
**Owner.** SRE + AI platform, sprint N+1.

### REL-INC-004 — Severity: High
**Finding.** Customer communication for incidents undocumented.
**Recommendation.** Templates per §5.4 and §7.4.
**Owner.** customer success + SRE, sprint N+2.

### REL-INC-005 — Severity: High
**Finding.** Provider release notes not subscribed.
**Recommendation.** Subscription + automated check per §6.7.
**Owner.** AI platform, sprint N+2.

### REL-INC-006 — Severity: High
**Finding.** Per-tenant metrics absent; multi-tenant incidents invisible.
**Recommendation.** Per-tenant dashboard per [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md).
**Owner.** observability-eng + AI platform, sprint N+2.

### REL-INC-007 — Severity: High
**Finding.** Post-incident review without action items closure.
**Recommendation.** Action item tracker per §8.6.
**Owner.** SRE + engineering management, sprint N+2.

### REL-INC-008 — Severity: High
**Finding.** Model version not pinned for production workloads.
**Recommendation.** Pin per §6.5; model catalogue tracks (cross-link to architecture sibling).
**Owner.** AI platform, sprint N+2.

### REL-INC-009 — Severity: Medium
**Finding.** Cross-team incident lessons not shared.
**Recommendation.** Per §8.5.
**Owner.** SRE + engineering management, sprint N+3.

### REL-INC-010 — Severity: Medium
**Finding.** Runbook drill not scheduled.
**Recommendation.** Quarterly drill per §9.11; verify runbooks current.
**Owner.** SRE, sprint N+3.

### REL-INC-011 — Severity: Medium
**Finding.** Pattern recognition across incidents absent.
**Recommendation.** Per §8.7.
**Owner.** SRE, sprint N+3.

### REL-INC-012 — Severity: Medium
**Finding.** Incident commander role undefined.
**Recommendation.** Per §10.8; standard practice.
**Owner.** SRE, sprint N+3.

### REL-INC-013 — Severity: Medium
**Finding.** Incident metrics not tracked over time.
**Recommendation.** Per §9.10; MTTR + customer impact + recurrence.
**Owner.** SRE, sprint N+4.

### REL-INC-014 — Severity: Medium
**Finding.** Composite incidents handled as single incident.
**Recommendation.** Treat each class separately per §2.6.
**Owner.** SRE, sprint N+4.

### REL-INC-015 — Severity: Low
**Finding.** Lightweight vs heavyweight review distinction unclear.
**Recommendation.** Per §9.9; document criteria.
**Owner.** SRE, sprint N+5.

### REL-INC-016 — Severity: Low
**Finding.** Provider-side issues sometimes counted as our incidents.
**Recommendation.** Classify provider-side separately; document per [fault-budgets-for-ai.md §8.5](./fault-budgets-for-ai.md).
**Owner.** SRE, sprint N+5.

### REL-INC-017 — Severity: Low
**Finding.** Customer-facing SLA breaches not tracked.
**Recommendation.** SLA breach tracking; credit / remediation process.
**Owner.** customer success + engineering management, sprint N+6.

### REL-INC-018 — Severity: Low
**Finding.** Annual incident retrospective not held.
**Recommendation.** Annual review of incident patterns; trends.
**Owner.** engineering management, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Define AI-specific incident classes per §2.**
- [ ] **Build per-class runbook (§3, §4, §5, §6, §7).**
- [ ] **Implement detection per class (alerts).**
- [ ] **Document customer communication templates (§5.4, §7.4).**
- [ ] **Subscribe to provider release notes (§6.7).**
- [ ] **Set up per-tenant metrics for multi-tenant incidents (§7.1).**
- [ ] **Define incident commander role.**
- [ ] **Adopt post-incident review template (§8.1).**
- [ ] **Track action items from reviews (§8.6).**
- [ ] **Quarterly runbook drills (§9.11).**
- [ ] **Track incident metrics over time (§9.10).**
- [ ] **Annual incident retrospective.**

---

## 13. References

**In this folder.**
- [timeout-strategy.md](./timeout-strategy.md) — timeouts that may surface in incidents.
- [retry-strategy.md](./retry-strategy.md) — retry behavior during incidents.
- [fallback-patterns.md](./fallback-patterns.md) — fallback paths during incidents.
- [circuit-breakers.md](./circuit-breakers.md) — breakers that engage.
- [degraded-mode-design.md](./degraded-mode-design.md) — degraded mode during incidents.
- [fault-budgets-for-ai.md](./fault-budgets-for-ai.md) — SLOs that incidents affect.
- [capacity-planning.md](./capacity-planning.md) — capacity issues that become incidents.
- [multi-provider-failover.md](./multi-provider-failover.md) — failover during provider outages.

**Elsewhere in this repo.**
- [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md) — cost incident response (companion).
- [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md) — alerting.
- [observability-and-telemetry/debugging-from-traces.md](../observability-and-telemetry/debugging-from-traces.md) — investigation tools.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — quality drift signal.
- [observability-and-telemetry/alerting-and-paging-design.md](../observability-and-telemetry/alerting-and-paging-design.md) — alert design.

**Sibling repos.**
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / noisy-neighbor-mitigation.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/noisy-neighbor-mitigation.md) — architectural context for multi-tenant incidents.
- [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md) — catalogue tracks deprecations.

**External.**
- Google SRE Book — incident response practices.
- PagerDuty / Opsgenie incident management.
- Blameless post-mortem literature.
- AWS Well-Architected — incident response.
