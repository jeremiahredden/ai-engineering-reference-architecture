# Tier Routing for Cost

> **Audience.** Engineers building cost-reducing routing into AI features. Tech leads who have seen "the cost line went up 40% last quarter" and are looking for a structural lever, not another negotiation with the vendor. **Scope.** The *engineering* practice of cheap-first model routing with escalation on signal. Pair with [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) (the bound on the routing) and [model-routing-and-tiering.md](../../ai-architecture-reference-architecture/model-strategy/model-routing-and-tiering.md) (the architecture-side decision framework). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Tier routing — using a cheaper model for the easy cases and escalating to a more capable model only on signal — is the single highest-leverage engineering pattern for AI cost reduction in 2026. Done well, it cuts cost 40-70% on tiered workloads without measurable quality loss. Done poorly, it produces silent quality regression (the cheap-tier worker is bad at some class of questions and the team does not notice).

The pattern works because most AI workloads have a long tail. The hard 20% of cases genuinely need the frontier model; the easy 80% are handled equivalently by a cheaper tier. Tier routing identifies the easy 80% and serves them cheap; the hard 20% escalate to the capable tier.

The engineering challenge is *correctly classifying* easy vs hard cases. Misclassification produces either quality loss (an easy-classification is actually hard, the cheap tier botches it) or cost loss (a hard-classification is actually easy, the expensive tier was unnecessary). The router architecture, the eval discipline that calibrates it, and the observability that catches drift are the engineering work this document covers.

This document is opinionated about three things:

1. **The router is itself an LLM call (usually a cheap one).** Rule-based routers handle a small set of cases; classifier-based and LLM-based routers handle the broader workload. Done at Haiku-tier with a small focused prompt, the router cost is negligible.
2. **Routing is eval-gated.** Every routing change is eval-validated against the broader eval suite to confirm no quality regression on the routed-down class. Without this gate, routing silently degrades quality.
3. **The router has explicit failure modes.** When the cheap-tier worker produces low-confidence output, the router can escalate mid-flight. The escalation path is engineered, not emergent.

Structure: (2) the routing patterns; (3) router architectures; (4) calibration discipline; (5) escalation patterns; (6) integration with the broader engineering stack; (7) cost / quality measurement; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The routing patterns

Three patterns. Each has a different cost / complexity profile.

### 2.1 Pattern A: Pre-call classification

Before the main call, a router (usually a cheap LLM call or a classifier) determines which tier should handle this request. The main call goes to the determined tier.

```
Request → Router classifies → Tier selected → Main LLM call → Response
```

**When to use.** When the request's class can be determined from inspection alone (the question's text, the request metadata). The classification cost is added once per request; the main call cost is reduced.

**Trade-off.** Adds a small classification call (~$0.001 at Haiku-tier); reduces the main call cost meaningfully. Net savings depends on the classification accuracy and the cost spread between tiers.

### 2.2 Pattern B: Try-cheap-then-escalate

The cheap tier handles the call first. If the response confidence is below threshold (low confidence from the model, or output validation fails, or a separate confidence-check returns low), escalate to a more capable tier.

```
Request → Cheap tier call → Confidence check:
   - High confidence → return cheap-tier response
   - Low confidence → Escalate to capable tier → Return capable-tier response
```

**When to use.** When the cheap tier handles most cases well but its confidence on low-confidence cases is reliably detectable. Common in workloads where the hard cases are a small percentage and the cheap tier does not produce confidently-wrong outputs on them.

**Trade-off.** No upfront classification cost. The hard cases pay for both the cheap-tier call (which fails the confidence check) and the capable-tier call. Net savings depends on the fraction of cases that escalate and the cheap-tier reliability on the not-escalated cases.

### 2.3 Pattern C: Specialty-specific routing

Different request classes route to specialized workers (different model tiers or different system prompts). The router is rule-based or classifier-based; the workers are specialized.

```
Request → Router classifies → Specialty worker:
   - Class A → Worker A (e.g., Sonnet with prompt A)
   - Class B → Worker B (e.g., Opus with prompt B)
   - Class C → Worker C (e.g., Haiku with prompt C)
```

**When to use.** Multi-class workloads where specialization beyond just tier matters. The Meridian Care Coordinator's classifier-dispatching-to-workers is an instance.

**Trade-off.** Most complex; most flexible. Each worker can be independently optimized. The team manages multiple worker prompts.

### 2.4 The pattern decision

