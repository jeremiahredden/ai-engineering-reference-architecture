# Fallback Patterns

> **Audience.** Engineers and SREs operating AI features in production. Tech leads weighing whether "the model is down" should be a user-visible outage or a transparent degraded experience. **Scope.** The *engineering* practice of fallback patterns — model fallback, tier fallback, provider fallback, retrieval fallback, degraded-mode fallback. Trigger conditions, eval discipline, cost/quality/latency trade-offs. Not the architectural decision about multi-provider strategy (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `model-strategy/` and `cost-and-performance-architecture/`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

AI services fail. Provider outages happen. Rate limits get hit. Models get deprecated mid-flight. Tools timeout. Retrievers return empty. Embedding pipelines fall behind. Streaming connections drop mid-response. In production, the question is not whether failures happen but what the system does when they do — fail outright, retry blindly, fall back gracefully, or degrade with explicit user-facing communication.

The fallback discipline is the engineering practice that turns failures into bounded incidents. The single biggest reliability difference between mature and immature AI production systems is whether fallbacks exist as designed components or whether failures cascade into user-visible outages.

This document is not a catalog of every possible fallback; it is the framework for choosing which fallbacks to engineer for which failure classes, calibrated to the workload's reliability requirements. The Care Coordinator's `ARCH-CARE-014` finding (model fallback not configured) is the concrete instance this document closes.

This document is opinionated about three things:

1. **Degraded-mode fallback is the most underused pattern.** Most AI features can degrade to a simpler answer rather than fail outright; few do. The simple "smaller model, no retrieval, explicit disclaimer" pattern is the most leveraged single reliability investment available.
2. **Multi-provider fallback is worth less than it sounds.** The operational cost (eval suite across providers, vendor relationships, complexity) exceeds the resilience gain for most workloads. Use sparingly.
3. **Fallback paths must be eval-validated.** A fallback that produces unacceptable quality is worse than a graceful failure. The fallback's quality is a deliberate engineering decision, not an emergent property.

Structure: (2) the failure-mode catalog; (3) the five fallback patterns; (4) trigger conditions; (5) the eval discipline for fallbacks; (6) cost/quality/latency trade-offs; (7) integration with circuit breakers and observability; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption checklist.

---

## 2. The failure-mode catalog

Fallbacks address specific failure modes. The catalog (drawn from [agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md) section 6.1 and extended):

| Failure mode | Frequency in production | Typical recovery |
|---|---|---|
| Transient provider error (5xx, timeout) | Common (~0.5% of calls) | Retry with backoff |
| Provider rate limit exceeded | Sometimes (peak hours) | Retry with backoff; route to provider quota headroom |
| Provider extended outage (>5 min) | Rare (a few times per year) | Provider fallback or degraded mode |
| Single model deprecated unexpectedly | Rare | Tier fallback or provider fallback |
| Tool transient error | Common (varies by tool) | Bounded retry; route to alternative tool |
| Tool permanent error | Common (authorization, not-found) | Surface to agent; agent picks alternative path |
| Retrieval returns empty | Sometimes | Retrieval fallback (different retriever or different query) |
| Retrieval returns stale results | Sometimes | Surface freshness; allow user to refresh |
| Cost circuit breaker tripped | Rare in steady state | Degraded mode (smaller model, no retrieval) |
| Agent loop budget exhausted | Sometimes | Graceful failure with best-effort partial response |
| Streaming connection dropped mid-response | Sometimes | Resume from last position; or restart with retry |
| Output validation failed | Sometimes | Repair-loop; fallback to less-strict output format |
| Content policy refusal | Rare | Escalate to human; do not retry the same content |

Each failure mode benefits from a specific fallback or recovery pattern. The map is not 1:1; some patterns cover multiple failure modes.

---

## 3. The five fallback patterns

### 3.1 Tier fallback (model-within-provider)

**Shape.** The primary call uses a larger / more-capable model tier. On failure (or on circuit-breaker trip), the fallback uses a smaller / less-capable tier from the same provider, with the same prompt or a tier-adapted prompt.

```
primary: Opus call
   │
   ▼ on failure / trip
fallback: Sonnet call (same provider, smaller model)
   │
   ▼ on failure / trip
degraded: Haiku call (smallest tier; explicit "limited capability" response)
   │
   ▼ on failure
graceful failure
```

**What it gives you.** Recovery within the same provider — no vendor-relationship complexity, no cross-provider eval, lower-latency fallback (typically smaller models respond faster anyway).

**What it costs.** Quality typically degrades on the fallback tier. The eval discipline (section 5) is what determines whether the degraded quality is acceptable; if not, the fallback should not be used and the failure should propagate.

**When to use.** Any production AI feature with a tiered model strategy. The pattern is cheap to set up and the most common fallback used.

**Trigger.** Provider 5xx after retries exhausted; rate-limit-exceeded with no headroom; per-interaction cost circuit-breaker trips; latency exceeds tolerance.

### 3.2 Provider fallback (cross-provider)

**Shape.** The primary call uses one vendor (Anthropic). On failure, the fallback uses a different vendor (OpenAI) with an equivalent-tier model.

```
primary: Anthropic Claude Opus 4.7
   │
   ▼ on failure
fallback: OpenAI GPT-5
```

**What it gives you.** Resilience to single-provider outages. For high-availability workloads where a provider outage would be a Sev-1 incident, this is the only protection.

**What it costs.** Operational overhead is significant: maintaining vendor relationships and contracts with multiple providers, maintaining eval coverage for the fallback provider (the eval must confirm the fallback produces acceptable quality), handling provider-specific quirks in the wrapper, managing prompt-portability (some prompts behave differently across providers).

**When to use.** Mission-critical workloads where availability requirements exceed what single-provider can offer. Workloads operating at sustained-high volume where multi-provider quota headroom is needed regardless of outage scenarios.

**When NOT to use.** Most workloads. The complexity is not worth the resilience gain for typical B2B SaaS uptime requirements.

**Trigger.** Provider extended outage; provider rate limits exceeded across all the team's accounts; provider authentication failure persisting > 5 minutes.

### 3.3 Retrieval fallback

**Shape.** The primary retrieval (e.g., hybrid BM25+vector with reranker) is the rich path. The fallback is a simpler, more available path — BM25-only, or a cached set of common retrievals, or a static knowledge base.

```
primary: hybrid retrieval + reranker
   │
   ▼ on failure / empty results
fallback: BM25-only against the same corpus
   │
   ▼ on failure / empty results
degraded: serve a "I do not have information on that" response
```

**What it gives you.** Recovery when vector store or reranker fails. BM25 is typically less likely to fail (simpler infrastructure) and gives the model at least some context to work from.

**What it costs.** Quality degrades; the model is working from less-relevant chunks. The eval discipline determines whether this is acceptable for the workload.

**When to use.** RAG features that depend on retrieval quality. The pattern is cheap to set up (BM25 is usually already part of the hybrid pipeline) and provides meaningful resilience.

**Trigger.** Vector store error / timeout; reranker error / timeout; retrieval returns empty when the model needed retrieval to answer.

### 3.4 Tool fallback

**Shape.** When a tool the agent intended to call is unavailable, the agent has an alternative tool that addresses a similar need, or a "knowledge-only" alternative that answers from the model's training rather than fresh data.

```
primary: call clinical-guidelines-API tool
   │
   ▼ on failure
fallback: call cached-clinical-guidelines tool (older content, served from cache)
   │
   ▼ on failure / not cached
degraded: answer from model's training with explicit disclaimer
```

**What it gives you.** Recovery when a specific tool is broken; the agent can still produce useful output via an alternative path.

**What it costs.** Engineering the alternative tools; eval-testing that the fallback tool produces equivalent-or-degraded-but-acceptable behavior.

**When to use.** When the primary tool is on infrastructure outside the team's control (third-party APIs, partner services). Less needed for tools the team owns.

**Trigger.** Tool returns permanent error; tool consistently failing (circuit-breaker on the tool); tool deprecated.

### 3.5 Degraded-mode fallback

**Shape.** When the primary AI feature cannot operate (cost circuit broke, provider outage, multi-component failures), the feature serves a degraded experience explicitly to the user.

```
primary: full Care Coordinator experience (supervisor / workers / retrieval / tools)
   │
   ▼ on full-feature failure (cost circuit, full provider outage, etc.)
degraded: Haiku-only single-call response with retrieval disabled,
          response prefaced with "I can't access the full clinical knowledge
          base right now; here's what I can offer based on general clinical
          knowledge..."
   │
   ▼ on degraded-mode failure
unavailable: explicit "Care Coordinator is temporarily unavailable; please
             contact your clinical supervisor" response
```

**What it gives you.** The biggest reliability win available. Most users prefer a degraded experience to a no-experience; the disclosure ("limited capability right now") manages expectations; the workflow continues.

**What it costs.** Engineering the degraded path requires deliberate design. The degraded mode must be eval-validated to ensure it does not cause harm (in clinical contexts, the degraded mode must not pretend to have capabilities it does not have).

**When to use.** Every production AI feature should have a degraded-mode plan. The pattern is the difference between "the AI feature is sometimes broken for hours" and "the AI feature occasionally serves an explicitly-limited version."

**Trigger.** Per-feature circuit-breaker trip; full-stack outage (multiple components failing); cost-budget feature-level trip; extended provider outage exceeding fallback capability.

---

## 4. Trigger conditions

Fallbacks are triggered by specific conditions. The trigger logic lives in the LLM-call wrapper (per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) and in the agent-loop runner.

### 4.1 Retry-then-fallback

Most fallbacks are preceded by bounded retries on the primary path. The pattern:

1. Primary call attempted.
2. On transient error: retry with exponential backoff (typically 3 retries, jitter, capped at ~5s total).
3. On retries exhausted (still failing): trigger fallback.
4. Fallback attempted.
5. On fallback success: return; surface "served from fallback" in the trace.
6. On fallback failure: trigger degraded mode or graceful failure.

The retry-then-fallback sequence is the standard pattern. Fast-failure straight to fallback skips potentially-recoverable transient errors.

### 4.2 Trigger thresholds

Each trigger has a threshold:

- **Provider 5xx fallback.** After N consecutive 5xx errors within W seconds.
- **Rate-limit fallback.** Immediately on rate-limit response (no retry).
- **Latency fallback.** Time-to-first-token exceeds Z ms.
- **Cost-circuit-breaker fallback.** Pre-call cost estimate trips the per-interaction budget.
- **Tool failure fallback.** Tool consistently failing (circuit-breaker on the tool).
- **Retrieval empty fallback.** Retrieval returns zero results from the primary retriever for a query that should match the corpus.

Thresholds are configured per fallback path; configurations are reviewed quarterly.

### 4.3 Per-tenant trigger awareness

Some fallbacks are tenant-aware. A premium-tier tenant on dedicated infrastructure should fall back differently than a standard-tier tenant on shared infrastructure (the premium tenant's dedicated provider account does not need provider-level fallback; the standard tenant's shared infrastructure does).

The trigger logic consumes tenant context (from the call context per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md)) to make tier-aware fallback decisions.

