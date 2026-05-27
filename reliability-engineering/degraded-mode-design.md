# Degraded Mode Design

> **Audience.** Engineers whose AI feature fails outright when the provider is degraded. Tech leads whose post-incident review repeatedly identifies "we could have served a cached answer instead of failing." Anyone who has heard "the AI is broken" from a customer when a graceful degradation could have made the failure invisible. **Scope.** The *engineering* practice of degraded-mode design for AI: the degraded-mode catalog (cached, smaller model, simpler prompt, no agent, structured error, refusal-with-context); the triggers (cost budget, error rate, latency, provider outage, quality regression); UX patterns (transparent, opaque, refusal); per-feature design; testing degraded mode in production. Not the failure conditions that trigger it (see [timeout-strategy.md](./timeout-strategy.md), [retry-strategy.md](./retry-strategy.md), [circuit-breakers.md](./circuit-breakers.md), companions). Not the fallback ladder (see [fallback-patterns.md](./fallback-patterns.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Degraded mode is the most underused reliability pattern in AI engineering. Most AI features have two states: working (full quality) or failed (nothing). The middle ground — "still working, but at reduced quality" — is rarely engineered, despite being the right answer for almost every user-facing AI feature.

The pattern from web services: a service that's healthy serves full content; a service that's slow serves cached / static content; a service that's down serves a "maintenance" page. The user's experience degrades but doesn't break.

For AI features, the same logic applies — and the levers are richer:

- **Cached response.** Previous answer to the same question.
- **Smaller model.** Haiku instead of Sonnet; faster + cheaper + lower quality but still useful.
- **Simpler prompt.** Drop the few-shot examples; drop the retrieval; deliver a faster but less informed answer.
- **No agent.** Direct LLM call instead of agent loop.
- **Structured error.** Honest "this feature is temporarily limited" message.
- **Refusal-with-context.** "I can't help right now; here's what you might do instead."

Each is a legitimate response when the full path is unavailable. Each is better than "the AI is broken."

The architectural failure mode this document addresses: shipping AI features with no degraded mode. The first provider incident produces a customer-facing outage that didn't have to happen.

This document is opinionated about four things:

1. **Every AI feature must have a degraded mode.** Not "should consider"; must. It's a launch-readiness requirement, not a future-improvement.
2. **The degraded mode must be testable in production.** A degraded mode that's never been exercised may not actually work. Periodic synthetic triggers verify.
3. **The user should know when they're in degraded mode (usually).** Honest UX builds trust; silent degradation erodes it. Some workloads need silent degradation; default to honest.
4. **Different triggers may invoke different degraded modes.** Cost trigger → cheaper model. Provider trigger → cached / fallback model. Quality trigger → structured error. The mapping is per-feature.

Structure: (2) the degraded-mode catalog; (3) the triggers; (4) the triggers-to-actions matrix; (5) UX patterns; (6) per-feature design; (7) testing in production; (8) observability; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The degraded-mode catalog

The set of degraded modes available. Each has cost, quality, latency, and operational properties.

### 2.1 Cached response

Return a previously-generated response for the same or similar query.

**Cost.** Near-zero (cache lookup).
**Quality.** Equal to whatever the cached response's quality was.
**Latency.** Sub-millisecond.
**Best for.** Workloads with high query overlap (FAQ, common search).

**Implementation.** Cross-link to [cost-and-finops/caching-for-cost.md](../cost-and-finops/caching-for-cost.md). Response cache + semantic cache.

**Pitfall.** Cache may be empty for the specific query. Need a deeper fallback.

### 2.2 Smaller model

Switch from primary (e.g., Sonnet) to cheaper alternative (e.g., Haiku).

**Cost.** 10-15x less than primary.
**Quality.** Lower; usually still acceptable for the workload.
**Latency.** ~30-50% faster than primary.
**Best for.** Workloads where quality difference is manageable (chat, generic Q&A, summarization).

**Pitfall.** Quality difference is workload-specific; eval per workload. Some workloads degrade unacceptably on the smaller model.

### 2.3 Simpler prompt (less context)

Same model, less prompt:

- Drop few-shot examples.
- Drop retrieved context (or use less).
- Drop persona overlay.

**Cost.** Lower (fewer input tokens).
**Quality.** Lower (less context).
**Latency.** Faster.
**Best for.** Workloads where the prompt overhead is large.

**Pitfall.** May break workloads that depend on the dropped context.

### 2.4 No agent (single LLM call)

For agent-based workloads, fall back to direct LLM call without the agent loop:

**Cost.** Much lower (1 call vs N).
**Quality.** Reduced; no tool use; no multi-step reasoning.
**Latency.** Much faster.
**Best for.** Agent features whose questions are sometimes answerable without tools.

**Pitfall.** Some agent workloads can't function without tools.

### 2.5 Static / templated response

Pre-defined responses for known query classes:

- "I can't access that information right now; please try again later."
- "Account balance unavailable; please check your statement."
- Templated greeting / closing.

**Cost.** Zero.
**Quality.** Lowest substantive (it's just templated).
**Latency.** Instant.
**Best for.** Last-resort fallback; specific known queries; greeting / boilerplate.

### 2.6 Refusal with context

"I can't answer that right now, but here's what you might do":

```
"I'm having trouble accessing the clinical reference materials I need
to give you a complete answer. You might try:
- Checking the patient's chart directly
- Consulting with the on-call physician
- Calling clinical informatics at extension 4123"
```

**Cost.** Low (short response).
**Quality.** Honest; useful guidance.
**Latency.** Fast.
**Best for.** When the full feature can't deliver but the user shouldn't be left empty-handed.

### 2.7 Structured error

```json
{
  "status": "degraded",
  "reason": "primary model unavailable",
  "user_message": "I'm temporarily limited; please retry in a moment.",
  "alternative_actions": [...],
  "incident_id": "..."
}
```

**Cost.** Zero.
**Quality.** Lowest in terms of user value.
**Latency.** Instant.
**Best for.** When no degraded path works; honest signaling.

### 2.8 Feature disable (kill switch)

The feature is unavailable. UI reflects this:

- Feature button is grayed out.
- "Service maintenance" message.
- Other features still work.

**Cost.** Zero.
**Quality.** None (feature not running).
**Latency.** N/A.
**Best for.** Major incidents; cost overruns requiring complete halt; safety-critical scenarios.

### 2.9 The catalog summary

| Mode | Cost | Quality | Latency | Best for |
| --- | --- | --- | --- | --- |
| Cached | Near-zero | Original quality | Sub-ms | High-overlap workloads |
| Smaller model | 10-15x less | Lower | Faster | Most workloads |
| Simpler prompt | Lower | Lower | Faster | Context-heavy workloads |
| No agent | Much less | Reduced | Much faster | Agent workloads |
| Templated | Zero | Boilerplate | Instant | Known queries |
| Refusal-with-context | Low | Honest | Fast | "Can't help full" |
| Structured error | Zero | None | Instant | Last resort |
| Feature disabled | Zero | None | N/A | Major incidents |

### 2.10 The cascading degradation ladder

Most features use multiple modes in cascade:

```
Primary path fails →
  Try smaller model →
    Try cached response →
      Try templated response →
        Return structured error
```

Each step is cheaper / lower quality but more likely to succeed. The first that succeeds returns to user.

---

## 3. The triggers

What conditions invoke degraded mode.

### 3.1 Provider outage / error rate

Provider returning 5xx errors at elevated rate, or 429s, or timeout failures.

**Detection.** Provider circuit-breaker open (cross-link to [circuit-breakers.md](./circuit-breakers.md)).

**Action.** Fall back per cascade.

### 3.2 Elevated latency

Response time is high; SLO at risk.

**Detection.** P99 latency metric exceeds threshold over window.

**Action.** Switch to faster model (smaller) or skip optional context.

### 3.3 Cost budget approached / exceeded

Per-feature or per-tenant budget at warning (80%) or hard limit (100%).

**Detection.** Cost-budget circuit-breaker (cross-link to [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)).

**Action.** Switch to cheaper model; serve cached; skip expensive paths.

### 3.4 Quality regression

Live-judge quality drops below threshold.

**Detection.** Quality circuit-breaker (cross-link to [circuit-breakers.md §5](./circuit-breakers.md)).

**Action.** Disable feature or serve cached / templated until investigation.

### 3.5 Rate-limit exhaustion

Per-tenant rate-limit hit; new calls would 429.

**Detection.** Per-tenant rate-limiter signal.

**Action.** Queue (if latency-tolerant), serve cached (if appropriate), or return 429 to caller.

### 3.6 Self-hosted GPU saturation

Self-hosted inference cluster at capacity.

**Detection.** GPU queue depth or latency metric.

**Action.** Route to hosted provider as overflow, or shed-load.

### 3.7 Multiple triggers

Sometimes triggers compose:

- Provider degraded + cost budget tight → smaller model (cost-effective fallback).
- Cost budget exhausted + quality fine → templated response (preserve cost; sacrifice quality).
- Provider degraded + cost OK → standard cascade.

The action depends on which triggers are active. Per-trigger-combination policy is documented.

---

## 4. The triggers-to-actions matrix

Map triggers to the right degraded action per feature.

### 4.1 The matrix structure

```yaml
feature: care-coordinator
trigger_responses:
  provider_outage:
    primary_action: switch_model_haiku
    secondary_action: cached_or_templated
    user_visibility: subtle_indicator
    
  cost_budget_exceeded:
    primary_action: switch_model_haiku
    secondary_action: feature_disable
    user_visibility: structured_error
    
  quality_regression:
    primary_action: feature_disable
    secondary_action: none
    user_visibility: structured_error
    
  rate_limit_exhausted:
    primary_action: queue
    secondary_action: cached_or_templated
    user_visibility: subtle_indicator
```

Per-feature; per-trigger; explicit action.

### 4.2 The "primary fails; try secondary"

Each trigger has a primary degraded action. If that fails (e.g., smaller model also unavailable; cache miss), the secondary action applies. If both fail, feature_disable or structured_error.

### 4.3 The decision flow

```
Request arrives →
  Check triggers (provider, cost, quality, rate, capacity):
    None active → Primary path.
    Provider outage → Switch model (or cached).
    Cost exceeded → Cheaper model (or disable).
    Quality regression → Disable (or cache).
    Rate exhausted → Queue (or cached).
  Action fails → Try secondary.
  Secondary fails → Disable / error.
```

The pre-call check (cross-link to [cost-and-finops/cost-aware-rate-limiting.md §7](../cost-and-finops/cost-aware-rate-limiting.md)) integrates with the trigger check.

### 4.4 The "multiple triggers active" tie-break

When multiple triggers active, prioritize:

- Quality regression → most severe; disable.
- Cost exhaustion → economically severe; degrade aggressively.
- Provider outage → fall back model.
- Rate limit → queue or cache.

Decide per-feature; document.

### 4.5 The composability check

The matrix should be exhaustive:

- For each combination of trigger states, what's the action?
- Are there combinations that aren't handled?

A simple table works; document the matrix per feature.

---

## 5. UX patterns

How the user experiences degraded mode.

### 5.1 The transparent pattern

User sees a clear indicator that they're in degraded mode:

```
"⚠️ Limited mode: AI capacity is temporarily reduced.
You may receive shorter or less detailed responses."
```

**Pros.** Honest; sets expectations; builds trust on recovery.
**Cons.** Some UX cost.
**Best for.** Most user-facing AI features.

### 5.2 The subtle indicator pattern

Visual indicator that something's different, but minimal:

- Small icon next to responses.
- Slightly different color scheme.
- "Quick mode" or "draft mode" label.

**Pros.** Less alarming than full transparency; user can investigate if curious.
**Cons.** Easy to overlook.
**Best for.** Features where full transparency is too alarming.

### 5.3 The opaque pattern

User is unaware they're in degraded mode. Response delivered without indication.

**Pros.** No UX cost.
**Cons.** Loss of trust if discovered; quality drift can go unnoticed.
**Best for.** When degradation is minimal (e.g., faster model with comparable quality) and transparency would be more alarming than helpful.

### 5.4 The "explain why" pattern

```
"I'm working with limited context right now (some background data
is temporarily unavailable). Here's my best response based on what
I can see..."
```

**Pros.** Explains the limitation honestly; sets context.
**Cons.** More words.
**Best for.** Workflows where the user benefits from knowing the limitation.

### 5.5 The "alternative action" pattern

```
"I can't help with that right now. You might:
- Try again in a few minutes
- Contact support
- See the FAQ at link"
```

**Pros.** User has something to do.
**Cons.** Implies failure; UX impact.
**Best for.** When refusal is the right answer.

### 5.6 The per-feature UX choice

Different features warrant different UX:

- Clinical decision support: full transparency (safety).
- Patient chat: subtle indicator (UX-sensitive).
- Internal admin tool: opaque (low stakes).
- Long-running agent: explain delay if degraded.

Decide per feature; document.

### 5.7 The "what to do if it's wrong" pattern

When degraded, the response may be lower quality. Include feedback / report mechanism:

- "Was this response helpful? [Yes / No]"
- "Report a problem with this response"

User-feedback signals also inform whether the degraded mode is producing acceptable quality.

---

## 6. Per-feature design

The design discipline per AI feature.

### 6.1 The launch-readiness checklist

For each AI feature at launch:

```
[ ] Degraded mode designed.
[ ] Triggers identified (which conditions invoke degraded mode).
[ ] Action per trigger documented.
[ ] UX pattern decided.
[ ] Tested in pre-production.
[ ] Synthetic trigger schedule documented.
[ ] Observability for degraded mode use.
```

Cross-link to [cost-and-finops/finops-process.md §5](../cost-and-finops/finops-process.md) (launch-readiness cost gate).

### 6.2 The design document

For each feature:

```yaml
feature: care-coordinator
primary_path:
  model: claude-sonnet-4-6
  context: full retrieval + history
  tools: 12 enabled
  
degraded_modes:
  level_1:
    trigger: provider outage on primary
    action: switch_to_haiku
    user_visibility: subtle_indicator
    
  level_2:
    trigger: secondary also degraded
    action: cached_response (last 7 days)
    user_visibility: explain_why
    
  level_3:
    trigger: cache miss + provider degraded
    action: refusal_with_context
    user_visibility: alternative_action
    
  level_4:
    trigger: cost_budget_exhausted
    action: feature_disable
    user_visibility: structured_error
```

Document per feature; reviewed; tested.

### 6.3 The design review

Before launch:

- Feature owner walks through the design.
- Reviewers (platform, UX, customer success) comment.
- Approval before launch.

Cross-functional review catches gaps.

### 6.4 The "this feature can't degrade" decision

Some features can't have a meaningful degraded mode:

- Highly safety-critical clinical workflow: degraded → wrong answer → patient risk.
- Real-time fraud detection: degraded → missed fraud.

For these, the "degraded mode" may be "feature unavailable; escalate to human":

- Refuse to serve.
- Escalate to a human operator.
- Inform the requestor explicitly.

Better than serving a degraded answer that misleads.

### 6.5 The "we don't need degraded mode for this internal tool" exception

Internal tools may not warrant degraded mode:

- Engineering admin copilot used by 5 engineers.
- Low cost; low impact if unavailable.

For these, structured-error is acceptable; full degraded-mode design is overkill.

The decision is made explicitly during design.

### 6.6 The evolution as the feature scales

Initial launch with simple degraded mode; refined over time:

- First version: cached or structured error.
- After 6 months: smaller model fallback.
- After 12 months: refined per-trigger policy.

Iterative; informed by production incidents.

---

## 7. Testing degraded mode in production

A degraded mode that's never tested may not work.

### 7.1 The synthetic trigger schedule

Periodically (weekly to monthly), trigger degraded mode synthetically:

- Force the provider breaker open for 5 minutes.
- Inject cost-budget exhaustion for one test tenant.
- Verify the degraded mode activates.
- Verify the user-facing UX renders correctly.
- Verify metrics record the event.

### 7.2 The chaos engineering pattern

For mature platforms, a more rigorous chaos schedule:

- Randomly inject failures.
- Verify the system handles them gracefully.
- Capture and analyze any unexpected behavior.

Each failure becomes a learning opportunity.

### 7.3 The pre-launch test

Before any new feature launches:

- Synthetic trigger of each degraded mode.
- Verification of UX.
- Observability validation.

Caught issues fixed before launch.

### 7.4 The post-incident exercise

After each real incident:

- Did degraded mode activate as expected?
- Did the UX work?
- Did observability capture the event?
- Updates to design as needed.

Incidents are unintentional tests; gather the data.

### 7.5 The "degraded mode that no one's ever used" red flag

A degraded mode that's never been triggered (real or synthetic) may have bugs:

- The fallback model isn't actually configured.
- The UX renders incorrectly.
- The metrics don't fire.

Audit annually; identify untested modes; trigger them.

### 7.6 The "degraded mode test in customer environment" caution

Synthetic triggers in production affect real users. Mitigations:

- Schedule during low-traffic windows.
- Synthetic load for a small fraction of traffic (canary).
- Communicate to customer success in advance.
- Roll back immediately if customer-visible issue.

### 7.7 The test-degradation metric

Track the fraction of synthetic triggers that succeed:

```
metric: degraded_mode_test_pass_rate
target: 95%+
```

Failures indicate design or implementation issues; address.

---

## 8. Observability

How to know when degraded mode fires.

### 8.1 The metrics

Per-feature per-degraded-mode:

- Count of activations (last day, last week).
- Average duration per activation.
- Trigger breakdown (provider / cost / quality / rate / capacity).

### 8.2 The dashboard

A "degraded modes" dashboard:

- Current state per feature (normal / degraded).
- Recent transitions.
- Trigger breakdown.

When something is degraded, the dashboard shows it.

### 8.3 The alerts

- New activation: notification (not page) when degraded mode triggers.
- Extended duration: page when degraded mode persists > N minutes.
- Unusual frequency: alert when degraded mode fires > N times in an hour (something underlying is wrong).

### 8.4 The user-facing visibility

If the UX is transparent (cross-link to §5.1), end-users see the indicator. The product team can survey users on the experience.

### 8.5 The post-mortem data

After each significant degraded-mode period:

- Why it triggered.
- How long it lasted.
- How users experienced it.
- What worked; what could improve.

### 8.6 The "degraded mode prevented an incident" credit

When degraded mode catches what would have been a customer-facing outage:

- The post-incident review credits the design.
- The team is encouraged.
- Other features adopt the pattern.

### 8.7 The Care Coordinator Q1 2026 metric

The Anthropic incident (cross-link to [retry-strategy.md §9.3](./retry-strategy.md)):

- Patient API chat in degraded mode (Haiku fallback) for 12 minutes.
- ~3,200 users served degraded responses.
- Customer support tickets: 4 (very low; degraded mode worked).
- Without degraded mode: estimated ~3,200 customers would have seen errors; support tickets would have been ~150.

---

## 9. Worked Meridian example

Meridian's degraded-mode design per feature.

### 9.1 The Care Coordinator design

```yaml
feature: care-coordinator
primary_path:
  model: claude-sonnet-4-6
  context: full retrieval + conversation history
  tools: 12 enabled (incl. external eligibility check)
  agent_loop: enabled

degraded_modes:
  level_1: # Provider degradation
    trigger: anthropic:sonnet:circuit-breaker:open
    action: switch_to_anthropic_haiku
    user_visibility: subtle_indicator ("Quick mode")
    quality_impact: ~5% lower per eval
    
  level_2: # Both Anthropic models degraded
    trigger: anthropic:sonnet AND anthropic:haiku breakers open
    action: serve_cached_or_refuse
    user_visibility: explain_why
    quality_impact: cached if available; refusal otherwise
    
  level_3: # External eligibility tool unavailable
    trigger: tool:external_eligibility_check:breaker:open
    action: continue_without_eligibility_check, mark_TBD
    user_visibility: subtle_indicator ("Eligibility check pending")
    quality_impact: minor
    
  level_4: # Cost budget exhausted
    trigger: feature:care-coordinator:cost-budget:exhausted
    action: escalate_to_engineering_lead (premium tier)
    user_visibility: structured_error
    quality_impact: feature unavailable
    
  level_5: # Quality regression
    trigger: feature:care-coordinator:quality:regression
    action: feature_disable_pending_investigation
    user_visibility: structured_error
    quality_impact: feature unavailable
```

### 9.2 The Patient API chat design

```yaml
feature: patient-api-chat
primary_path:
  model: claude-sonnet-4-6
  context: tenant overlay + retrieval
  
degraded_modes:
  level_1:
    trigger: provider degradation
    action: switch_to_haiku
    user_visibility: opaque (no indicator)
    quality_impact: ~5% lower
    
  level_2:
    trigger: both models degraded
    action: cached_response_or_templated
    user_visibility: explain_why
    quality_impact: limited; fallback to templated
    
  level_3:
    trigger: cost_budget_exhausted (per tenant)
    action: rate_limit_429_with_advice
    user_visibility: structured_error
    quality_impact: feature unavailable for tenant
```

### 9.3 The clinical decision support exception

For clinical decision support, no degraded mode for quality reasons:

```yaml
feature: clinical-decision-support
primary_path:
  model: claude-sonnet-4-6
  
degraded_modes:
  any_failure:
    action: escalate_to_human_clinician
    user_visibility: explicit_refusal_with_context
```

Safety-critical workflows refuse rather than degrade.

### 9.4 The Q1 2026 Anthropic incident

Provider had 12 minutes of degradation.

- Care Coordinator: switched to Haiku for new tasks; in-flight tasks paused per workflow.
- Patient API chat: switched to Haiku; users saw normal responses (opaque).
- Clinical decision support: refused new requests; escalated 3 to on-call clinical informaticist.
- Document classification: queued (per workload policy; not degraded mode).

User-facing impact:
- Care Coordinator: ~15 clinicians saw "Quick mode" indicator for ~12 min; no support tickets.
- Patient API chat: ~3,200 users; 4 support tickets ("response seems shorter than usual"); standard response.
- Clinical decision support: 3 escalations handled in real-time by clinical informatics.

Without degraded mode, estimated: 1,000+ support tickets; major incident; customer trust impact.

### 9.5 The Q2 2026 external eligibility outage

External partner's API was down for ~2.5 hours.

- Care Coordinator: continued with "eligibility TBD" markers on ~140 tasks.
- Clinicians worked with the markers; eligibility verified manually for urgent cases.
- After partner restored, batch eligibility check ran on the marked tasks.

User-facing impact: minimal. Clinicians knew "eligibility pending" was a temporary state.

### 9.6 The synthetic trigger schedule

Each feature's degraded modes are tested:

- Care Coordinator: weekly synthetic provider trigger (5 min, off-peak).
- Patient API chat: weekly synthetic trigger.
- Clinical decision support: monthly (rarely needed but tested).

Failures (rare) caught and fixed pre-launch.

### 9.7 The design review process

New AI features at Meridian go through degraded-mode design review:

- Feature owner presents the design (1 page).
- Platform team comments.
- UX team comments.
- Approval before launch.

Average review takes 15-30 minutes per feature; catches most gaps.

### 9.8 What the design discipline produces

- Zero customer-facing AI outages in 18 months despite ~6 significant provider / tool incidents.
- ~4 false-positive degraded-mode activations per month (mostly transient; corrected by next-window check).
- Customer trust: high; "the AI works through bad days" is a customer-success talking point.

### 9.9 The lessons learned

- Degraded mode design is a one-time engineering investment; ongoing benefit.
- Per-feature triggers are essential; one-size-fits-all doesn't work.
- Synthetic testing catches design issues before real incidents.
- UX transparency matters; users prefer "limited mode" over silent quality drift.

---

## 10. Anti-patterns

### 10.1 The "we'll add degraded mode later" deferral

**Pattern.** Feature ships without degraded mode. First provider incident → customer-facing outage.

**Corrective.** Launch-readiness requirement per §6.1.

### 10.2 The silent quality drift

**Pattern.** Degraded mode is opaque; users don't know they're in it. Quality is lower; users notice eventually; trust erodes.

**Corrective.** Transparency per §5.1 for most workloads; opaque only when degradation is truly minimal.

### 10.3 The single-trigger degraded mode

**Pattern.** "When provider is down, fall back." But cost budget exhausted triggers nothing; quality regression triggers nothing.

**Corrective.** All trigger classes per §3.

### 10.4 The one-size-fits-all degraded mode

**Pattern.** Same degraded action for all triggers. Cost trigger → cheaper model (good). Provider trigger → cheaper model (good). Quality trigger → cheaper model (wrong; cheaper model isn't the answer for quality regression).

**Corrective.** Per-trigger action per §4.

### 10.5 The untested degraded mode

**Pattern.** Degraded mode designed; never triggered. First real fire reveals broken implementation.

**Corrective.** Synthetic testing per §7.

### 10.6 The cascading degradation that never bottoms out

**Pattern.** Primary fails → secondary fails → tertiary fails → ... loops or hangs. No final fallback.

**Corrective.** Cascade has a defined endpoint (structured error or feature disable) per §2.10.

### 10.7 The degraded UX that frightens users

**Pattern.** "FEATURE FAILED ⚠️⚠️⚠️" in red. Users panic; support tickets surge.

**Corrective.** Calm, informative UX per §5; "limited mode" is friendlier than "error."

### 10.8 The degraded mode that's worse than no answer

**Pattern.** Degraded mode produces low-quality response that users act on; harm follows. Better to have refused.

**Corrective.** Workload-aware: some workloads should refuse, not degrade (§6.4).

### 10.9 The persistent degraded mode

**Pattern.** Degraded mode fired; never reset. Feature stuck in degraded indefinitely.

**Corrective.** Auto-reset when trigger conditions clear; alert if stuck.

### 10.10 The degraded mode that triggers itself

**Pattern.** Cost trigger → cheaper model → cheaper model has different latency profile → latency trigger → ...cascading.

**Corrective.** Trigger debouncing; degraded modes don't immediately invoke each other.

---

## 11. Findings (sprint-assignable)

### REL-DM-001 — Severity: Critical
**Finding.** AI features ship without degraded mode design.
**Recommendation.** Launch-readiness requirement per §6.1.
**Owner.** engineering management + AI platform, sprint N+1.

### REL-DM-002 — Severity: Critical
**Finding.** Provider failure produces customer-facing outage.
**Recommendation.** Per-feature provider-degraded mode per §3.1 and §4.
**Owner.** AI platform, sprint N+1.

### REL-DM-003 — Severity: Critical
**Finding.** Cost-trigger degraded mode absent.
**Recommendation.** Per-feature cost-trigger action per §3.3.
**Owner.** AI platform + FinOps, sprint N+1.

### REL-DM-004 — Severity: High
**Finding.** Quality-trigger degraded mode absent.
**Recommendation.** Per §3.4; integrate with quality circuit-breaker.
**Owner.** AI platform, sprint N+2.

### REL-DM-005 — Severity: High
**Finding.** Per-feature degraded-mode design not documented.
**Recommendation.** Design doc per §6.2.
**Owner.** feature teams, sprint N+2.

### REL-DM-006 — Severity: High
**Finding.** Degraded modes untested in pre-production.
**Recommendation.** Pre-launch test per §7.3.
**Owner.** AI platform + QA, sprint N+2.

### REL-DM-007 — Severity: High
**Finding.** Synthetic trigger schedule absent.
**Recommendation.** Weekly to monthly per §7.1.
**Owner.** SRE, sprint N+2.

### REL-DM-008 — Severity: High
**Finding.** Degraded mode UX not designed.
**Recommendation.** Per-feature UX pattern per §5.
**Owner.** product + UX, sprint N+2.

### REL-DM-009 — Severity: Medium
**Finding.** Degraded mode observability absent.
**Recommendation.** Metrics + dashboard per §8.
**Owner.** observability-eng, sprint N+3.

### REL-DM-010 — Severity: Medium
**Finding.** Cascading fallback ladder doesn't bottom out.
**Recommendation.** Cascade endpoint per §2.10.
**Owner.** AI platform, sprint N+3.

### REL-DM-011 — Severity: Medium
**Finding.** "Should this feature degrade at all?" not explicit.
**Recommendation.** Decision per §6.4; safety-critical may refuse instead.
**Owner.** product + AI platform, sprint N+3.

### REL-DM-012 — Severity: Medium
**Finding.** Per-trigger action not differentiated; one size fits all.
**Recommendation.** Per-trigger matrix per §4.
**Owner.** AI platform + feature teams, sprint N+3.

### REL-DM-013 — Severity: Medium
**Finding.** Degraded mode design review absent in launch process.
**Recommendation.** Review per §6.3.
**Owner.** engineering management, sprint N+4.

### REL-DM-014 — Severity: Medium
**Finding.** Internal tools have no degraded mode (acceptable but undocumented).
**Recommendation.** Explicit decision per §6.5.
**Owner.** internal tool owners, sprint N+4.

### REL-DM-015 — Severity: Low
**Finding.** Degraded mode that's never triggered (real or synthetic) — possibly broken.
**Recommendation.** Audit annually; trigger synthetically per §7.5.
**Owner.** SRE, sprint N+5.

### REL-DM-016 — Severity: Low
**Finding.** User feedback during degraded mode not captured.
**Recommendation.** Feedback mechanism per §5.7.
**Owner.** product, sprint N+5.

### REL-DM-017 — Severity: Low
**Finding.** Post-incident degraded-mode validation not performed.
**Recommendation.** Post-incident review per §7.4.
**Owner.** SRE, sprint N+5.

### REL-DM-018 — Severity: Low
**Finding.** Feature design evolves; degraded mode doesn't.
**Recommendation.** Quarterly review per feature; refine.
**Owner.** feature teams, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Define launch-readiness requirement (§6.1).** No new features without degraded mode.
- [ ] **For each existing feature, design degraded modes (§6.2).** Per-trigger actions.
- [ ] **Adopt per-feature UX pattern (§5).**
- [ ] **Implement trigger detection (§3).** Integrate with circuit-breakers and cost-budget.
- [ ] **Build per-feature triggers-to-actions matrix (§4).**
- [ ] **Implement cascade fallback ladder (§2.10).**
- [ ] **Pre-launch test each degraded mode (§7.3).**
- [ ] **Synthetic trigger schedule (§7.1).** Weekly to monthly.
- [ ] **Build degraded-mode observability (§8).** Metrics, dashboard, alerts.
- [ ] **Document design per feature.** Reviewed; approved before launch.
- [ ] **Post-incident validation (§7.4).** After each real fire.
- [ ] **Annual audit of untriggered degraded modes (§7.5).**

---

## 13. References

**In this folder.**
- [timeout-strategy.md](./timeout-strategy.md) — timeout that may trigger degraded mode.
- [retry-strategy.md](./retry-strategy.md) — retry failures that exhaust to fallback.
- [fallback-patterns.md](./fallback-patterns.md) — fallback ladder; degraded mode is a layer.
- [circuit-breakers.md](./circuit-breakers.md) — circuit-breakers that trigger degraded mode.
- [fault-budgets-for-ai.md](./fault-budgets-for-ai.md) *(companion)* — error budgets that inform degraded mode choices.
- [capacity-planning.md](./capacity-planning.md) *(companion)* — capacity issues that trigger degradation.

**Elsewhere in this repo.**
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost trigger.
- [cost-and-finops/caching-for-cost.md](../cost-and-finops/caching-for-cost.md) — cache enables cached-response degraded mode.
- [cost-and-finops/finops-process.md](../cost-and-finops/finops-process.md) — launch-readiness includes degraded-mode design.

**Sibling repos.**
- [ai-architecture-reference-architecture / guardrails-and-policy-architecture / refusal-and-escalation-design.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/refusal-and-escalation-design.md) — architectural refusal patterns.
- [ai-architecture-reference-architecture / integration-architecture / integration-failure-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/integration-failure-patterns.md) — failure modes that inform degraded mode design.

**External.**
- Google SRE Book — chapter on graceful degradation.
- "Release It!" by Michael Nygard — bulkhead pattern + graceful degradation.
- Chaos engineering literature (Chaos Monkey, Gremlin).