| Workload characteristic | Recommended pattern |
|---|---|
| Single-class workload; easy classification from inspection | Pattern A |
| Single-class workload; cheap-tier mostly works but occasional hard cases | Pattern B |
| Multi-class workload; classes benefit from different specialization | Pattern C |
| Cost-sensitive with high traffic volume | A or B |
| Quality-sensitive with low traffic volume | B (escalation provides safety net) |

Most teams start with A or B and graduate to C as the workload's class structure becomes clear.

---

## 3. Router architectures

The router itself can be implemented several ways.

### 3.1 Rule-based router

A set of explicit rules: "if the query matches pattern X, route to tier Y." Implemented in code.

**When to use.** Small number of well-understood rules; deterministic behavior is valuable; the rules are stable.

**Trade-off.** Easy to reason about; rigid. New cases require code changes. Cannot handle nuance the rules do not capture.

### 3.2 Classifier-based router

A trained classifier (often a small fine-tuned model or a logistic-regression on embeddings) predicts the route from the query.

**When to use.** Many classes; labeled training data exists; the classification is stable enough to justify training.

**Trade-off.** Better at handling nuance than rules; the trained classifier is itself a maintained artifact; needs re-training as data drifts.

### 3.3 LLM-as-router

A cheap LLM call (typically Haiku-tier with a small prompt) determines the route. The prompt asks the model to classify or to recommend a tier.

**When to use.** Cases too varied for rules; no labeled training data for a classifier; the workload's class structure may evolve.

**Trade-off.** Flexible; the prompt can be evolved. Cost is non-zero (a Haiku call per request); needs eval coverage to confirm classification accuracy.

### 3.4 Hybrid router

Rules handle the clear cases; LLM-as-router handles the unclear. The rules catch fast-paths; the LLM handles the rest.

**When to use.** Most production scenarios. The hybrid lets the team get the best of rules (fast, deterministic) and LLM-router (flexible).

**The Meridian Care Coordinator uses this**: simple lookup-shaped questions are rule-routed to the clinical-knowledge worker; complex multi-class questions go through the LLM-as-router (the classifier worker).

### 3.5 The router-as-platform-component vs router-as-application-code decision

- **Platform component.** A central router serves all features. Routing decisions are consistent; eval is centralized.
- **Application code.** Per-feature routing. More flexible; risk of inconsistent decisions across features.

For Meridian: platform component (the classifier worker is a shared platform pattern). New features adopt the pattern.

---

## 4. Calibration discipline

The router's accuracy is itself a metric. Calibration is the engineering practice of keeping it accurate.

### 4.1 The accuracy measurement

For a labeled set of representative requests, the router should classify each correctly. Measure:

- **Classification accuracy.** What fraction of cases are routed correctly?
- **Per-class precision / recall.** For each route, how often is the routing correct?

Run this measurement on:
- A labeled validation set (curated examples with known correct routes).
- A periodic sample of production traffic (relabeled by review).

### 4.2 The cost-quality joint optimization

Pure routing accuracy is not the goal. The goal is the cost-quality joint: how much cost is saved relative to the quality maintained.

The discipline:
- For each candidate routing configuration: measure aggregate cost; measure aggregate quality (via eval suite).
- A configuration that saves 30% cost with no quality loss is better than one that saves 50% cost with 5-point quality drop.
- Document the trade-off explicitly; the team decides.

### 4.3 Recalibration triggers

The router needs recalibration when:
- Model tiers change (a new tier added; an existing tier's behavior changed).
- Prompt-versions change (a new prompt may handle some cases differently).
- Workload class distribution shifts (new feature surfaces a new class).
- Production drift signals (online judge on routed-down cases shows quality slippage).

Quarterly recalibration is the default cadence; trigger-based recalibration handles the rest.

### 4.4 The two-week observation rule

Before activating a router for new traffic: shadow the router against a sample for 2 weeks. Compare router decisions against expert review; measure accuracy; calibrate if needed.

Shadow-first prevents shipping a router that has not been proven on the workload.

---

## 5. Escalation patterns

The router decides up front; what happens when the routed-down tier struggles?

### 5.1 Confidence-based escalation

The routed-down tier's response includes a confidence indicator (the model's own logprobs, a self-rating, output validation). If confidence is below threshold, escalate to the more capable tier.

The pattern works when the cheap tier's confidence is reliably calibrated. If the cheap tier is confidently wrong, escalation does not fire and the wrong answer ships.