### 4.4 The cascade

Fallbacks cascade. Tier fallback fails → provider fallback. Provider fallback fails → degraded mode. Degraded mode fails → graceful failure. The cascade is the depth of the defense.

But the cascade has a limit. Each fallback adds latency and cost. The configuration determines how deep the cascade goes for each feature. For most features, two or three levels are sufficient.

---

## 5. The eval discipline for fallbacks

A fallback that produces unacceptable quality is worse than a graceful failure. The eval discipline confirms each fallback path is fit for purpose.

### 5.1 Eval the fallback path against the same suite as the primary

The fallback path is run through the same eval suite as the primary, with the fallback configuration. The pass rate of the fallback is documented and compared against the primary.

For Meridian Care Coordinator:
- Primary (Opus): 95% pass rate on clinical golden set.
- Tier fallback (Sonnet): 87% pass rate on the same set.
- Degraded mode (Haiku, no retrieval, disclaimer): 64% pass rate, with 100% of responses containing the disclaimer (verified separately).

The team accepts the degraded mode's quality knowing that it explicitly disclaims limitations.

### 5.2 The fallback-acceptability gate

For each fallback path, the team defines acceptability criteria:

- Minimum pass rate (relative to primary or absolute floor).
- Maximum hallucination rate.
- Maximum harmful-content rate.
- Required disclaimer presence (for degraded modes).

