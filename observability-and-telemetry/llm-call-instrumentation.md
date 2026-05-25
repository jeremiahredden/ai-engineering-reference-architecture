# LLM Call Instrumentation

> **Audience.** Engineers building or refactoring the wrapper that every LLM call passes through. Tech leads who want every model invocation across the platform to be observable, attributable, and cost-tracked uniformly. **Scope.** The *engineering* pattern for a single LLM-call wrapper. Builds on [trace-and-span-design.md](./trace-and-span-design.md) (which sets the trace framework) and applies it specifically to the LLM-call boundary. Not the agent-loop instrumentation ([agent-step-instrumentation.md](./agent-step-instrumentation.md) owns that). Not the architectural gateway pattern (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `guardrails-and-policy-architecture/ai-gateway-pattern.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Every LLM call in a production AI system should produce a uniform trace span with a uniform attribute set. The team that achieves this can answer "which prompt version was running when this interaction happened" and "what did each model call cost" and "how did time-to-first-token vary across providers" — all from the trace alone. The team that does not achieve this answers these questions by greping logs, correlating timestamps, and reading source code.

The engineering pattern is straightforward: every LLM call goes through a single wrapper; the wrapper captures the full attribute set described in this document; the wrapper handles streaming, cached-input accounting, and error classification uniformly. The wrapper *is* the instrumentation; there is no other path.

The discipline this enforces sounds obvious but is rarely achieved. Most teams have a primary LLM-call path that is well-instrumented, plus several secondary paths (an eval runner, a batch job, a tool that wraps a model call) that are minimally instrumented or use ad-hoc attributes. The cumulative effect is that trace and cost data is inconsistent across the system, and cross-feature analysis requires translation between attribute schemas.

This document is opinionated about three things:

1. **One wrapper, one path.** Every LLM call goes through the wrapper. Lint and code review enforce. The wrapper is a small piece of code in front of the SDK, not a complex framework.
2. **Cost is computed at call time, in the wrapper.** Not derived from the provider invoice. The wrapper has the token counts, the model, the pricing table; it computes cost and emits the span attribute. This is the foundation for cost-attribution and circuit-breakers (see [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)).
3. **Streaming is first-class.** Time-to-first-token, time-to-useful-content, and total latency are separately captured. Streaming-specific failure modes (stream closes mid-response, idle-timeout) are detected by the wrapper, not by the consumer.

Structure: (2) the wrapper interface; (3) attribute capture; (4) cost computation; (5) streaming instrumentation; (6) error and finish-reason classification; (7) cached-input accounting; (8) provider quirks; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The wrapper interface

The wrapper is a thin layer in front of every LLM provider SDK. It exposes a uniform method that the rest of the platform calls; internally it dispatches to the right provider SDK based on the requested model.

### 2.1 The signature

```python
class LLMClient:
    def call(
        self,
        provider: str,                  # "anthropic" | "openai" | etc.
        model: str,                     # provider-specific model identifier
        messages: list[Message],
        *,
        prompt_version: str,            # versioned prompt artifact ID
        context: CallContext,           # tenant, user, feature, session, trace
        max_tokens: int | None = None,
        temperature: float | None = None,
        tools: list[Tool] | None = None,
        stream: bool = False,
        timeout_ms: int = 30000,
    ) -> LLMResponse | LLMStreamingResponse:
        ...
```

The signature is uniform across providers. Provider-specific knobs (Anthropic's system parameter, OpenAI's response_format, Google's safety settings) are either normalized into the wrapper's parameters or passed via a provider_specific dict that the wrapper records but does not interpret.

The `context` parameter is required (no default). It carries the attribution dimensions: tenant ID, user ID, feature ID, session ID, interaction ID, parent trace context. The wrapper uses it to enforce the per-call attribute population and the per-tenant/per-feature cost circuits.

The `prompt_version` parameter is required. The pinned prompt version that was used to build the messages. Without it, "which prompt was running" cannot be reconstructed from the trace; with it, every span has the answer.

### 2.2 The wrapper's responsibilities

What the wrapper does on every call:

1. Open a span (named `llm_call`) with the universal attributes (tenant, user, feature, session, interaction).
2. Populate the `gen_ai.*` attributes (provider, model, parameters) and the `ai.llm.*` extensions (prompt version, etc.).
3. Estimate cost before the call (max-output assumption).
4. Pass through the circuit-breaker check (per the cost-budget circuit-breaker pattern).
5. Dispatch to the provider SDK.
6. For streaming responses: instrument time-to-first-token, monitor for idle, record each chunk size.
7. For non-streaming: record latency, finish reason, output tokens.
8. Compute actual cost; record on span; emit to cost-aggregation pipeline.
9. Classify any error / non-success finish reason; emit span events.
10. Close the span; return the response (or streaming wrapper) to the caller.

What the wrapper does NOT do:

- Build the messages. The caller is responsible for prompt assembly; the wrapper consumes finished messages.
- Decide retries. Retry policy is the caller's concern (the wrapper exposes the failure information to the caller).
- Implement guardrails. Input / output filtering is the AI gateway's responsibility (the wrapper sits inside the gateway in most architectures).
- Choose models or tiers. Routing is upstream; the wrapper is invoked with the model already chosen.

### 2.3 The "only path" discipline

The provider SDKs are private to the wrapper. No application code imports the Anthropic SDK, the OpenAI SDK, the Google GenAI SDK, or any other model provider's SDK directly. Lint rules enforce this; code review rejects PRs that violate it; new contributors are onboarded with the discipline.

Without this discipline, the wrapper's instrumentation guarantees do not hold. With it, every LLM call in the platform is uniformly observed.

---

## 3. Attribute capture

The wrapper populates a comprehensive attribute set on every span. The taxonomy aligns with the OpenTelemetry `gen_ai.*` semantic convention where applicable and extends with `ai.llm.*` where the convention does not cover.

### 3.1 Provider and model

| Attribute | Source | Example |
|---|---|---|
| `gen_ai.system` | Wrapper parameter | `anthropic`, `openai`, `google`, `azure-openai`, `bedrock` |
| `gen_ai.request.model` | Wrapper parameter | `claude-opus-4-7`, `gpt-5`, `gemini-2.5-pro` |
| `ai.llm.model_version` | Wrapper resolves from registry | `claude-opus-4-7@2026-04-12` |
| `ai.llm.model_alias_resolved` | True if input was alias | True / False |

The model_version attribute is the pinned version (full string, not the alias). If the caller passed an alias, the wrapper resolves and records both the alias used and the version it resolved to — making alias-related drift visible.

### 3.2 Request parameters

| Attribute | Source |
|---|---|
| `gen_ai.request.temperature` | parameter |
| `gen_ai.request.top_p` | parameter |
| `gen_ai.request.top_k` | parameter |
| `gen_ai.request.max_tokens` | parameter |
| `gen_ai.request.stream` | True / False |
| `ai.llm.request.tool_count` | If tools were provided, how many |
| `ai.llm.request.tool_names` | If tools were provided, their names |
| `ai.llm.request.response_format` | `text` / `json_object` / `tool_call` |

The tool-related attributes let you see at a glance which calls had tools available — useful for distinguishing agent-step calls from non-agent calls.

### 3.3 Prompt versioning

| Attribute | Source |
|---|---|
| `ai.llm.prompt.version` | parameter (required) |
| `ai.llm.prompt.id` | Resolved from version (the prompt-store ID) |
| `ai.llm.prompt.hash` | SHA-256 of the assembled message list |
| `ai.llm.prompt.message_count` | Number of messages in the request |
| `ai.llm.prompt.system_present` | Boolean |

The prompt hash is useful for de-duplicating runs that used the same exact prompt (e.g., during eval) without needing to compare full message arrays.

### 3.4 Token counts

| Attribute | Source |
|---|---|
| `gen_ai.usage.input_tokens` | Provider response (or tokenizer if not returned) |
| `gen_ai.usage.output_tokens` | Provider response |
| `ai.llm.usage.input_tokens_cached` | Provider response (when supported) |
| `ai.llm.usage.input_tokens_uncached` | Computed: total input minus cached |
| `ai.llm.usage.cache_hit_rate` | Computed: cached / total input |

The cached / uncached split matters for cost (cached tokens often priced 10x cheaper) and for prompt-prefix caching optimization (high cache hit rate means the prompt-prefix structure is working).

### 3.5 Cost

| Attribute | Source |
|---|---|
| `ai.cost.usd` | Computed by wrapper from tokens × pricing table |
| `ai.cost.usd.input` | Sub-component |
| `ai.cost.usd.input_cached` | Sub-component (often near-zero) |
| `ai.cost.usd.output` | Sub-component |

Cost is computed at call time, written to the span, and also emitted to the cost-aggregation pipeline (Redis counters for circuit-breaker checks, longer-term storage for the FinOps dashboard).

### 3.6 Latency

| Attribute | Source |
|---|---|
| `ai.llm.latency.queue_ms` | Time between wrapper invocation and provider call start |
| `ai.llm.latency.connect_ms` | Time to establish provider connection (when measurable) |
| `gen_ai.response.latency.ttft_ms` | Time to first token (streaming) |
| `ai.llm.latency.last_token_ms` | Time to last token (streaming) |
| `gen_ai.response.latency.total_ms` | Total latency from wrapper entry to response close |
| `ai.llm.latency.stream_idle_max_ms` | Longest gap between streamed chunks (streaming) |

Latency is split into stages so the failure mode is diagnosable. A high TTFT with low last-token-ms suggests the provider was slow to start but fast once started. A long stream_idle_max suggests stream interruption.

### 3.7 Response classification

| Attribute | Source |
|---|---|
| `gen_ai.response.finish_reasons` | Provider response (array, in case of multiple turns) |
| `ai.llm.response.had_tool_calls` | Boolean |
| `ai.llm.response.tool_call_count` | Integer |
| `ai.llm.response.tool_names_called` | Array of tool names |
| `ai.llm.response.error` | If errored, the classified error type |
| `ai.llm.response.refusal` | If model refused (content policy), the classification |

The finish-reason and refusal attributes are critical for the post-call analysis — distinguishing "the model gave a clean answer" from "the model was cut off by max_tokens" from "the model refused due to content policy" from "the model returned tool calls."

---

## 4. Cost computation

Cost is computed at call time, in the wrapper. The pattern:

### 4.1 The pricing table

A versioned data structure (Python dict, YAML file, or config-as-code) mapping provider × model × token-type to per-million-token price. Updated on every provider pricing change.

```yaml
anthropic:
  claude-opus-4-7:
    input_per_million_usd: 15.00
    input_cached_per_million_usd: 1.50
    output_per_million_usd: 75.00
  claude-sonnet-4-6:
    input_per_million_usd: 3.00
    input_cached_per_million_usd: 0.30
    output_per_million_usd: 15.00
  ...

openai:
  gpt-5:
    input_per_million_usd: 10.00
    output_per_million_usd: 50.00
  ...
```

The wrapper looks up the price by provider + model and computes:

```python
def compute_cost(provider: str, model: str, input_tokens: int,
                 input_cached_tokens: int, output_tokens: int) -> float:
    prices = PRICING_TABLE[provider][model]
    uncached_input = input_tokens - input_cached_tokens
    return (
        uncached_input * prices["input_per_million_usd"] / 1_000_000
        + input_cached_tokens * prices["input_cached_per_million_usd"] / 1_000_000
        + output_tokens * prices["output_per_million_usd"] / 1_000_000
    )
```

### 4.2 The pricing-table maintenance discipline

- The table is in version control; updates are PRs.
- The table version is recorded on the span (so cost numbers can be re-attributed if a pricing table bug is discovered later).
- A monthly reconciliation compares the wrapper-computed total cost against the provider invoice; significant discrepancies trigger pricing-table validation.
- New models added by the provider require pricing-table updates before they can be used through the wrapper; the wrapper refuses to dispatch a call to an unknown model.

### 4.3 The pre-call estimate

For circuit-breaker decisions, the wrapper computes an estimated cost *before* the call:

```python
def estimate_cost(provider: str, model: str, input_messages: list,
                  max_output_tokens: int) -> float:
    input_tokens = _count_tokens(provider, model, input_messages)
    return compute_cost(provider, model,
                        input_tokens=input_tokens,
                        input_cached_tokens=0,  # conservative
                        output_tokens=max_output_tokens)  # max
```

The estimate is intentionally conservative: assumes no cache hit (likely under-estimates cache savings) and assumes full max-output (likely over-estimates output cost). This makes the circuit-breaker error on the side of stopping slightly too early, which is the right direction.

### 4.4 Cost telemetry pipeline

After the actual cost is computed, the wrapper:
- Sets the cost attribute on the span.
- Writes the cost to the per-aggregation-level counters (Redis for per-tenant per-day, per-feature per-day; in-process for per-interaction and per-session).
- Emits a cost event to the longer-term cost data store (for FinOps dashboards).

The wrapper is the single source of truth for "this call cost this much." All downstream uses (dashboards, circuit breakers, chargebacks) consume from this signal.

---

## 5. Streaming instrumentation

Streaming responses are harder to instrument than non-streaming responses. The wrapper handles the patterns:

### 5.1 Time-to-first-token (TTFT)

The first signal the user sees in a streamed response. Often the most-watched latency metric for chat-shaped workloads.

The wrapper:
- Records the timestamp when the provider stream opens.
- Records the timestamp when the first content chunk arrives.
- Emits `gen_ai.response.latency.ttft_ms` as the difference.

For Meridian chat: TTFT SLO is < 1.5s p95. Production traces let on-call see at any time which features / tenants / models are hitting or missing the SLO.

### 5.2 Time to useful content

Some responses start with non-substantive content (acknowledgements, structured-output preamble). For chat-shaped workloads, "time to useful content" is more meaningful than TTFT but harder to define automatically.

The wrapper does not attempt to define it; it captures TTFT (the first-chunk time) and the consumer can layer its own definition.

### 5.3 Last-token and idle-gap

The wrapper records:
- `ai.llm.latency.last_token_ms`: the time of the last content chunk.
- `ai.llm.latency.stream_idle_max_ms`: the longest gap between chunks observed during the stream.

A long `stream_idle_max_ms` suggests the stream was interrupted (network glitch, provider hiccup) and may need a re-stream or a fallback.

### 5.4 Stream errors

Streams can error mid-flight:
- The connection is reset.
- The provider returns an error chunk.
- The stream goes idle past the idle timeout.

The wrapper classifies these:
- `ai.llm.response.error.kind`: `connection_reset` / `provider_stream_error` / `idle_timeout` / `client_cancel`.
- `ai.llm.response.error.bytes_received_before_error`: how much content arrived before the error.

This classification matters for the consumer's response. A connection_reset late in a long response may be recoverable by partial-response use; an idle_timeout often indicates the provider is having trouble and a different model should be tried.

### 5.5 Token-count accuracy for streamed responses

Some providers return token counts only at stream completion; others return per-chunk counts; others require client-side tokenization for output counts. The wrapper normalizes:

- If provider returns final counts: use them.
- If provider returns per-chunk counts: sum.
- Otherwise: tokenize the assembled output client-side using the provider's tokenizer.

The attribute `ai.llm.usage.output_tokens.source` records which method was used (`provider_final`, `provider_per_chunk`, `client_tokenizer`), so cost computation can be back-checked against actual invoices.

---

## 6. Error and finish-reason classification

Provider responses come in many shapes. The wrapper normalizes them so consumers can reason uniformly.

### 6.1 Finish reasons

OpenTelemetry's `gen_ai.response.finish_reasons` enum:
- `stop`: normal completion
- `length`: hit max-tokens
- `content_filter`: safety / content policy
- `tool_calls`: returned tool calls (continue the loop)
- `error`: errored

The wrapper maps each provider's finish-reason strings into this enum. Provider-specific finish reasons that do not fit cleanly get a `provider_specific` value and the raw string in `ai.llm.response.finish_reason.raw`.

### 6.2 Error classification

When the call errors (or the stream errors mid-flight), the wrapper classifies:

| `ai.llm.response.error.kind` | Meaning |
|---|---|
| `rate_limited` | Provider rate limit (RPM or TPM) |
| `timeout` | Request exceeded configured timeout |
| `provider_5xx` | Provider returned a 5xx (transient) |
| `provider_4xx` | Provider returned a 4xx (permanent for this request) |
| `auth_failure` | Authentication / authorization failure |
| `content_policy` | Provider refused on content policy |
| `connection_error` | Network failure |
| `stream_idle_timeout` | Stream went idle past threshold |
| `circuit_breaker_tripped` | Pre-call circuit breaker rejected |
| `model_not_found` | Pricing-table check rejected (model not registered) |

This classification drives the consumer's retry decision and the on-call alert decision. Rate-limited errors are different from auth_failures; provider_5xx errors are different from content_policy errors.

### 6.3 Refusal handling

The model refused to answer (content policy, safety, alignment). Distinct from an error:

- `ai.llm.response.refusal`: True
- `ai.llm.response.refusal.classification`: provider-reported reason
- The response content is whatever the model returned (often a refusal message)

For Meridian's Care Coordinator: clinical questions occasionally trigger model-side refusal (e.g., questions about controlled substances). The refusal classification feeds into the workflow's escalation logic — refusals route to human escalation rather than being retried.

---

## 7. Cached-input accounting

Most major providers in 2026 support prompt-prefix caching: the input prefix is cached server-side and re-used across calls, often at 10x lower cost than uncached input. The wrapper captures cache utilization explicitly.

### 7.1 The attributes

- `ai.llm.usage.input_tokens_cached`: cached portion (from provider response).
- `ai.llm.usage.input_tokens_uncached`: total minus cached.
- `ai.llm.usage.cache_hit_rate`: cached / total input.

### 7.2 Why it matters

The cache hit rate is the leading indicator for prompt-prefix-cache optimization. Healthy multi-turn conversational systems show 60–80% cache hit rates on second-turn-and-later interactions. Sub-20% cache hit rates suggest the prompt structure is invalidating the cache between calls (often due to non-stable elements early in the prompt).

The cost difference is large enough that cache utilization is a first-order cost lever. A 70% cache hit rate on a 5K-token input cuts cost roughly 60%.

### 7.3 Cache-utilization dashboards

The wrapper's cost attributes feed dashboards that show:
- Per-feature cache hit rate trend.
- Cache-hit-rate by prompt version (so a new prompt that invalidates cache is visible immediately).
- Per-tenant cache hit rate (some tenants' workloads have more cache-friendly patterns than others).

When cache hit rate drops, the diagnostic playbook is:
1. Compare against a baseline week to confirm regression.
2. Check if a prompt version landed recently; correlate timing.
3. Inspect the new prompt for non-stable elements early in the assembly.
4. Refactor the prompt to push variable content to the end of the messages.
5. Re-deploy and verify cache hit rate recovers.

---

## 8. Provider quirks

Each provider has API quirks that the wrapper normalizes. The wrapper exists in part to absorb these.

### 8.1 Anthropic

- System prompts are a top-level parameter, not a message.
- Tool calls return as content blocks alongside text.
- Streaming uses `message_delta` events with cumulative usage.
- Prompt caching is configured per content block (`cache_control` parameter).

The wrapper accepts a unified `messages` parameter that may include a system message; internally it routes the system message to Anthropic's `system` parameter when dispatching.

### 8.2 OpenAI

- System messages are role=system within the messages list.
- Tool calls return as `tool_calls` field on assistant messages.
- Streaming uses Server-Sent Events with `data: {chunk}` lines.
- Prompt caching is automatic for system prompts and tools (no explicit parameter).

The wrapper handles the role-based message structure and normalizes OpenAI's tool-call format to the platform's standard tool-call representation.

### 8.3 Google Gemini

- System instructions are a separate parameter from messages.
- Tool calls use Google's specific function-call format.
- Safety settings are configured per call.

The wrapper normalizes Google's specific structures into the platform's uniform interface.

### 8.4 Azure OpenAI

- Uses OpenAI's API surface but with Azure-specific authentication and endpoint routing.
- Per-deployment configuration matters.

The wrapper treats Azure OpenAI as a separate provider for attribute purposes (`gen_ai.system: azure-openai`) so per-provider analysis is clean.

### 8.5 Bedrock

- Each model has its own request / response format (Anthropic on Bedrock differs from OpenAI on Bedrock differs from Cohere on Bedrock).
- Bedrock-specific authentication (AWS SigV4).
- Cost can be tracked at the AWS account level or at the model level.

The wrapper exposes a `gen_ai.system: bedrock` value and sub-attributes for the specific model family. Cost is computed using the Bedrock pricing table (which differs from the underlying providers' direct-API pricing).

### 8.6 The general principle

The wrapper absorbs provider-specific structure. Application code sees a uniform interface. Adding a new provider is a wrapper-level change; no application code changes.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The wrapper invocation

For a typical Care Coordinator chat interaction, the supervisor call looks like:

```python
response = llm_client.call(
    provider="anthropic",
    model="claude-opus-4-7",
    messages=assembled_messages,
    prompt_version="supervisor@2.4.1",
    context=CallContext(
        tenant_id="mercy-cleveland",
        user_id="u-mercy-cleveland-rn-001234",
        user_role="rn",
        feature_id="care-coordinator",
        session_id="session-2026-05-25-a7b8-3",
        interaction_id="interaction-2026-05-25-a7b8",
        trace_context=current_trace_context(),
    ),
    max_tokens=2000,
    temperature=0.2,
    tools=supervisor_tools,
    stream=True,
    timeout_ms=15000,
)
```

The wrapper opens the span, populates all the universal and LLM-specific attributes, runs the circuit-breaker check, dispatches to Anthropic, instruments the streamed response, computes cost, closes the span, and returns the streaming response wrapper.

### 9.2 Sample span

```
span: llm_call
duration: 2.1s

attributes:
  ai.trace.interaction_id: interaction-2026-05-25-a7b8
  ai.tenant.id: mercy-cleveland
  ai.feature.id: care-coordinator
  ai.user.id: u-mercy-cleveland-rn-001234
  ai.user.role: rn
  ai.session.id: session-2026-05-25-a7b8-3
  ai.cost.usd: 0.058

  gen_ai.system: anthropic
  gen_ai.request.model: claude-opus-4-7
  ai.llm.model_version: claude-opus-4-7@2026-04-12
  ai.llm.model_alias_resolved: False

  gen_ai.request.temperature: 0.2
  gen_ai.request.max_tokens: 2000
  gen_ai.request.stream: True
  ai.llm.request.tool_count: 6
  ai.llm.request.tool_names: ["dispatch_classifier","dispatch_clinical_knowledge",
                              "dispatch_drafting","dispatch_query_rewriter",
                              "consult_drug_interaction_graph","finalize_answer"]

  ai.llm.prompt.version: "supervisor@2.4.1"
  ai.llm.prompt.id: "prompt-care-coord-supervisor"
  ai.llm.prompt.hash: "9c2a1f8b..."
  ai.llm.prompt.message_count: 12
  ai.llm.prompt.system_present: True

  gen_ai.usage.input_tokens: 3247
  ai.llm.usage.input_tokens_cached: 2890
  ai.llm.usage.input_tokens_uncached: 357
  ai.llm.usage.cache_hit_rate: 0.89
  gen_ai.usage.output_tokens: 412

  ai.cost.usd.input: 0.0535
  ai.cost.usd.input_cached: 0.0043
  ai.cost.usd.output: 0.0049
  ai.cost.usd: 0.0627

  gen_ai.response.latency.ttft_ms: 980
  ai.llm.latency.last_token_ms: 2050
  gen_ai.response.latency.total_ms: 2103
  ai.llm.latency.stream_idle_max_ms: 95

  gen_ai.response.finish_reasons: ["tool_calls"]
  ai.llm.response.had_tool_calls: True
  ai.llm.response.tool_call_count: 2
  ai.llm.response.tool_names_called: ["dispatch_classifier","dispatch_clinical_knowledge"]
```

This span is fully self-contained. An incident investigator can read this and know:
- Which prompt version was running (supervisor@2.4.1).
- Which model version (claude-opus-4-7@2026-04-12, not the alias).
- The cache hit rate (89%, healthy).
- TTFT (980ms, within p95 budget).
- The supervisor's decision (dispatched to classifier and clinical-knowledge in parallel).
- The cost ($0.063, within per-interaction budget).

### 9.3 The pricing table

Meridian maintains `meridian/llm_client/pricing_table.yaml` with the current pricing for every model in use. The table is updated by ai-platform-eng when providers announce price changes. Monthly reconciliation against invoices catches discrepancies.

Models not in the table cannot be invoked through the wrapper — the wrapper rejects the call with a `model_not_found` error. This forces the deliberate adoption decision (add the new model to the pricing table; security-review; eval against the workload) before production use.

### 9.4 Vendor routing

The Care Coordinator uses Anthropic for the supervisor and clinical-knowledge worker, Anthropic for drafting and classifier, Cohere for reranking, OpenAI for embeddings. The wrapper handles all five — different `gen_ai.system` values, different attribute populations, uniform interface.

If one provider has a partial outage, the team can swap a model in the routing layer (e.g., supervisor falls back to claude-sonnet-4-6 via the same provider or to gpt-5 via OpenAI); the wrapper just dispatches the new call with the new attributes. No application code change.

### 9.5 Cache utilization metrics

The cache hit rate dashboard shows:
- Per-feature: care-coordinator-supervisor averages 87%; clinical-knowledge-worker averages 82%; drafting-worker averages 65% (less repetition across calls); classifier averages 94% (small stable prompt).
- Per prompt version: a new supervisor prompt that landed dropped cache hit rate from 87% to 71%; investigation found a non-stable timestamp in the system prompt's first 200 tokens; fix moved the timestamp later in the assembly; cache hit rate recovered.

### 9.6 The platform discipline

- `meridian.llm_client` is the only public API for LLM calls. Direct SDK imports lint-rejected outside the package.
- The pricing table is in version control; updates require ai-platform-eng + finops review.
- Monthly invoice reconciliation runs against the cost-aggregation store; >2% discrepancy triggers investigation.
- New models go through a registration process: pricing-table entry, security review, eval-suite check, then enablement.

---

## 10. Anti-patterns

### 10.1 "Wrapper is optional"

The team built a wrapper but it is one of several paths to the LLM SDKs. Application code that imports the SDKs directly is in production. Wrapper coverage is partial; trace coverage is partial.

**Corrective.** Wrapper is the only path. Lint rule against direct SDK imports. Code review enforces.

### 10.2 "Cost is computed post-hoc from invoices"

The wrapper does not compute cost; it relies on monthly reconciliation with provider invoices. Per-call cost data does not exist; circuit breakers cannot be built; per-tenant chargeback is monthly-batch.

**Corrective.** Compute cost at call time. The invoice is the verification, not the source.

### 10.3 "Provider SDK swap requires application-code changes"

Adding or swapping a provider requires changes scattered across the codebase. The wrapper is too thin to insulate the application from provider differences.

**Corrective.** Wrapper absorbs provider quirks. Application code sees uniform interface. Provider changes are wrapper-only.

### 10.4 "Streaming instrumentation is whatever the SDK provides"

The team uses the provider SDK's default streaming; the trace shows total latency but not TTFT, not stream idle, not per-chunk timing. Latency analysis is coarse.

**Corrective.** Wrapper instruments streaming explicitly. TTFT, last-token, idle-gap, error classification are first-class attributes.

### 10.5 "Cached / uncached input tokens are not distinguished"

The wrapper sums all input tokens into one attribute. Cache hit rate is invisible; cost optimization opportunities are missed; per-prompt cache regression cannot be detected.

**Corrective.** Capture cached and uncached separately. Compute cache hit rate. Surface in dashboards.

### 10.6 "Prompt version is not on the span"

LLM-call spans show the model and the latency and the cost but not the prompt version. When the team needs to ask "which prompt was running," they have to correlate by timestamp against the deployment log.

**Corrective.** Prompt version is a required parameter and a span attribute. Every call carries the answer.

### 10.7 "Pricing table is hard-coded constants"

The cost calculation has model prices hard-coded in the wrapper source. Provider price changes require a code deploy. Reconciling against invoices requires reading the code.

**Corrective.** Pricing table as data (YAML, config, database). Version-controlled. Updateable without code deploy. Versioned table attribution on spans.

### 10.8 "Finish reasons are not classified"

The wrapper records the provider's raw finish_reason string. Consumers handle it inconsistently — one path treats `max_tokens_exceeded` as success, another as failure.

**Corrective.** Normalize to the OpenTelemetry enum. Consumers handle the normalized values; the wrapper handles the provider-specific translation.

---

## 11. Findings (sprint-assignable)

### OBS-LLM-001 — Severity: Critical
**Finding.** LLM calls go through multiple paths; instrumentation coverage is partial.
**Recommendation.** Single wrapper as the only path; lint rule against direct SDK imports; code review enforces.
**Owner.** ai-platform-eng, sprint N+1.

### OBS-LLM-002 — Severity: Critical
**Finding.** Cost is computed by reconciling provider invoices; no per-call cost telemetry exists.
**Recommendation.** Cost computed at call time in the wrapper; pricing table in code; emit to the cost-aggregation pipeline.
**Owner.** ai-platform-eng + finops, sprint N+1.

### OBS-LLM-003 — Severity: High
**Finding.** Prompt version is not captured in LLM-call telemetry; quality regressions cannot be traced to prompt changes.
**Recommendation.** Make `prompt_version` a required parameter; populate the span attribute on every call.
**Owner.** ai-platform-eng + prompt-engineering, sprint N+1.

### OBS-LLM-004 — Severity: High
**Finding.** Model is referenced by alias (e.g., `claude-opus-latest`); model version drift is invisible.
**Recommendation.** Pin to full model version strings; record both alias used and resolved version on the span; reject aliases in production calls.
**Owner.** ai-platform-eng, sprint N+1.

### OBS-LLM-005 — Severity: High
**Finding.** Streaming spans show total latency only; TTFT, last-token, idle-gap are not measured.
**Recommendation.** Streaming-specific instrumentation per section 5; TTFT as a first-class SLI.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-LLM-006 — Severity: High
**Finding.** Cached / uncached input tokens are not distinguished; cache utilization is unknown.
**Recommendation.** Capture cached split per section 7; surface cache hit rate per feature / per prompt version.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-LLM-007 — Severity: High
**Finding.** Finish reasons are recorded as raw provider strings; consumers handle them inconsistently.
**Recommendation.** Normalize to the OpenTelemetry enum per section 6.1; consumers consume normalized values.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-LLM-008 — Severity: High
**Finding.** Error classification is not uniform; rate-limit vs auth-failure vs timeout vs content-policy are not distinguishable from the span.
**Recommendation.** Implement the error-classification taxonomy (section 6.2); consumers' retry logic depends on it.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-LLM-009 — Severity: Medium
**Finding.** Pricing table is hard-coded in wrapper source; provider price changes require code deploy.
**Recommendation.** Move to YAML or config-as-code; version on the span; monthly reconciliation against invoices.
**Owner.** ai-platform-eng + finops, sprint N+3.

### OBS-LLM-010 — Severity: Medium
**Finding.** New models can be invoked without pricing-table entry; cost computation silently returns zero.
**Recommendation.** Wrapper rejects calls for models not in the pricing table with `model_not_found`; force the deliberate enablement decision.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-LLM-011 — Severity: Medium
**Finding.** Stream idle-timeout is not detected; mid-stream interruptions look like normal completion.
**Recommendation.** Idle-timeout detection per section 5.4; classify as `stream_idle_timeout` error.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-LLM-012 — Severity: Medium
**Finding.** Tool-call presence on responses is not separately tracked; agent-step vs non-agent calls cannot be distinguished from spans.
**Recommendation.** `ai.llm.response.had_tool_calls`, `tool_call_count`, `tool_names_called` per section 3.7.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-LLM-013 — Severity: Medium
**Finding.** Refusals are conflated with errors; consumers do not have a clean signal to escalate vs retry.
**Recommendation.** Separate `ai.llm.response.refusal` boolean and classification per section 6.3.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-LLM-014 — Severity: Medium
**Finding.** Token counting for streamed responses uses the provider's final count, which is inconsistent across providers.
**Recommendation.** Per provider, document and implement the counting path; record `ai.llm.usage.output_tokens.source`.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-LLM-015 — Severity: Medium
**Finding.** Cache-hit-rate trends are not surfaced; prompt-induced cache regressions are invisible until cost spikes.
**Recommendation.** Cache-hit-rate dashboard per feature / per prompt version; alert on regression > 15 percentage points.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### OBS-LLM-016 — Severity: Low
**Finding.** Latency stages (queue, connect, TTFT, total) are not separately tracked; diagnostics treat latency as one number.
**Recommendation.** Stage breakdown per section 3.6; surface in latency dashboards.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-LLM-017 — Severity: Low
**Finding.** Provider-specific quirks leak into application code; provider swap is harder than necessary.
**Recommendation.** Wrapper absorbs provider quirks (section 8); application code sees uniform interface.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-LLM-018 — Severity: Low
**Finding.** Monthly invoice reconciliation against the wrapper-computed cost is not scheduled; pricing-table drift is undetected.
**Recommendation.** Schedule monthly reconciliation; alert on > 2% discrepancy.
**Owner.** ai-platform-eng + finops, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team adding an LLM-call wrapper to a feature that does not have one:

- [ ] **Sprint 0 — design.** Define the wrapper interface (section 2). Inventory existing LLM-call paths; plan the consolidation.
- [ ] **Sprint 1 — wrapper.** Build the wrapper for one provider (typically the primary). Pricing table in YAML. Universal attributes populated.
- [ ] **Sprint 1 — migrate primary path.** Migrate the main production LLM-call path to the wrapper. Verify trace coverage.
- [ ] **Sprint 2 — cost telemetry.** Cost computed at call time; emitted to cost-aggregation pipeline; surfaces in FinOps dashboards.
- [ ] **Sprint 2 — additional providers.** Add second and third provider support. Verify uniform interface holds.
- [ ] **Sprint 3 — streaming.** Streaming-specific instrumentation. TTFT, idle-gap, error classification.
- [ ] **Sprint 3 — migrate remaining paths.** Eval runners, batch jobs, tools that wrap LLM calls — all to the wrapper.
- [ ] **Sprint 4 — enforcement.** Lint rule against direct SDK imports. Code review process documented. New provider additions go through review.
- [ ] **Sprint 5 — cache instrumentation.** Cached / uncached split; cache-hit-rate dashboard; alerting on regression.
- [ ] **Ongoing — pricing table maintenance.** Monthly invoice reconciliation. Provider pricing-change PRs go through finops review.

A team that completes this sequence has the single source of truth for "this call cost this much" and "this prompt version produced this output." Every downstream observability, FinOps, and reliability concern (circuit breakers, drift detection, on-call investigation) builds on this foundation.

---

## 13. References

- OpenTelemetry `gen_ai.*` semantic conventions (2024-2026).
- Anthropic, OpenAI, Google, Azure OpenAI, Bedrock provider documentation — specifically the streaming and usage-reporting sections.
- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — the foundation framework.
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](./agent-step-instrumentation.md) — the per-agent-step wrapper that uses this LLM-call wrapper inside.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — the circuit-breaker pattern consuming the cost attributes this wrapper emits.
- This repo: [cost-and-finops/cost-attribution.md](../cost-and-finops/) (coming) — the cost-aggregation pipeline that consumes cost emissions.
- This repo: [prompt-engineering/prompt-versioning.md](../prompt-engineering/) (coming) — the prompt-versioning practice this wrapper depends on.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/) (coming) — the architecture in which the LLM-call wrapper typically lives.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked architecture using these patterns.