### 5.2 Output-validation escalation

The routed-down tier's response is validated (schema check, citation validation, format check). If validation fails, escalate.

Useful for structured-output workloads where the cheap tier sometimes produces invalid structured output.

### 5.3 Multi-step escalation

For agent loops: the cheap tier runs the loop until a complexity threshold (turn count, tool-call count, confidence drop). If the threshold is breached, the loop's remaining turns escalate to the capable tier.

The pattern saves cost on simple multi-step cases; capable-tier kicks in only when the loop's complexity warrants it.

### 5.4 The escalation cost accounting

Each escalation pays for both calls (the cheap tier that didn't succeed, plus the capable tier that did). If the escalation rate is high, the savings are diminished:

- 90% of cases handled cheap-only: 90% × cheap-cost + 10% × (cheap-cost + capable-cost) = 100% × cheap-cost + 10% × capable-cost. Strong savings.
- 50% of cases handled cheap-only: 50% × cheap-cost + 50% × (cheap-cost + capable-cost) = 100% × cheap-cost + 50% × capable-cost. Meaningful savings but less.
- Above 50% escalation: maybe just route everything to capable tier; the cheap-tier overhead is wasted.

The escalation rate is itself a SLI; if it drifts up, routing is not earning.

---

## 6. Integration with the broader engineering stack

The router integrates with multiple platform components.

### 6.1 The LLM-call wrapper

The router invokes the wrapper (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) just like any other consumer. The wrapper records the router's call separately from the main call:

- Router span: `llm_call` with `ai.llm.prompt.version = "router_v1.2"` (or whatever the router's identifier is).
- Main span: `llm_call` with `ai.llm.prompt.version = "main_v2.4.1"` and `ai.llm.model_version = ...`.

The trace shows both calls; cost attribution is per-call; the router's contribution to total cost is visible.

### 6.2 The cost circuit-breaker

Per [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md), the per-interaction budget includes both the router's cost and the main tier's cost. The router itself does not bypass the budget.

If the budget trips during a request that involved escalation: the budget breach handling applies normally.

### 6.3 Observability

The trace records the routing decision:

| Attribute | Meaning |
|---|---|
| `ai.routing.applied` | True if routing was used |
| `ai.routing.router_type` | rule / classifier / llm / hybrid |
| `ai.routing.tier_selected` | which tier was selected |
| `ai.routing.confidence` | router's confidence (if applicable) |
| `ai.routing.escalated` | True if escalation fired |
| `ai.routing.escalation_reason` | reason for escalation |

These attributes support routing-effectiveness dashboards.

### 6.4 The eval gate

Routing changes (new rules, new classifier model, new router prompt) go through the eval gate per [eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md). The eval validates that no routed-down class regresses on quality.

### 6.5 The model registry

Each tier in the routing config is a model in the registry per [model-registry.md](../model-lifecycle/model-registry.md). Adding a new tier requires registry registration first.

---

## 7. Cost / quality measurement

How to know if routing is earning.

### 7.1 The baseline measurement

Before deploying routing: measure aggregate cost-per-interaction and quality SLI for the unrouted (all-on-capable-tier) baseline.

### 7.2 The post-routing measurement

After deploying: measure the same metrics in the routed configuration. Compare:

- **Cost savings.** Aggregate cost-per-interaction with routing vs without.
- **Quality difference.** Quality SLI with routing vs without.
- **Escalation rate.** What fraction of cases escalated.

### 7.3 The continuous monitoring

The savings can drift:
- New model versions may change tier capabilities; the cheap tier may handle more or fewer cases.
- Workload composition may shift; the easy / hard ratio changes.
- Provider pricing may change; the savings calculation shifts.

Monthly: re-measure the baseline (sample some traffic without routing) and the routed configuration. Compare. Adjust if drift is meaningful.

### 7.4 The dashboards

- **Per-feature aggregate cost-savings from routing.** A trend line.
- **Per-route distribution.** What fraction of traffic goes to each tier.
- **Quality SLI by route.** Per-tier quality; degradation on any route is a leading indicator.
- **Escalation rate.** Per feature.

---

## 8. Worked Meridian Care Coordinator example

### 8.1 The routing configuration

The Care Coordinator's routing pattern is multi-layered:

**Outer layer: classifier-based pattern C (specialty-specific routing).** The classifier worker (Haiku-tier) determines: what kind of question is this? Output: `clinical_protocol` / `drug_interaction` / `general_inquiry` / `coordination_task` / `out_of_scope`.

**Worker assignment based on classification:**
- `clinical_protocol` → clinical-knowledge worker (Opus-tier).
- `drug_interaction` → clinical-knowledge worker (Opus) with drug-interaction graph attached.
- `general_inquiry` → drafting worker (Sonnet-tier) with no clinical-knowledge dispatch.
- `coordination_task` → multi-step workflow (mix of tiers).
- `out_of_scope` → escalate-to-human (no LLM cost).

**Inner layer: per-worker tier escalation (pattern B).** The drafting worker (Sonnet) handles most drafting; if it cannot produce acceptable output, escalates to Opus. (Used sparingly — most drafting on Sonnet is good enough.)

### 8.2 The savings calculation

Pre-routing (all-Opus-baseline): per-interaction cost ~$0.42.

Post-routing (current configuration): per-interaction cost ~$0.18.

→ **57% cost savings.**

Quality SLI: 95.2% on clinical golden set (vs 95.4% on all-Opus-baseline). Within tolerance.

The savings are durable across quarters; quarterly re-measurement shows the savings holding at 55-60%.

### 8.3 The classifier prompt

The classifier worker prompt is short (~400 tokens):

```
You are classifying a clinical-staff question into one of five categories:

1. clinical_protocol: questions about clinical guidelines, protocols, or
   evidence-based recommendations.
2. drug_interaction: questions specifically about medication interactions or
   contraindications.
3. general_inquiry: lookup-shaped questions about scheduling, logistics, or
   patient-context that do not need clinical reasoning.
4. coordination_task: multi-step tasks like preparing patient outreach for a
   cohort.
5. out_of_scope: questions outside the Care Coordinator's domain.

Return JSON: { "class": "...", "confidence": <0-1> }
```

The prompt is versioned (currently `classifier_v1.2.0`). Classification accuracy on the labeled set is 94%.

### 8.4 Escalation in practice

The drafting worker's pattern B escalation:

- Cheap-tier first: Sonnet drafts the patient message.
- Confidence check: output validation against the patient-message schema (tone, reading level, citation presence, escalation flag absent unless needed).
- If validation fails: escalate to Opus.

Escalation rate: ~6% of drafting calls escalate. The savings on the 94% non-escalated traffic substantially outweighs the cost of double-paying on the 6%.

### 8.5 The observability

Every Care Coordinator interaction's trace shows the routing decision:

- The classifier worker's span shows the classification output.
- The dispatched worker's span shows which tier was selected.
- If escalation fires, both the cheap-tier span and the escalated span are present.

The routing dashboard shows: per-class distribution (what fraction of traffic is each class), per-class cost (showing the savings), escalation rate, classification accuracy (from periodic relabeling).

### 8.6 The 2026-Q1 routing-recalibration story

Quarterly recalibration in 2026-Q1 surfaced an issue: a new sub-class of drug-interaction questions (involving newly-approved drugs with complex interaction profiles) was being routed to the standard clinical-knowledge worker (Opus) but the drug-interaction graph was not being queried because the classifier was not detecting the new sub-class.

The fix:
1. Classifier prompt updated to include the new drug-interaction sub-class.
2. Eval-validated (the classifier's accuracy on the labeled drug-interaction subset improved from 91% to 96%).
3. Deployed. The new drug-interaction questions now correctly dispatch to the clinical-knowledge worker with the drug-interaction graph attached.

The recalibration paid back the engineering cost in ~2 weeks of avoided quality issues.

### 8.7 The platform discipline

- The classifier worker is a platform-level pattern. New AI features adopt it.
- Every router change goes through the eval gate.
- Quarterly recalibration is scheduled.
- Routing-effectiveness dashboards reviewed monthly.

---

## 9. Anti-patterns

### 9.1 "Route to cheap tier without eval validation"

The team deploys a router that sends most traffic to the cheap tier. Quality regressions on the routed-down class go undetected until users complain.

**Corrective.** Every routing change is eval-validated against the broader eval suite per section 4.

### 9.2 "Router itself is expensive"

The router uses the most capable model (Opus) for classification "to be safe." The router's cost cancels the savings from routing to the cheap tier.

**Corrective.** Router on cheapest tier that produces acceptable classification accuracy. For most workloads, Haiku-tier works.

### 9.3 "Confidence threshold not calibrated"

The cheap tier's confidence threshold for escalation is set to a default value without calibration. Either escalation rate is too high (savings lost) or too low (quality issues).

**Corrective.** Calibrate from production data; tune to the cost-quality joint optimization point.

### 9.4 "Routing decisions not in the trace"

Routing happens but the trace does not record which tier was selected or why. Diagnosing "why did the cheap tier handle this hard question" requires reconstruction.

**Corrective.** Routing attributes per section 6.3 on every span.

### 9.5 "Router prompt versioning is informal"

The router is an LLM call; the router prompt is treated less rigorously than other prompts; changes ship without the eval discipline.

**Corrective.** Router prompts go through the same prompts-as-code discipline per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md).

### 9.6 "Escalation does not stop at the right boundary"

For agent loops, the escalation fires partway through; the agent's prior turns were on the cheap tier; the escalated turn has incomplete context (the cheap tier's working memory was tier-specific). Quality regresses.

**Corrective.** Escalation pattern designed deliberately — either escalate the whole loop from the start, or design the cheap-tier loop to produce escalation-compatible state.

### 9.7 "Routing savings drift unnoticed"

The team set up routing once; the savings calculation was done once; the team assumes the savings hold. In practice, drift accumulates (model pricing changes, workload composition shifts, model behavior evolves).

**Corrective.** Monthly re-measurement of baseline-vs-routed; quarterly recalibration.

### 9.8 "Single-tier escalation fallback"

When the capable-tier call fails (provider outage, rate limit), the system falls back to the cheap tier without disclosure. Quality regression goes unannounced.

**Corrective.** Capable-tier failures follow the fallback patterns per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md); disclosure required when fallback is degraded.