If the fallback fails the acceptability gate, the fallback should not be used. The failure should propagate to the next level of the cascade or to graceful failure.

### 5.3 Periodic fallback re-eval

When a primary changes (new prompt version, new model version), the fallback path must be re-evaluated against the new prompt or new model context. Skipping this lets the fallback drift away from production reality.

Quarterly re-eval of every active fallback path. Re-calibration if drift is observed.

### 5.4 The "fallback was right" vs "fallback was wrong" data

When fallbacks fire in production, the trace records "primary failed, fallback succeeded." Periodic analysis of these traces:

- Was the fallback's response actually acceptable to the user?
- Did the user complete their task with the fallback response?
- Did the user retry (suggesting the fallback was insufficient)?

This is the production feedback loop on fallback quality. Without it, fallbacks fire and the team assumes they worked; in some cases, they did not.

---

## 6. Cost / quality / latency trade-offs

Each fallback has its own profile. The trade-offs:

| Pattern | Cost vs primary | Latency vs primary | Quality vs primary | Engineering cost to build |
|---|---|---|---|---|
| Tier fallback (Opus → Sonnet) | ~25% | ~50% (faster) | -8 to -15 points | Low |
| Tier fallback (Opus → Haiku) | ~3% | ~30% (much faster) | -25 to -35 points | Low |
| Provider fallback (Anthropic → OpenAI) | similar | similar | varies; needs eval | High (vendor relationship, eval coverage, complexity) |
| Retrieval fallback (hybrid → BM25-only) | ~70% | ~80% | -5 to -10 points | Low (BM25 already in pipeline) |
| Tool fallback (live tool → cached) | <10% | ~10% (very fast) | depends on cache freshness | Medium |
| Degraded mode (Haiku + no retrieval + disclaimer) | ~5% | ~30% | -30 points but explicit | Medium (deliberate design needed) |
| Graceful failure (no AI response) | $0 | instant | 0 (no answer) | Low |

