# Multi-Provider Failover

> **Audience.** Engineers weighing whether to add a second provider for resilience. Tech leads whose CFO asked "what if Anthropic goes down?" — and who want a defensible answer that isn't "we add OpenAI too." Anyone who built a "provider-agnostic" abstraction that turned into provider-specific anyway. **Scope.** The *engineering* practice of multi-provider failover for AI: when multi-provider is worth the operational cost; the abstraction layer that supports it; eval discipline for fallback quality; per-provider config (rate limits, idempotency, model variants); failover mechanics (active-active, active-passive, manual); observability. Not the architectural build-vs-buy decision (see [ai-architecture-reference-architecture / model-strategy / build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md)). Not the model-routing patterns (see [ai-architecture-reference-architecture / model-strategy / model-routing-and-tiering.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-routing-and-tiering.md)). Not the fallback ladder within a provider (see [fallback-patterns.md](./fallback-patterns.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Multi-provider is requested often and justified rarely.

Common requests:
- "What if Anthropic goes down? We should have OpenAI as backup."
- "We don't want vendor lock-in; let's design for multi-provider."
- "Different providers excel at different things; let's use the best of each."

Each of these has a partial truth and a substantial operational cost. The cost is usually higher than the alleged benefit; many "multi-provider for resilience" architectures are net-negative.

What multi-provider costs:

- Two SDKs to maintain.
- Two contracts to negotiate.
- Two rate-limit budgets to track.
- Two model-version cadences to follow.
- Two compliance postures to verify.
- Two eval suites to run.
- Two debugging surfaces during incidents.
- Per-call routing decisions to make and validate.
- Abstraction-layer code that can lag behind one provider's new features.

What it provides:

- Resilience to one provider's outage.
- Resilience to one provider's pricing changes.
- Some workload-specific optimization (different providers' strengths).

For most workloads, the cost exceeds the benefit. The exception cases — when multi-provider is genuinely the right answer — are narrow but real.

This document covers the criteria for when multi-provider makes sense; the abstraction layer that supports it without coupling to one provider's quirks; the eval discipline that validates fallback quality; the failover mechanics; the observability that monitors it.

This document is opinionated about four things:

1. **Default to single-provider for most workloads.** The operational cost of multi-provider is real; don't take it on without a justification.
2. **Multi-provider for resilience is overrated.** Major providers have 99.9%+ availability; the resilience benefit of a second provider is modest. Multi-region within one provider often suffices.
3. **Multi-provider for capability is sometimes right.** When provider A is genuinely better than B for some workload, using both is justified. This is a per-workload decision.
4. **The abstraction layer must be thin.** Heavy abstraction hides provider-specific features and slows adoption of new capabilities. Thin abstraction allows provider-specific code where it matters.

Structure: (2) when multi-provider is worth it; (3) when it isn't; (4) the abstraction layer; (5) eval discipline for fallback quality; (6) per-provider config; (7) failover mechanics (active-active, active-passive, manual); (8) observability; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. When multi-provider is worth it

The cases where the operational cost is justified.

### 2.1 High-availability requirements that exceed one provider's SLA

If your SLA promises 99.99% availability but no single provider commits to that:

- One provider with 99.9%: maximum ~9 hours downtime/year.
- Two independent providers: aggregate ~99.99% theoretical (when failover works).

Multi-provider justified for the SLA.

### 2.2 Rate-limit headroom

If aggregate workload exceeds any single provider's rate limit:

- Provider A: 5,000 RPM.
- Workload: 8,000 RPM peak.
- Add Provider B: 5,000 RPM.
- Combined: 10,000 RPM.

Multi-provider justified for capacity.

### 2.3 Geographic / regulatory distribution

When residency requires multi-region but no single provider has the region:

- Customer requires `ca-central-1` for residency.
- Anthropic doesn't (yet) deploy there.
- Use Cohere in `ca-central-1`.
- Use Anthropic in US for non-Canadian customers.

Multi-provider justified for residency.

### 2.4 Workload-specific capability

When one provider is genuinely better for a specific workload:

- OpenAI for some workloads; Anthropic for others.
- The difference is verified via eval.
- The workload's value justifies the operational complexity.

Multi-provider justified for capability.

### 2.5 Contractual / commercial diversification

Some enterprise customers require multi-vendor commitments:

- Banking customers refusing single-vendor concentration.
- Government contracts requiring multi-vendor.

Multi-provider justified for commercial terms.

### 2.6 The "we expect provider exit" scenario

When provider's continued existence is uncertain:

- Smaller white-label providers.
- Early-stage providers without long track record.

Maintain a second provider as insurance against exit.

### 2.7 The "is this case real" check

Each justification should answer:

- Is this a real, documented business requirement?
- Has the cost of multi-provider been weighed against the benefit?
- Are alternatives (multi-region within one provider) exhausted?

If unclear, default to single-provider.

---

## 3. When it isn't worth it

The cases where multi-provider is over-engineering.

### 3.1 The "what if they go down" speculation

Provider downtime is rare; major providers maintain 99.9%+ availability.

For most workloads, accepting occasional 12-30 minute incidents (with graceful degradation) is cheaper than multi-provider.

### 3.2 The vague "no lock-in" desire

"No lock-in" without specific cost. Reality:

- Every provider has unique features.
- Migration to another provider is months of work even with abstraction.
- "No lock-in" via thin abstraction is still mostly lock-in.

If lock-in is a real concern, document the specific scenario; weigh against operational cost.

### 3.3 The "we'll use them both" enthusiasm

Engineers excited about both providers; both end up in production without justification.

- Doubled operational surface.
- Half the engineering benefit.

Pick one; revisit annually.

### 3.4 The startup with low volume

A startup with $1k/month AI spend has minimal benefit from multi-provider; the operational cost overshadows the resilience.

Single-provider until volume justifies the work.

### 3.5 The "premium customer requires it but didn't say why"

Customer requirement that's actually just a preference; engineering accepts without negotiation.

Push back; ask for the specific requirement.

### 3.6 The hidden cost of "we have two providers"

Real costs:
- 2 rate-limit budgets to track.
- 2 SDKs to update.
- 2 eval suites to maintain.
- 2 incident-response runbooks.

If team doesn't have capacity for these, multi-provider becomes a slow degradation.

### 3.7 The single-provider with multi-region

For most resilience use cases, multi-region within one provider is sufficient:

- Anthropic US East + EU.
- Each region serves its geography.
- If one region degrades, the other can serve.

Without multi-provider operational overhead.

---

## 4. The abstraction layer

If multi-provider is justified, the engineering layer.

### 4.1 The abstraction goals

- Common interface to multiple providers.
- Per-provider configuration.
- Failover decisions made in one place.
- Provider-specific features still accessible.

### 4.2 The interface shape

```python
class LLMProvider:
    def call(self, messages: list, model: str, **kwargs) -> Response:
        pass
    
    def stream(self, messages: list, model: str, **kwargs) -> Iterator[Token]:
        pass

class AnthropicProvider(LLMProvider):
    def call(self, messages, model, **kwargs):
        # Anthropic-specific implementation
        pass

class OpenAIProvider(LLMProvider):
    def call(self, messages, model, **kwargs):
        # OpenAI-specific implementation
        pass
```

Each provider implements the interface; consumers call through the interface.

### 4.3 The thin-abstraction principle

The abstraction covers common operations:

- Send messages; receive response.
- Stream tokens.
- Tool / function calling.
- Idempotency-key support.

Provider-specific features (extended thinking, prompt caching, structured outputs) can be:

- Exposed through the abstraction with provider-specific flags.
- Accessed directly via provider's SDK when needed.

Don't try to make everything provider-agnostic; impossible and costly.

### 4.4 The routing layer

A layer above the providers that decides which to call:

```python
class ProviderRouter:
    def __init__(self):
        self.providers = {
            "anthropic": AnthropicProvider(),
            "openai": OpenAIProvider(),
        }
    
    def route(self, request: Request) -> Response:
        primary = self.determine_primary(request)
        try:
            return self.providers[primary].call(request)
        except CircuitOpen:
            secondary = self.determine_secondary(request)
            return self.providers[secondary].call(request)
```

The router knows about providers; consumers don't.

### 4.5 The provider-specific configuration

Each provider has its own:

- API key.
- Endpoint URL.
- Rate limits.
- Model name conventions (Anthropic: `claude-sonnet-4-6`; OpenAI: `gpt-4o`).
- Idempotency-key header conventions.

Config per provider; consumed by provider implementations.

### 4.6 The model-name normalization

Different providers use different names. Choose:

- Provider-native names: `claude-sonnet-4-6`, `gpt-4o`.
- Abstract names: `large-fast-model`, `premium-reasoning-model`.

Native names are clearer; abstract names hide which provider is in use (sometimes desirable, often not).

### 4.7 The abstraction-leakage acceptance

Some things won't abstract cleanly:

- Streaming behavior.
- Tool call format.
- Error response shape.

Accept the leakage; document the provider-specific behavior; don't fight it.

---

## 5. Eval discipline for fallback quality

If failover happens, the fallback's quality must be validated.

### 5.1 The eval suite per provider

For each provider × workload:

- Same eval set (cross-link to [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md)).
- Run periodically.
- Compare quality.

If provider B's quality on the workload is much lower than A's, failover is degraded — and the customer should know.

### 5.2 The "is fallback acceptable" question

For each workload:

- Is provider B's quality within acceptable degradation range?
- If yes: failover is acceptable.
- If no: failover should serve "feature unavailable" rather than degraded quality.

### 5.3 The "we never tested the fallback" anti-pattern

A common failure: the fallback was added, never validated, never used. First time it fires, it produces broken output.

Quarterly synthetic failover tests; verify fallback works.

### 5.4 The per-workload eval matrix

```
Workload                       Primary  Fallback  Quality delta
─────────────────────────────────────────────────────────────────
Care Coordinator clinical     Anthropic OpenAI    -8% (acceptable)
Patient API chat              Anthropic OpenAI    -3% (acceptable)
Document classification       Anthropic OpenAI    -2% (acceptable)
Clinical decision support     Anthropic n/a       (no failover; refuse)
```

The matrix surfaces which workloads have viable failover; which don't.

### 5.5 The provider-version compatibility

When provider releases a new version, the eval must be re-run:

- Anthropic releases Sonnet 4.7.
- Re-evaluate against the eval suite.
- Verify failover-to-OpenAI behavior is still acceptable.

### 5.6 The "fallback quality has drifted" detection

Live monitoring during failover:

- Quality metric for fallback-served requests.
- Compare to fallback's historical quality.
- If drift, investigate before next failover.

### 5.7 The customer-facing communication

When failover happens with reduced quality:

- User-visible indicator (cross-link to [degraded-mode-design.md §5](./degraded-mode-design.md)).
- Documentation of expected behavior during failover.
- Customer agreement / SLA terms.

---

## 6. Per-provider config

The configuration surface.

### 6.1 The config schema

```yaml
providers:
  anthropic:
    api_key: env:ANTHROPIC_API_KEY
    endpoint: https://api.anthropic.com
    rate_limits:
      rpm: 5000
      tpm: 2_000_000
    idempotency_header: Idempotency-Key
    supports_streaming: true
    supports_tools: true
    supports_prompt_caching: true
    models:
      large-fast: claude-sonnet-4-6
      reasoning: claude-opus-4-7
      small-fast: claude-haiku-4-5
    
  openai:
    api_key: env:OPENAI_API_KEY
    endpoint: https://api.openai.com
    rate_limits:
      rpm: 4000
      tpm: 1_800_000
    idempotency_header: Idempotency-Key
    supports_streaming: true
    supports_tools: true
    supports_prompt_caching: true
    models:
      large-fast: gpt-4o
      reasoning: o1
      small-fast: gpt-4o-mini
```

Per-provider; comprehensive; consumed by abstraction.

### 6.2 The model-mapping per provider

When the abstraction uses abstract names, the mapping is in config:

```yaml
abstract_models:
  large-fast:
    anthropic: claude-sonnet-4-6
    openai: gpt-4o
  reasoning:
    anthropic: claude-opus-4-7
    openai: o1
```

When the consumer says "call large-fast," the abstraction picks the provider's name.

### 6.3 The per-provider rate-limit budget

Track per-provider headroom independently:

```python
def get_headroom(provider):
    return {
        "rpm": redis.get(f"provider_rpm:{provider}"),
        "tpm": redis.get(f"provider_tpm:{provider}"),
    }
```

Failover decisions consult per-provider headroom.

### 6.4 The per-provider idempotency

Each provider has its own idempotency scheme:

- Anthropic: `Idempotency-Key` header.
- OpenAI: `Idempotency-Key` header (similar).
- Cohere: per-request configuration.

The abstraction abstracts the header; specific implementations differ.

### 6.5 The per-provider tool format

Tool / function calling format differs:

- Anthropic: JSON schema with `input_schema`.
- OpenAI: JSON schema with `parameters`.

The abstraction can normalize, or expose both formats. Decide per workload.

### 6.6 The per-provider error mapping

Errors map to standard classes:

```python
def map_error(provider_error):
    if isinstance(provider_error, anthropic.RateLimitError):
        return StandardError(class_="rate_limit", retry_after=provider_error.retry_after)
    if isinstance(provider_error, openai.RateLimitError):
        return StandardError(class_="rate_limit", retry_after=provider_error.retry_after)
    ...
```

Consumer code handles `StandardError`; provider-specific errors abstracted away.

---

## 7. Failover mechanics

The three patterns.

### 7.1 Active-active

Both providers serve traffic simultaneously:

- 50% to A; 50% to B.
- If one fails, the other absorbs.
- Continuous validation that both work.

**Pros.** Always tested; smooth failover; capacity from both.
**Cons.** Cost roughly 2x; complexity; harder to reason about.

**Best for.** Very high-volume workloads where capacity from both is needed; high-availability requirements.

### 7.2 Active-passive

Primary serves; secondary is on standby:

- 100% to A.
- If A fails, switch to B.
- B is exercised periodically (synthetic test).

**Pros.** Cost-effective (one is the main); operational simplicity.
**Cons.** Secondary may degrade silently if not exercised; failover has switch-over cost.

**Best for.** Most multi-provider use cases.

### 7.3 Manual failover

Provider switch is a deliberate engineering decision:

- Primary fails or degrades.
- Engineer decides to switch to secondary.
- Switch is deployed.

**Pros.** Full control; safer for low-trust failover quality.
**Cons.** Manual response time; not real-time failover.

**Best for.** Workloads where automated failover could degrade quality unacceptably.

### 7.4 The failover trigger

For active-passive:

- Provider circuit-breaker opens (cross-link to [circuit-breakers.md §3](./circuit-breakers.md)).
- Rate-limit headroom critically low.
- Manual override.

Each trigger is appropriate for different scenarios.

### 7.5 The "fail forward" pattern

When secondary also fails:

- Tertiary provider (rarely; usually impractical).
- Fall back to cached / templated.
- Surface structured error.

Plan for the failure of failover.

### 7.6 The "failover during outage" priority

When primary is in outage:

- Critical workloads failover first.
- Non-critical workloads queue or shed.

Cross-link to [degraded-mode-design.md](./degraded-mode-design.md).

### 7.7 The "failover with idempotency" handling

If a request was in-flight when primary failed:

- Was it processed? (idempotency key check).
- If processed: use the result.
- If not: retry against secondary with idempotency key.

Cross-link to [retry-strategy.md §3](./retry-strategy.md).

---

## 8. Observability

The metrics that monitor multi-provider.

### 8.1 Per-provider metrics

- Calls per minute.
- Success rate.
- P99 latency.
- Cost per call (per model).
- Headroom %.
- Failover count (when this provider was selected as fallback).

### 8.2 The provider dashboard

For each provider:

- Current state (active / failover / paused).
- Real-time metrics.
- Historical trends.

### 8.3 The failover history

Every failover event logged:

- Timestamp.
- Trigger (which signal fired).
- From provider, to provider.
- Affected workloads.
- Duration.
- Quality observed during failover.

### 8.4 The "are both providers healthy" signal

Per-provider health summary:

- Active-active: both should be ~50%; deviation indicates one is failing.
- Active-passive: primary should be ~100%; failover events are visible.

### 8.5 The cost comparison

```
Last 30 days:
  Provider A (Anthropic): $52k for X requests
  Provider B (OpenAI): $5k for Y requests (mostly during 2 failover events)
  
Cost per request:
  A: $0.040
  B: $0.042 (small premium during failover)
```

Visible to FinOps.

### 8.6 The quality comparison during failover

Per workload:

- Quality on primary (steady-state).
- Quality on secondary (during failover events).
- Trend.

Drift in fallback quality is visible.

### 8.7 The synthetic failover validation

Quarterly:

- Force failover for 5-10 minutes.
- Verify secondary handles correctly.
- Capture metrics.
- Tune.

---

## 9. Worked Meridian example

Meridian's multi-provider posture: deliberately single-primary except where genuinely justified.

### 9.1 The provider catalog

```
Anthropic (US):
  - Primary for Care Coordinator, Patient API chat (US), Analytics
  - Sonnet 4.6 + Haiku 4.5
  - Volume: ~95% of total traffic

Cohere (CA):
  - Primary for Patient API chat (Canadian customers)
  - Command R+
  - Volume: ~3% of total traffic
  - Justification: residency (Anthropic not in ca-central-1)

OpenAI:
  - Not currently used in production
  - Could be added for failover if Anthropic's reliability degrades

Self-hosted Llama:
  - Primary for document classification, embedding
  - Volume: ~2% (in calls; large in tokens)
  - Justification: cost (50% savings at scale)
```

### 9.2 The "do we add OpenAI" decision

Engineering discussed adding OpenAI as failover for Anthropic. Analysis:

**Pro arguments:**
- Resilience to Anthropic outage.
- Vendor diversification.

**Con arguments:**
- Operational cost: ~0.25 FTE ongoing.
- Quality delta: eval shows ~8% degradation on Care Coordinator with GPT-4o vs Sonnet 4.6.
- Provider lock-in is real either way (different SDKs, different APIs, different features).
- Anthropic's 2025 availability: 99.94% (above SLA).

**Decision:** Don't add OpenAI as automatic failover. Manual fallback procedure documented; switch only if Anthropic has multiple-week outage.

The decision is revisited annually.

### 9.3 The Cohere addition (justified)

When Atlantic Maple Canadian customer onboarded, Anthropic wasn't deployed in `ca-central-1`. Options:

- Cross-border with Anthropic-US (Atlantic Maple's compliance team uncomfortable).
- Self-host in `ca-central-1` (significant operational lift).
- Use Cohere Command R+ (deployed in `ca-central-1`).

Decision: Cohere for Canadian customers.

Implementation:
- Thin abstraction layer wrapping both Anthropic and Cohere.
- Per-region routing: Canadian customers → Cohere; others → Anthropic.
- Per-workload eval: Cohere achieves 92% on Care Coordinator eval suite vs Anthropic's 95%. Acceptable; Atlantic Maple agreed.

### 9.4 The abstraction layer

```python
class LLMProvider:
    def call(self, ...): pass

class AnthropicProvider(LLMProvider):
    def call(self, ...):
        return anthropic.messages.create(...)

class CohereProvider(LLMProvider):
    def call(self, ...):
        return cohere.chat(...)

class ProviderRouter:
    def select(self, tenant_id, region) -> LLMProvider:
        if region == "ca-central-1":
            return CohereProvider()
        return AnthropicProvider()
```

Routing per tenant region. No fall-back-between-providers logic (single-provider per region).

### 9.5 The eval per workload

For Care Coordinator:

- Anthropic Sonnet: 95% pass rate (target 95%; met).
- Cohere Command R+: 92% pass rate (delta -3 pts; acceptable for Canadian customers).

For document classification (self-hosted Llama):
- Llama 3 70B: 92% pass rate (target 90%; exceeds).

Per-workload validation; quarterly re-run.

### 9.6 The manual fallback procedure (Anthropic → OpenAI)

For severe Anthropic outage (multi-hour):

- Document fallback procedure in runbook.
- Manual deployment of OpenAI-routing version.
- Test in staging quarterly.
- Estimated activation time: 30-60 minutes.

Not real-time failover; deliberate decision based on cost-benefit.

### 9.7 The "we got asked about multi-provider" stakeholder conversation

Customer asks: "What's your multi-provider story?"

Response template:
- Primary: Anthropic (98% availability target; actual 99.94%).
- Secondary: documented procedure to migrate to OpenAI within 1 hour for major outage.
- Region-specific: Cohere for `ca-central-1`.
- Single-tenant dedicated infrastructure: available for enterprise.

Honest; defensible; doesn't promise more than delivered.

### 9.8 The Q3 2025 incident

Anthropic had a 28-minute degradation:

- Care Coordinator's circuit-breaker opened.
- Patient API chat fell back to Haiku within Anthropic (cross-link to [fallback-patterns.md](./fallback-patterns.md)).
- No multi-provider failover; the Anthropic-internal fallback was sufficient.
- Total cost during incident: ~$80.

If OpenAI had been wired in:
- Would have failed-over.
- Quality delta on Patient API chat: ~3% (acceptable).
- Cost: comparable.
- Engineering ongoing cost: ~$25k/year operational.

Decision validated: Anthropic-internal fallback was sufficient; multi-provider would have been over-engineering.

### 9.9 What the multi-provider posture costs

- Cohere addition (justified by residency): ~3 weeks initial; ongoing ~$200/month + 0.1 FTE.
- Self-hosted Llama (justified by cost): ~8 weeks initial; ongoing 0.3 FTE.
- No-OpenAI-failover: $0 (the savings).

### 9.10 The lessons

- Single-provider with internal fallback (model tier, region) is often sufficient.
- Multi-provider justified for residency more often than for resilience.
- The abstraction must be thin; trying to make Anthropic and Cohere fully interchangeable was abandoned.
- Annual review prevents drift; multi-provider posture is reconsidered.

---

## 10. Anti-patterns

### 10.1 The "what if Anthropic goes down" speculation

**Pattern.** Multi-provider for hypothetical risk. Major providers don't go down often; operational cost exceeds the benefit.

**Corrective.** Document specific scenarios per §2.1; evaluate cost.

### 10.2 The "no lock-in" theater

**Pattern.** Abstraction layer added for "no lock-in"; doesn't actually enable easy switching (each provider has unique features used).

**Corrective.** Accept some lock-in; the abstraction is for managing two providers, not eliminating lock-in.

### 10.3 The untested fallback

**Pattern.** Secondary provider added; never exercised; quality unknown. First real failover produces customer-visible quality regression.

**Corrective.** Synthetic failover tests per §5.3.

### 10.4 The thick abstraction

**Pattern.** Abstraction tries to be fully provider-agnostic; can't access provider-specific features; lags behind both providers' improvements.

**Corrective.** Thin abstraction per §4.3.

### 10.5 The "we'll add the second provider later" deferral

**Pattern.** Multi-provider on the roadmap "for v2." Years pass; "v2" never comes; resilience theater.

**Corrective.** Either add it (with operational cost) or don't pretend to.

### 10.6 The provider that's added then abandoned

**Pattern.** Second provider added; team doesn't maintain; quality drifts; capability lags.

**Corrective.** Ongoing maintenance budget per §2.7; if no resources, don't add.

### 10.7 The "multi-provider means automatic failover" assumption

**Pattern.** Adding a second provider doesn't automatically mean automatic failover; that's a different engineering effort.

**Corrective.** Multi-provider for capability (or residency) is different from multi-provider for failover.

### 10.8 The hidden cost of "we have two providers"

**Pattern.** Two SDKs, two contracts, two eval suites — operational drag exceeds the architectural benefit.

**Corrective.** Document the cost per §1; weigh against benefit.

### 10.9 The cross-provider quality drift

**Pattern.** Quality on primary stays good; quality on secondary drifts (different rate of improvement); fallback quality decreases over time.

**Corrective.** Per-workload eval per provider per §5.

### 10.10 The "multi-provider = no need for degraded mode" assumption

**Pattern.** Multi-provider as a substitute for degraded mode. When both providers degrade simultaneously, no fallback.

**Corrective.** Degraded mode per [degraded-mode-design.md](./degraded-mode-design.md) is still required.

---

## 11. Findings (sprint-assignable)

### REL-MPF-001 — Severity: Critical
**Finding.** Multi-provider posture undocumented or unjustified.
**Recommendation.** Document per §2 with specific justifications.
**Owner.** engineering management + AI platform, sprint N+1.

### REL-MPF-002 — Severity: High
**Finding.** Abstraction layer (if any) is too thick; hides provider features.
**Recommendation.** Thin abstraction per §4.3.
**Owner.** AI platform, sprint N+2.

### REL-MPF-003 — Severity: High
**Finding.** No eval per provider per workload.
**Recommendation.** Per-workload, per-provider eval per §5.4.
**Owner.** AI platform + eval, sprint N+2.

### REL-MPF-004 — Severity: High
**Finding.** Untested fallback (synthetic tests absent).
**Recommendation.** Quarterly synthetic failover per §5.3.
**Owner.** SRE + AI platform, sprint N+2.

### REL-MPF-005 — Severity: Medium
**Finding.** Failover trigger ambiguous.
**Recommendation.** Per §7.4; circuit-breaker, headroom, manual.
**Owner.** AI platform + SRE, sprint N+3.

### REL-MPF-006 — Severity: Medium
**Finding.** Per-provider observability incomplete.
**Recommendation.** Metrics + dashboard per §8.
**Owner.** observability-eng, sprint N+3.

### REL-MPF-007 — Severity: Medium
**Finding.** Multi-provider added without ongoing maintenance budget.
**Recommendation.** FTE allocation per §2.7.
**Owner.** engineering management, sprint N+3.

### REL-MPF-008 — Severity: Medium
**Finding.** Failover during incident not in runbook.
**Recommendation.** Procedure per §7; documented.
**Owner.** SRE, sprint N+3.

### REL-MPF-009 — Severity: Medium
**Finding.** Single-region within one provider not considered as resilience alternative.
**Recommendation.** Per §3.7; evaluate multi-region before multi-provider.
**Owner.** AI platform, sprint N+4.

### REL-MPF-010 — Severity: Medium
**Finding.** Active-active vs active-passive choice not deliberate.
**Recommendation.** Per §7.1, §7.2; documented.
**Owner.** AI platform, sprint N+4.

### REL-MPF-011 — Severity: Medium
**Finding.** Cost comparison per provider absent.
**Recommendation.** Per §8.5.
**Owner.** FinOps + observability, sprint N+4.

### REL-MPF-012 — Severity: Medium
**Finding.** Per-provider rate-limit tracking missing.
**Recommendation.** Per §6.3.
**Owner.** AI platform, sprint N+4.

### REL-MPF-013 — Severity: Low
**Finding.** Customer-facing multi-provider story undocumented.
**Recommendation.** Per §9.7.
**Owner.** product + customer success, sprint N+5.

### REL-MPF-014 — Severity: Low
**Finding.** Annual review of multi-provider posture not scheduled.
**Recommendation.** Per §9.10.
**Owner.** engineering management, sprint N+5.

### REL-MPF-015 — Severity: Low
**Finding.** Provider-version compatibility re-eval not on cadence.
**Recommendation.** Per §5.5.
**Owner.** AI platform, sprint N+5.

### REL-MPF-016 — Severity: Low
**Finding.** Tertiary provider absent for "fail forward" scenarios.
**Recommendation.** Plan per §7.5.
**Owner.** AI platform, sprint N+6.

### REL-MPF-017 — Severity: Low
**Finding.** Customer-facing communication during failover undocumented.
**Recommendation.** Per §5.7.
**Owner.** customer success, sprint N+6.

### REL-MPF-018 — Severity: Low
**Finding.** Idempotency-key handling across providers not verified.
**Recommendation.** Per §7.7; tested.
**Owner.** AI platform, sprint N+6.

---

## 12. Adoption sequencing checklist

For a team considering multi-provider:

- [ ] **Identify the specific justification (§2).** Document the use case.
- [ ] **Weigh against single-provider alternatives (§3).** Multi-region within one provider; degraded mode.
- [ ] **If justified, design abstraction layer (§4).** Thin; not eliminating lock-in.
- [ ] **Per-provider config (§6).**
- [ ] **Per-provider eval (§5).** Per-workload quality validation.
- [ ] **Failover mechanism (§7).** Active-active, active-passive, or manual.
- [ ] **Synthetic failover testing (§5.3).** Quarterly.
- [ ] **Observability per provider (§8).**
- [ ] **Per-provider runbook for incidents.**
- [ ] **Annual multi-provider posture review.**
- [ ] **Ongoing maintenance allocation.**

---

## 13. References

**In this folder.**
- [fallback-patterns.md](./fallback-patterns.md) — fallback ladder within one provider (often sufficient).
- [circuit-breakers.md](./circuit-breakers.md) — per-provider breaker that triggers failover.
- [timeout-strategy.md](./timeout-strategy.md) — timeouts per provider.
- [retry-strategy.md](./retry-strategy.md) — retry per provider.
- [degraded-mode-design.md](./degraded-mode-design.md) — degraded mode complements failover.
- [capacity-planning.md](./capacity-planning.md) — capacity dimensions per provider.

**Elsewhere in this repo.**
- [eval-engineering/regression-eval-suites.md](../eval-engineering/regression-eval-suites.md) — eval discipline for fallback quality.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — cost per provider.

**Sibling repos.**
- [ai-architecture-reference-architecture / model-strategy / build-vs-buy-decision.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/build-vs-buy-decision.md) — architectural framework that includes multi-provider analysis.
- [ai-architecture-reference-architecture / model-strategy / model-routing-and-tiering.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-routing-and-tiering.md) — routing patterns.
- [ai-architecture-reference-architecture / model-strategy / model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md) — catalogue tracking provider × model.
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / data-residency-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/data-residency-patterns.md) — residency-driven multi-provider use case.

**External.**
- Provider SDKs (Anthropic, OpenAI, Cohere, Google) documentation.
- LiteLLM (open-source multi-provider abstraction).
- AWS Bedrock, GCP Vertex (multi-provider via cloud).
- "Release It!" by Michael Nygard — circuit-breaker and failover patterns.