---

## 10. Findings (sprint-assignable)

### COST-TIER-001 — Severity: High
**Finding.** No tier routing is in place; all traffic on the most capable tier; cost is much higher than necessary.
**Recommendation.** Implement tier routing per section 2; start with the highest-volume feature. Eval-validate.
**Owner.** ai-platform-eng, sprint N+1.

### COST-TIER-002 — Severity: High
**Finding.** Routing is implemented but not eval-validated; quality on routed-down classes is unverified.
**Recommendation.** Eval-validate routing per section 4; document the cost / quality trade-off.
**Owner.** ai-platform-eng, sprint N+1.

### COST-TIER-003 — Severity: High
**Finding.** Router uses the most capable model for classification; router cost cancels routing savings.
**Recommendation.** Router on cheapest tier that produces acceptable accuracy per section 3.3.
**Owner.** ai-platform-eng, sprint N+2.

### COST-TIER-004 — Severity: High
**Finding.** Routing decisions are not in the trace; diagnosis of routing issues requires correlation.
**Recommendation.** Routing attributes per section 6.3 on every span.
**Owner.** ai-platform-eng, sprint N+2.

### COST-TIER-005 — Severity: High
**Finding.** Router prompt is treated less rigorously than other prompts; changes ship without eval gate.
**Recommendation.** Router prompts through the prompts-as-code discipline per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md).
**Owner.** ai-platform-eng, sprint N+2.