The Meridian Care Coordinator's fallback configuration is calibrated against these trade-offs.

---

## 7. Integration with circuit breakers and observability

Fallbacks are wired into the circuit-breaker pattern (per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)) and into observability (per [trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) and [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md)).

### 7.1 Circuit-breaker triggered fallbacks

When a circuit breaker trips:
- Per-interaction circuit → tier fallback (smaller model) or degraded mode for the rest of the interaction.
- Per-session circuit → degraded mode for the rest of the session.
- Per-tenant circuit → read-only or degraded mode for the rest of the day.
- Per-feature circuit → degraded mode platform-wide; on-call paged.

The circuit-breaker layer in the gateway evaluates the budget; the fallback layer responds to the breach. The two are wired together.

### 7.2 Observability for fallbacks

Every fallback-triggered call carries:
- `ai.fallback.triggered`: True
- `ai.fallback.reason`: the trigger (provider_5xx, rate_limited, circuit_breaker, timeout, etc.)
- `ai.fallback.primary_attempted`: True (always — fallback only after primary attempt)
- `ai.fallback.path`: which fallback (tier_fallback, provider_fallback, retrieval_fallback, degraded_mode)
- `ai.fallback.degradation`: the quality / cost / latency change vs primary

Aggregate dashboards:
- Fallback rate per feature.
- Fallback reason distribution.
- Per-fallback quality (from sampled judge runs).
- Per-fallback user impact (retry rate, abandonment rate).

