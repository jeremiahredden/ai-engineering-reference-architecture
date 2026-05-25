# Reliability Engineering

## What this folder is

The engineering practice of keeping AI systems reliable in production — timeout strategy, retry strategy, fallback patterns, circuit breakers, fault budgets, capacity planning for spiky workloads, multi-provider failover, and the design discipline that distinguishes "the model is slow" from "the model is broken." The material here is what I put in front of an SRE team when the question is: *we are getting paged for AI service issues and our standard SRE playbook does not quite fit — what does reliability engineering actually look like for an LLM-shaped system?*

## The organizing principle

AI services have failure modes that conventional distributed systems do not. They are slow in expected steady state (seconds for a chat call, tens of seconds for a long agent), they occasionally produce bad-but-not-error output that traditional health checks miss, they have provider-side outages and rate limits that look like client-side bugs, and they have a brand new failure mode — *the model returned a successful response that is wrong* — that no liveness probe or HTTP status code captures.

So the patterns here adapt the SRE canon to those shapes. Timeouts are calibrated to the long-tail of LLM latency (p99 is usually 5x p50), retries are limited because LLM calls are expensive and many AI operations are non-idempotent, fallbacks are designed around the "smaller model with worse answer" vs "no answer" trade-off, and SLOs include quality dimensions alongside latency and availability.

The folder is opinionated about three things specifically. First, that *the default retry pattern is wrong for many AI calls* — agent steps that take real-world side effects must not be naively retried, and even retrieval calls have failure modes (corpus-side outage producing empty results) that retry does not help. Second, that *multi-provider failover is worth less than it sounds* until traffic is well above the provider rate limits or the SLO is genuinely strict — the operational cost of a second provider can exceed the resilience gain on most workloads. Third, that *degraded-mode design is the most underused reliability pattern* — most AI features can degrade to a simpler answer instead of failing outright, and few do.

## Planned documents

- **timeout-strategy.md** *(coming)* — Calibrating timeouts to the actual latency distribution: per-LLM-call timeout, per-tool-call timeout, per-agent-turn timeout, total-request timeout, the streaming-aware timeout pattern (idle-stream timeout instead of total timeout for streamed responses), and the timeout-as-cost-control role (a timeout caps the maximum cost a single request can incur).
- **retry-strategy.md** *(coming)* — The retry decision tree: retry on transient provider errors yes, retry on rate-limit with backoff yes, retry on timeout sometimes, retry on bad-output rarely (and only with prompt-repair), retry on tool-side-effect-taken never. The idempotency analysis that determines whether retry is safe.
- **fallback-patterns.md** *(coming)* — Model fallback (primary model errors → secondary model retries), tier fallback (Opus errors → Sonnet retries), provider fallback (Anthropic errors → OpenAI retries), retrieval fallback (vector search errors → keyword search), and the degraded-answer fallback (everything errors → "I cannot answer this right now, here is what I can offer instead").
- **circuit-breakers.md** *(coming)* — Circuit breakers around model providers (provider outage → fail fast → fall back), around tools (tool consistently failing → stop trying), and around features (this AI feature is broken → serve degraded mode → page on-call). The threshold-tuning discipline and the integration with cost-as-circuit-breaker.
- **degraded-mode-design.md** *(coming)* — The architectural pattern of every AI feature having a degraded mode (smaller model, no agent loop, cached answer, "feature unavailable" with helpful fallback). The triggers (cost budget breached, error rate elevated, latency elevated, provider outage), the UX patterns, and the runbook integration.
- **fault-budgets-for-ai.md** *(coming)* — Adapting the error-budget SRE pattern to AI: per-feature quality SLO (judge-pass-rate ≥ X%), per-feature latency SLO, per-feature cost SLO, per-feature availability SLO. The error-budget-burn alert pattern and the "stop shipping changes when the budget is exhausted" discipline.
- **capacity-planning.md** *(coming)* — Capacity for AI workloads: provider rate-limit headroom, multi-tenancy of provider quotas, the burst-vs-sustained traffic pattern, the queue-and-shed-load design for over-capacity, and the auto-scaling patterns for self-hosted inference.
- **multi-provider-failover.md** *(coming)* — When multi-provider is worth the operational cost (high-availability requirements, rate-limit headroom, geographic / regulatory distribution), the abstraction layer that supports it without coupling to one provider's quirks, and the eval discipline that validates that "fallback to provider B" produces acceptable quality.
- **incident-response-for-ai.md** *(coming)* — Incident classes that are specific to AI systems: cost incident (runaway cost), quality incident (regression in production quality), provider outage, model deprecation surprise (the vendor silently changed a model behind an alias), and the runbook templates for each.

## How to use this section

**If your team is taking on the on-call for AI features for the first time**, `timeout-strategy.md`, `retry-strategy.md`, and `incident-response-for-ai.md` are the starter pack. The other patterns build on top of those primitives.

**If you have AI features but no degraded modes**, `degraded-mode-design.md` is the highest-leverage reliability investment available. Degraded mode is the difference between a quality incident that users notice and one that does not page.

**If you are facing model provider rate-limit pressure**, `capacity-planning.md` and `multi-provider-failover.md` together describe the options. The right answer is usually multi-tenancy of provider quotas before multi-provider.

## What this section is not

- **A general SRE primer.** SLO design, error budgets, runbook practice, blameless postmortems — the SRE canon is mature and well-covered. This folder is about the AI-specific overlays.
- **A status-page-engineering guide.** Communicating incidents externally is a related and separate discipline; not in scope here.