### COST-TIER-006 — Severity: High
**Finding.** Confidence-based escalation threshold is uncalibrated; escalation rate is either too high or too low.
**Recommendation.** Calibrate from production data per section 4.2.
**Owner.** ai-platform-eng, sprint N+2.

### COST-TIER-007 — Severity: High
**Finding.** Routing savings drift is not monitored; the team assumes savings persist without measurement.
**Recommendation.** Monthly re-measurement per section 7.3.
**Owner.** ai-platform-eng + finops, sprint N+3.

### COST-TIER-008 — Severity: High
**Finding.** Capable-tier escalations bypass the cost circuit-breaker; cost runaways during escalation cycles.
**Recommendation.** Escalations go through the same gateway / circuit-breaker per [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md).
**Owner.** ai-platform-eng + finops, sprint N+2.

### COST-TIER-009 — Severity: Medium
**Finding.** Classifier accuracy is not periodically re-measured; classification drift goes unnoticed.
**Recommendation.** Quarterly accuracy measurement on a labeled set; trigger-based on workload shifts.
**Owner.** ai-platform-eng, sprint N+3.

### COST-TIER-010 — Severity: Medium
**Finding.** Multi-step escalation pattern (within an agent loop) is not designed; escalation fires unpredictably mid-loop.
**Recommendation.** Design escalation boundaries per section 5.3; document.
**Owner.** ai-platform-eng, sprint N+3.