The dashboards surface "the fallback fires more than expected" as a leading indicator of upstream problems.

### 7.3 Alerting

- Sustained fallback rate above baseline → page on-call (likely a provider issue or a model deprecation).
- Degraded-mode usage rate above 1% of traffic → page on-call (the platform is leaning on degraded mode more than it should).
- Fallback's quality SLI degrades → page on-call (the fallback itself is broken).

---

## 8. Worked Meridian Health example

### 8.1 The Care Coordinator's fallback configuration

For the Care Coordinator's primary chat path:

| Layer | Configuration |
|---|---|
| Primary | Opus 4.7 supervisor + workers, hybrid retrieval, full tool surface |
| Tier fallback (provider transient errors) | Sonnet 4.6 supervisor + workers; same retrieval; full tool surface |
| Retrieval fallback (vector store down) | Same model tier; BM25-only retrieval; full tool surface |
| Degraded mode (cost circuit / multi-component failure / extended outage) | Haiku 4.5 single-call; no retrieval; clinical-disclaimer prefix; tool surface restricted to escalate-to-human |
| Graceful failure | "Care Coordinator is temporarily unavailable; please contact your clinical supervisor" |

The cascade for a typical failure (provider rate-limit exceeded):
1. Primary Opus call attempted. Rate-limited.
2. Retry with backoff (3 attempts). All rate-limited.
3. Tier fallback to Sonnet attempted. Succeeds.
4. Response served; trace shows `fallback_triggered=True, fallback_path=tier_fallback, fallback_reason=rate_limited`.

The cascade for a major incident (provider extended outage):
1. Primary attempted. 5xx for several minutes.
2. Tier fallback (same provider) also fails.
3. Provider fallback is not configured for the standard tier; cascade goes to degraded mode.
4. Degraded mode (Haiku via the secondary provider in the cost circuit-breaker design, or Anthropic Haiku if the primary outage was scoped to Opus) succeeds.
5. Response with disclaimer; trace records the degradation.
6. On-call paged because degraded-mode rate spiked.

### 8.2 The premium tenant's variant

The premium-tier customer (on dedicated infrastructure per [isolation-models.md](../../ai-architecture-reference-architecture/multi-tenancy-and-isolation/isolation-models.md)) has additional configuration:
- Provider fallback is configured (Anthropic → OpenAI for sustained Anthropic outages).
- Degraded mode is the same.

The provider fallback was contractually committed; the operational overhead is justified by the premium pricing.

### 8.3 Production fallback patterns observed

In a quarter, the Care Coordinator's fallback rate:
- Tier fallback: 0.4% of interactions (mostly transient provider errors, occasional rate limits).
- Retrieval fallback: 0.1% (rare vector-store transient errors).
- Degraded mode: 0.02% (one cost-circuit incident, two brief provider outages).
- Graceful failure: 0.001% (a few catastrophic-multi-component failures).

The dashboards show these rates as steady-state baselines. Spikes above baseline page on-call.

### 8.4 The eval coverage

Each fallback path is evaluated:
- Primary: 95% pass rate on clinical golden set.
- Tier fallback (Sonnet): 87% pass rate. Acceptable per the team's threshold (>80%).
- Retrieval fallback (BM25-only): 91% pass rate on the same set. Acceptable (>85%).
- Degraded mode: 64% pass rate. Below the team's normal threshold but accepted because of the explicit disclaimer; 100% of degraded responses contain the disclaimer (verified).

Quarterly re-eval catches drift.

### 8.5 The 2026-04-08 cost-incident response

The cost circuit-breaker trip described in [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) was followed by degraded-mode activation for the affected tenants. The degraded mode served clinicians with the disclaimer; clinicians proceeded with their work without functional impact; one minor user complaint ("the AI seemed less helpful for an hour") was the only user-visible impact.

Without the degraded-mode fallback, the cost-circuit trip would have failed all interactions for the affected scope until the cost issue was resolved. The fallback turned a Sev-1 user-impact into a Sev-3 user-experience issue.

---

## 9. Anti-patterns

### 9.1 "No degraded mode"

The team has retry and timeout configured but no degraded-mode plan. When the primary fails, users get an error and the workflow stops. The team treats outages as binary.

**Corrective.** Degraded mode is the highest-leverage reliability investment. Even a one-day engineering effort produces a meaningful uptime improvement.

### 9.2 "Fallback to the same provider's same model"

The retry path uses the same model and the same provider. When the failure is provider-level or model-level, the retry fails for the same reason. The fallback adds latency without adding resilience.

**Corrective.** Fallback to a different tier, a different provider, or a different mode. The fallback's resilience requires different infrastructure than the primary.

### 9.3 "Multi-provider fallback without eval"

The team configures provider fallback (Anthropic → OpenAI) but never evaluates the OpenAI path's quality on the workload. When the fallback fires, the responses are subtly different in ways the team did not anticipate.

**Corrective.** Eval-validate every fallback path against the same eval suite as the primary; calibrate quality expectations; document the acceptability threshold.

### 9.4 "Retry forever"

The retry loop has no bounded count. Transient errors that persist (a multi-minute outage) cause the retry to consume cost and time without recovery. The user sees the request as hung.

**Corrective.** Bounded retries (typically 3 with backoff, capped at 5s total). After retries exhausted, the fallback fires.

### 9.5 "Degraded mode does not disclose"

The fallback serves a degraded response without telling the user that the response is degraded. Users assume the AI's normal quality; they trust a response that the AI itself would not stand behind.

**Corrective.** Degraded responses include explicit disclosure ("I can't access the full knowledge base right now; this is a limited response based on..."). The disclosure is non-negotiable; verify its presence in eval.

### 9.6 "Fallback rate is not monitored"

Fallbacks fire frequently in production but the dashboards do not surface the rate. The team only learns about fallback issues when an incident escalates.

**Corrective.** Fallback rate per feature is an SLI. Spikes above baseline page on-call. Trends in fallback rate are a leading indicator.

### 9.7 "Fallback for tools the team owns"

The team has built fallbacks for failures of tools they own (their own database, their own internal API). The fallback complexity is engineering effort that would be better spent making the primary more reliable.

**Corrective.** For self-owned tools, invest in reliability of the primary; reserve fallback complexity for failures the team cannot prevent (provider outages, third-party APIs).

### 9.8 "Provider fallback for non-critical workloads"

The team configured provider fallback for a low-volume, low-stakes feature. The operational cost (eval coverage on two providers, vendor relationships) far exceeds the resilience need.

**Corrective.** Multi-provider fallback is for mission-critical workloads only. Most workloads are well-served by tier fallback + degraded mode.

---

## 10. Findings (sprint-assignable)

### REL-FB-001 — Severity: Critical
**Finding.** Production AI feature has no degraded-mode plan; failures result in user-visible outages.
**Recommendation.** Design and ship a degraded-mode fallback per section 3.5; eval-validate; integrate with circuit breakers.
**Owner.** ai-platform-eng + product, sprint N+1.

### REL-FB-002 — Severity: Critical
**Finding.** Degraded-mode responses do not include disclosure; users cannot distinguish degraded from primary responses.
**Recommendation.** Add explicit disclaimer to degraded responses; eval-verify 100% disclosure presence.
**Owner.** ai-platform-eng + product, sprint N+1.