### COST-TIER-011 — Severity: Medium
**Finding.** Routing-effectiveness dashboards do not exist; team cannot answer "is routing earning."
**Recommendation.** Dashboards per section 7.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### COST-TIER-012 — Severity: Medium
**Finding.** Routing logic is hardcoded; changes require deploys.
**Recommendation.** Move routing config to runtime configuration; threshold and route changes via config.
**Owner.** ai-platform-eng, sprint N+3.

### COST-TIER-013 — Severity: Medium
**Finding.** Quality SLI per-route is not separately tracked; quality regressions on one route do not surface.
**Recommendation.** Per-route quality SLI per section 7.4.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### COST-TIER-014 — Severity: Medium
**Finding.** Escalation rate is not tracked; high escalation rates that eliminate savings go unnoticed.
**Recommendation.** Escalation-rate SLI; alert if escalation rate > 30%.
**Owner.** ai-platform-eng, sprint N+3.

### COST-TIER-015 — Severity: Medium
**Finding.** Shadow-test pattern (run new router against sample traffic before activation) is not used; routing changes go live without validation.
**Recommendation.** Shadow for 2 weeks per section 4.4; activate after accuracy validated.
**Owner.** ai-platform-eng, sprint N+4.

### COST-TIER-016 — Severity: Low
**Finding.** Capable-tier fallback on cheap-tier provider failure does not disclose; users see degraded quality silently.
**Recommendation.** Fallback disclosure per [fallback-patterns.md](../reliability-engineering/fallback-patterns.md).
**Owner.** ai-platform-eng, sprint N+4.

### COST-TIER-017 — Severity: Low
**Finding.** Per-tenant routing is not differentiated; premium tenants get same routing as standard.
**Recommendation.** Per-tier tenant routing (premium gets capable tier by default; standard gets cheap with escalation).
**Owner.** ai-platform-eng + product, sprint N+5.

### COST-TIER-018 — Severity: Low
**Finding.** Routing decisions are not documented for non-engineers; product / finance teams cannot reason about the savings.
**Recommendation.** Documentation of the routing strategy; quarterly review with finops.
**Owner.** ai-platform-eng team lead + finops, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team without tier routing:

- [ ] **Sprint 0 — design.** Inventory the workload; identify class structure. Choose pattern A / B / C. Decide router architecture.
- [ ] **Sprint 1 — baseline measurement.** Measure unrouted cost and quality.
- [ ] **Sprint 1 — first router.** Implement for the highest-volume feature with the simplest classification.
- [ ] **Sprint 2 — eval validation.** Eval-validate the routing against the broader eval suite.
- [ ] **Sprint 2 — shadow validation.** Shadow the router against production sample for 2 weeks.
- [ ] **Sprint 3 — activate.** Roll out the router; measure post-routing cost and quality.
- [ ] **Sprint 3 — observability.** Routing attributes on traces; routing dashboards.
- [ ] **Sprint 4 — escalation pattern.** If using pattern B, implement confidence-based escalation; calibrate threshold.
- [ ] **Sprint 4 — ongoing monitoring.** Per-route quality SLI; escalation rate; cost savings.
- [ ] **Sprint 5 — recalibration cadence.** Quarterly recalibration scheduled.
- [ ] **Sprint 5 — extend to other features.** Apply the pattern across the platform.

A team that completes this sequence has the cost-engineering discipline that buys 40-70% cost reduction at no quality cost. A team that skips ends up paying the all-capable-tier baseline.

---

## 12. References

- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — the circuit-breaker that bounds routing's total cost.
- This repo: [cost-and-finops/cost-attribution.md](./) (coming) — the cost telemetry that measures routing savings.
- This repo: [cost-and-finops/caching-for-cost.md](./) (coming) — complementary cost-reduction pattern.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — the eval gate that routing changes go through.
- This repo: [reliability-engineering/fallback-patterns.md](../reliability-engineering/fallback-patterns.md) — fallback patterns for failures during routing.
- This repo: [prompt-engineering/prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) — discipline for router prompts.
- This repo: [model-lifecycle/model-registry.md](../model-lifecycle/model-registry.md) — the registry that catalogs the routing tiers.
- Sibling repo: [ai-architecture-reference-architecture/model-strategy/model-routing-and-tiering.md](https://github.com/jeremiahredden/reference-architecture/blob/main/model-strategy/model-routing-and-tiering.md) — the architecture-side decision framework.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked architecture using these patterns.