### REL-FB-003 — Severity: High
**Finding.** Tier fallback (Opus → Sonnet) is not configured; provider rate-limits cause user-visible failures.
**Recommendation.** Configure tier fallback in the gateway; trigger on provider 5xx-after-retries and rate-limit; eval-validate the Sonnet path.
**Owner.** ai-platform-eng, sprint N+2.

### REL-FB-004 — Severity: High
**Finding.** Retrieval fallback is not configured; vector-store transient errors propagate to user-facing failures.
**Recommendation.** Add BM25-only retrieval fallback when vector retrieval fails; eval-validate.
**Owner.** ai-platform-eng, sprint N+2.

### REL-FB-005 — Severity: High
**Finding.** Retry-then-fallback sequence is not implemented; retries are unbounded or fallbacks fire immediately.
**Recommendation.** Bounded retry (3 attempts, backoff, 5s cap) then fallback per section 4.1.
**Owner.** ai-platform-eng, sprint N+2.

### REL-FB-006 — Severity: High
**Finding.** Fallback paths are not evaluated against the eval suite; quality on fallbacks is unknown.
**Recommendation.** Eval-validate every active fallback path per section 5.1; document acceptability thresholds.
**Owner.** ai-platform-eng, sprint N+2.

### REL-FB-007 — Severity: High
**Finding.** Fallback rate per feature is not surfaced in dashboards; spikes are not detected.
**Recommendation.** Fallback rate as SLI; alert on baseline-deviation; surface in observability dashboards.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### REL-FB-008 — Severity: Medium
**Finding.** Provider fallback is configured for non-mission-critical workloads where the operational overhead is not justified.
**Recommendation.** Remove provider fallback from non-critical features; rely on tier fallback + degraded mode; document the decision.
**Owner.** ai-platform-eng, sprint N+3.

### REL-FB-009 — Severity: Medium
**Finding.** Fallback responses do not carry observability attributes; trace cannot distinguish fallback responses from primary.
**Recommendation.** Add `ai.fallback.triggered`, `ai.fallback.reason`, `ai.fallback.path` attributes per section 7.2.
**Owner.** ai-platform-eng, sprint N+3.

### REL-FB-010 — Severity: Medium
**Finding.** Periodic re-eval of fallback paths is not scheduled; fallbacks drift relative to production primary.
**Recommendation.** Quarterly re-eval per section 5.3; calibrate or re-engineer drifted fallbacks.
**Owner.** ai-platform-eng, sprint N+4.

### REL-FB-011 — Severity: Medium
**Finding.** Cascade depth is not configured; multi-step fallback chains do not have explicit limits.
**Recommendation.** Document the cascade per feature; cap depth per section 4.4; surface in runbook.
**Owner.** ai-platform-eng, sprint N+3.

### REL-FB-012 — Severity: Medium
**Finding.** Tool fallback is engineered for tools the team owns; complexity is misallocated.
**Recommendation.** Remove fallback for self-owned tools; invest in primary reliability instead; reserve fallback for third-party / shared-infrastructure tool failures.
**Owner.** ai-platform-eng, sprint N+4.

### REL-FB-013 — Severity: Medium
**Finding.** Production fallback firings are not analyzed for user-visible impact (retry rate, abandonment).
**Recommendation.** Per section 5.4, periodic analysis of fallback-served interactions; feedback into fallback acceptability criteria.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### REL-FB-014 — Severity: Medium
**Finding.** Premium-tier and standard-tier tenants share identical fallback configuration; contractual differences are not reflected.
**Recommendation.** Per-tier fallback configuration per section 4.3; premium tier's contracted resilience operationalized.
**Owner.** ai-platform-eng + customer-success, sprint N+4.

### REL-FB-015 — Severity: Medium
**Finding.** Fallback configuration is hardcoded in application code; changes require deploys.
**Recommendation.** Move to runtime configuration; trigger thresholds, paths, acceptability all updateable without deploy.
**Owner.** ai-platform-eng, sprint N+4.

### REL-FB-016 — Severity: Low
**Finding.** Degraded-mode prompts are not separately versioned; changes to the disclaimer require prompt-engineering changes.
**Recommendation.** Degraded-mode prompts as first-class artifacts per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md).
**Owner.** ai-platform-eng, sprint N+5.

### REL-FB-017 — Severity: Low
**Finding.** Fallback alerting thresholds are not calibrated; alerts either fire too often or miss real issues.
**Recommendation.** Calibrate alert thresholds from observed fallback rates; quarterly recalibration.
**Owner.** ai-platform-eng + sre, sprint N+5.

### REL-FB-018 — Severity: Low
**Finding.** Fallback runbook is undocumented; on-call response to fallback-rate alerts is ad-hoc.
**Recommendation.** Per-fallback runbook; integrate with the incident-response process.
**Owner.** ai-platform-eng + sre, sprint N+5.

---

## 11. Adoption sequencing checklist

For a team without fallback discipline:

- [ ] **Sprint 0 — inventory.** Catalog every AI feature; identify its current behavior on common failures. Most features fail outright.
- [ ] **Sprint 1 — degraded mode (most leveraged).** Design and ship degraded-mode fallback for the highest-impact feature. Disclosure mandatory.
- [ ] **Sprint 1 — bounded retry.** Replace unbounded or absent retry with bounded retry-then-fallback.
- [ ] **Sprint 2 — tier fallback.** Configure tier fallback (primary tier → smaller tier within same provider) for the highest-impact feature.
- [ ] **Sprint 2 — retrieval fallback.** For RAG features, add BM25-only fallback for vector-store failures.
- [ ] **Sprint 3 — eval-validate.** Eval every active fallback path against the same suite as the primary; document acceptability thresholds; reject fallbacks below threshold.
- [ ] **Sprint 3 — observability.** Fallback attributes on every fallback-served call; dashboards; alerting.
- [ ] **Sprint 4 — extend to remaining features.** Apply patterns to additional features in priority order.
- [ ] **Sprint 5 — provider fallback (if needed).** For mission-critical workloads only, configure provider fallback with full eval coverage.
- [ ] **Sprint 5 — runbooks.** Per-fallback runbook; on-call rehearsal.
- [ ] **Ongoing — quarterly re-eval.** Fallback drift detection; recalibration.

A team that completes this sequence has the reliability discipline that turns "the AI feature is down" into "the AI feature is degraded gracefully" — the difference between Sev-1 user-impact incidents and Sev-3 user-experience events.

---

## 12. References

- The SRE canon on graceful degradation (Google SRE book chapter on handling overload) — the foundation pattern.
- Circuit breaker / bulkhead / timeout patterns (Hystrix / Resilience4j) — the broader resilience-engineering family.
- This repo: [reliability-engineering/timeout-strategy.md](./) (coming) — the timeout-calibration pattern that interacts with fallbacks.
- This repo: [reliability-engineering/retry-strategy.md](./) (coming) — the retry decision tree that determines when fallbacks fire.
- This repo: [reliability-engineering/circuit-breakers.md](./) (coming) — the broader circuit-breaker pattern.
- This repo: [reliability-engineering/degraded-mode-design.md](./) (coming) — depth on degraded-mode design.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — the cost circuit-breaker that triggers fallbacks.
- This repo: [observability-and-telemetry/trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) — the trace framework for fallback observability.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) — the eval discipline for fallback paths.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — ARCH-CARE-014 is the cross-link finding.
- Sibling repo: [ai-architecture-reference-architecture/cost-and-performance-architecture/](https://github.com/jeremiahredden/ai-architecture-reference-architecture/tree/main/cost-and-performance-architecture) — the architecture context for the cost / latency / availability decisions.
