# Timeout Strategy

> **Audience.** Engineers calibrating timeouts for AI workloads. SREs whose default 30-second HTTP timeout is timing out 8% of agent calls. Anyone who copy-pasted timeout values from a non-AI service and is now wondering why the customer's UI is hanging. **Scope.** The *engineering* practice of timeout calibration for AI: latency distributions and why P99 is 5x P50; per-LLM-call timeout; streaming-aware patterns (TTFT, inter-token, total-duration); per-tool-call and per-agent-turn timeouts; timeout-as-cost-control role; connection-pool / network timeouts. Not the broader integration-shape decision (see [ai-architecture-reference-architecture / integration-architecture / sync-vs-async-vs-streaming.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/sync-vs-async-vs-streaming.md)). Not the retry policy (see [retry-strategy.md](./retry-strategy.md), companion). Not the circuit-breaker primitive (see [circuit-breakers.md](./circuit-breakers.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Timeouts are a primitive reliability concern, but AI workloads break the assumptions that timeout defaults are based on. Conventional services have tight, predictable latency distributions: P50 is some milliseconds, P99 is a small multiple of P50, and a 30-second timeout is generous safety margin. AI workloads have wildly different distributions:

- LLM call P50 may be 1.2 seconds; P99 may be 8 seconds (~7x).
- Long-context calls may take 30+ seconds even at P50.
- Streaming responses can take 60+ seconds for long outputs.
- Agent tasks (multi-step) can take 1-3 minutes routinely.

A "default 30s timeout" that works fine for a REST service times out 5-15% of legitimate LLM calls; agent timeouts at 30s break entire workflows.

The complication: longer timeouts have direct cost implications. Each timeout-protected request consumes a connection slot, a worker, and (for streaming) bandwidth. Indefinite timeouts make a single bad request consume resources indefinitely. The architectural goal is timeouts long enough to allow legitimate slow calls but short enough to bound resource consumption.

Beyond resource concerns, timeouts in AI workloads serve a unique additional role: cost control. An LLM call without a timeout can run away in output length; the bill accumulates per output token. A timeout caps the cost per call.

This document covers the calibration: how to set timeouts to the actual distribution; the streaming-aware patterns that conventional total-duration timeouts can't express; the per-tool and per-agent layers that compose into total request budgets; and the dual purpose as cost-control mechanism.

This document is opinionated about four things:

1. **Timeouts must be calibrated to the actual P99 (plus margin), not to defaults.** A 30-second default timeout that times out 8% of calls is producing false failures. Calibrate from production data.
2. **Streaming requires three timeouts, not one.** Time-to-first-token, inter-token, and total-duration each catch different failure modes. A single "total timeout" misses streams that hang silently.
3. **Multi-step workloads need a cascading timeout budget.** The total-request budget divides among LLM calls, tool calls, agent steps, and overhead. Each component must respect its allocated sub-budget, not assume it has the full budget.
4. **A timeout is also a cost cap.** An LLM call without a timeout is a check that can be cashed indefinitely. Tight timeouts bound worst-case cost per call.

Structure: (2) latency distributions and the P99 reality; (3) the latency budget cascade; (4) per-LLM-call timeout; (5) streaming-aware timeouts; (6) per-tool-call timeout; (7) per-agent-turn and per-agent-task timeout; (8) timeout as cost-control; (9) connection-pool / network-layer timeouts; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. Latency distributions and the P99 reality

The starting point for calibration.

### 2.1 Typical LLM-call latency distribution

For Claude Sonnet 4.6 (representative, 2026):

- P50: ~1200ms (short prompt, short output).
- P75: ~2400ms.
- P95: ~5500ms.
- P99: ~8200ms.
- P99.9: ~15,000ms.

The distribution is right-skewed; the tail is long. P99 is ~7x P50.

For long-context workloads (8k+ input tokens):

- P50: ~3500ms.
- P95: ~9000ms.
- P99: ~14,000ms.

For agent tasks (multi-call):

- P50: ~12,000ms.
- P95: ~35,000ms.
- P99: ~80,000ms.

### 2.2 The distribution shape (per model and per workload)

- Short-form classification: tight distribution; P99 close to P50.
- Long-context analysis: long tail; P99 far above P50.
- Agent tasks: very long tail; P99 may be 5-10x P50.

Each workload has its own distribution; one timeout doesn't fit all.

### 2.3 The calibration data source

Production telemetry. Per-workload latency histograms over a window (last 30 days). The P99 of that distribution informs the timeout.

Without production data: use provider-published expectations as starting point; recalibrate after first month of production data.

### 2.4 The "2x margin" rule

A common pattern: timeout = 2x P99 of the observed distribution.

- Catches the legitimate P99 calls.
- Provides margin for outliers (P99.9 may be 1.5-2x P99).
- Cuts off truly stuck calls (P99.99+ is usually a stuck call, not a slow one).

For a Claude Sonnet workload with P99=8200ms, timeout = 16,000ms. Conservative; minimizes false timeouts.

### 2.5 The "tighter timeout for cost control" trade-off

Cost-sensitive workloads may use tighter timeouts:

- Timeout = 1.2x P99.
- Tighter cap on worst-case cost per call.
- More false timeouts (P99.5 calls fail; they would have succeeded with the wider margin).

Choose per workload. UX-critical: wider margin. Cost-sensitive batch: tighter.

### 2.6 The "P99 changes over time" reality

The distribution drifts:

- Provider-side changes (capacity, model updates).
- Workload changes (input size, prompt complexity).
- Network conditions.

Recalibrate quarterly. Track P99 trend; adjust timeouts if drift is significant (e.g., > 30%).

### 2.7 Per-percentile decision

Instead of one timeout, multiple actions at different percentiles:

- P99 timeout: cancel and fall back.
- P95 latency: surface as alert metric (not action; just visibility).
- P90 latency: routine; no action.

Layers handle different scenarios.

---

## 3. The latency budget cascade

When multiple components contribute to total request latency, each gets a sub-budget.

### 3.1 The cascade structure

For a user-facing request:

```
Total user-facing budget:           5000ms (caller's SLO)
  ↓
  Connection setup:                    50ms
  Authentication / authorization:     100ms
  Per-tenant rate-limit check:         10ms
  Retrieval (vector store):           300ms
  Prompt assembly:                     20ms
  LLM call (the big one):           4000ms
  Output validation:                   50ms
  Cache write + response serialize:    50ms
  Headroom for retry / variance:      420ms
```

The LLM call gets 4000ms of the 5000ms budget. If it exceeds, the entire request fails to meet SLO.

### 3.2 The hierarchical timeout

Each component's timeout is its allocated budget:

```python
# Total request budget
with timeout(5000):
    auth_result = authenticate()  # timeout 100ms
    with timeout(300):
        retrieved = retrieve_context(query)
    with timeout(4000):
        response = call_llm(prompt, context)
    with timeout(50):
        validated = validate(response)
    return validated
```

Each layer respects its sub-budget; failure in any layer triggers timeout handling.

### 3.3 The "leave headroom for retry" pattern

If the LLM call's budget is 4000ms and we retry once on timeout, we need 2 × 4000ms = 8000ms; exceeds the 5000ms total. Either:

- No retry (single attempt).
- Tighter LLM-call timeout (2000ms with retry; total 4000ms).
- Wider total budget.

Decision per workload. Most UX-critical: no retry, wide timeout. Most batch: retry with tighter per-attempt timeout.

### 3.4 The asymmetric cascade

For long-running agents, the cascade is asymmetric:

```
Total budget:                300,000ms (5 min; user expects async)
  ↓
  Per-agent-step timeout:    30,000ms
  Per-LLM-call timeout:      10,000ms (within agent step)
  Per-tool-call timeout:      5,000ms (within agent step)
  Per-retrieval timeout:        500ms (within agent step)
```

Each layer has its allocation; agents have many steps within their total.

### 3.5 The "soft timeout" vs "hard timeout"

- **Hard timeout.** Cancel the work; return failure (or fall back).
- **Soft timeout.** Continue the work but log a warning / metric; don't cancel.

For LLM calls, hard timeout is standard. For long agent tasks where partial work is valuable, soft timeout (warn but continue) at the upper end is sometimes used.

### 3.6 The dependency-aware cascade

When a downstream service is the bottleneck:

```
Total budget:           5000ms
  ↓
  LLM call A:           2000ms
  ↓ (LLM A's output)
  LLM call B:           2000ms (sequential; depends on A)
  Combined:             4000ms (+ 1000ms overhead)
```

Sequential LLM calls within one user request budget. Each must respect its sub-budget; total must fit.

---

## 4. Per-LLM-call timeout

The single most important timeout to calibrate.

### 4.1 The default

A reasonable starting timeout for LLM calls:

- Short-form (classification, extraction): 5,000ms.
- Medium (chat, summarization): 10,000ms.
- Long-context (analysis): 30,000ms.
- Reasoning-heavy (Claude Opus extended thinking, OpenAI o-series): 60,000ms+.

Adjust based on production P99.

### 4.2 The provider SDK default

Most provider SDKs have their own timeout defaults:

- Anthropic Python SDK: 600 seconds (10 min) by default — too generous.
- OpenAI Python SDK: 600 seconds default.
- AWS Bedrock SDK: variable.

Override the SDK default; set the timeout you actually want.

### 4.3 The timeout vs max_tokens relationship

`max_tokens` caps output length; timeout caps wall-clock time. Both matter:

- A 4000-token output at ~25 tokens/sec = 160 seconds. If timeout is 30s, max_tokens is effectively limited by timeout.
- max_tokens=200 with timeout=30s; output finishes in ~8s; timeout doesn't bind.

For cost control, both are levers; for latency control, timeout is the primary.

### 4.4 The streaming case

For streaming, the timeout is more nuanced (cross-link to §5). The total-duration timeout still applies; additional timeouts cover stream-specific failures.

### 4.5 The provider-side timeout

Providers may have their own timeout (e.g., Anthropic disconnects after extended idle). Stay within the provider's tolerance to avoid spurious disconnects.

For most workloads, the provider's timeout is not the binding constraint (your timeout is shorter).

### 4.6 The "abort on timeout" cleanup

When a timeout fires:

- The LLM call may have been in-flight; you may still be billed.
- Output (if any partial) is discarded.
- Resources (connection, worker) are released.

Idempotency for retry depends on whether the provider charges for the timed-out call (typically yes if the call reached the model).

### 4.7 The estimate-aware timeout

For very long calls, the timeout can be adaptive:

```python
def adaptive_timeout(model, input_tokens, max_tokens):
    estimated_output_time = max_tokens / tokens_per_sec_for(model)
    estimated_processing_time = input_tokens / input_processing_rate_for(model)
    return (estimated_output_time + estimated_processing_time) * 2  # 2x margin
```

More accurate than static; harder to debug when wrong. Use for workloads with high latency variance (e.g., Claude Opus extended thinking).

---

## 5. Streaming-aware timeouts

Streaming responses break the single-timeout pattern.

### 5.1 The three streaming timeouts

For a streamed response, three independent timeouts:

- **TTFT (Time To First Token).** From request submission to first token. Typically 100-2000ms.
- **Inter-token timeout.** Maximum gap between consecutive tokens. Typically 5-30 seconds.
- **Total-duration timeout.** Maximum total stream duration. Typically 60-300 seconds.

Each catches a different failure mode.

### 5.2 The TTFT failure mode

If TTFT exceeds threshold, the stream is stuck at startup:

- Provider is queueing.
- Network is broken.
- Connection setup failed silently.

Threshold typically: provider's documented TTFT × 3 or so. If TTFT > 5 seconds, something is wrong.

```python
def stream_with_ttft_timeout(prompt, ttft_ms=5000):
    start = time.now()
    stream = create_stream(prompt)
    for token in stream:
        if time.now() - start > ttft_ms and not first_token_seen:
            raise TimeoutError("TTFT exceeded")
        first_token_seen = True
        yield token
```

### 5.3 The inter-token failure mode

If gap between tokens exceeds threshold, the stream is mid-generation hung:

- Provider stalled.
- Network buffering issue.
- Model is in a long "thinking" pause.

Threshold typically: 30 seconds for normal generation; 60 seconds for reasoning models with extended thinking.

### 5.4 The total-duration failure mode

Even with tokens flowing, total runaway:

- Model is verbose past expectation.
- Output is far exceeding max_tokens (rarely; provider should respect max_tokens).
- Overall request budget exceeded.

Threshold typically: max_tokens × time-per-token × 1.5.

### 5.5 The composite check

All three timeouts run concurrently:

```python
def stream_with_full_timeouts(prompt, ttft=5000, inter_token=30_000, total=120_000):
    start = time.now()
    last_token_time = start
    first_token_seen = False
    
    stream = create_stream(prompt)
    
    for token in stream:
        now = time.now()
        
        if not first_token_seen and (now - start) > ttft:
            raise TimeoutError("TTFT exceeded")
        
        if first_token_seen and (now - last_token_time) > inter_token:
            raise TimeoutError("Inter-token timeout exceeded")
        
        if (now - start) > total:
            raise TimeoutError("Total stream duration exceeded")
        
        first_token_seen = True
        last_token_time = now
        yield token
```

Each timeout independent; first one to fire wins.

### 5.6 The client-side display

In streaming UIs, the timeouts can be surfaced:

- Long TTFT: show "thinking..." indicator.
- Long inter-token: show "still generating..." indicator.
- Approach to total-duration: warn user "this is taking longer than usual."

UX-aware streaming surfaces the wait without alarming.

### 5.7 The reasoning-model variant

Models with extended thinking (Claude Opus extended thinking, OpenAI o-series) may have very long TTFT during thinking. Adjust timeouts:

- TTFT: up to 60 seconds (thinking can take that long).
- Inter-token: same (once tokens start flowing, normal pace).
- Total-duration: longer to accommodate thinking time.

---

## 6. Per-tool-call timeout

For agent workloads, each tool call has its own timeout.

### 6.1 The tool-call latency profile

Tool calls vary:

- Local function: < 1ms.
- HTTP API call: 50-500ms.
- Database lookup: 10-200ms.
- Vector search: 50-500ms.
- External third-party (slow): 5-30 seconds.

Each tool's timeout is calibrated to its expected latency.

### 6.2 The tool-call timeout policy

```python
TOOL_TIMEOUTS = {
    "fetch_patient": 1000,
    "search_clinical_notes": 5000,
    "external_eligibility_check": 15000,
    "send_notification": 2000,
}
```

Each tool has a documented timeout; agent dispatcher enforces.

### 6.3 The "slow tool blocks agent" risk

If a tool times out within an agent loop, the agent waits the full timeout duration. For a slow tool with 30s timeout, the agent's progress is blocked 30s.

Mitigations:
- Tighten tool timeouts to actual P99 + margin.
- Run tool calls in parallel where the agent allows.
- Use cached results for slow tools where freshness allows.

### 6.4 The "tool returned slowly; budget exhausted" cascade

A 5-step agent with 5-second budget per step has 25s budget. If step 2's tool returns in 10s (within its individual timeout but using twice its allotted time), step 5 may not complete within total.

Pattern: aggregate remaining budget; later steps have less time. Cross-link to §3.4.

### 6.5 The retry within a tool

Tool implementations may retry internally (e.g., HTTP client retries network errors). The tool's external timeout includes its internal retries; consumer doesn't know.

Pattern: bound tool's internal retries so total time stays within external timeout. Don't let internal retries blow past the timeout.

### 6.6 The cancellation semantics

When a tool times out, the underlying operation may continue:

- HTTP request may still be in flight.
- Database transaction may complete after the agent stops waiting.
- External API call may produce side effects after timeout.

For tools with side effects, the timeout doesn't reverse the side effect. Idempotency at the tool boundary is needed.

---

## 7. Per-agent-turn and per-agent-task timeout

For agent workloads, timeouts at the turn and task level.

### 7.1 The per-turn timeout

A "turn" is one round of: model generates → tool calls → tool returns → model receives → repeat or done.

Per-turn timeout = LLM call timeout + max tool call timeout + overhead.

For Care Coordinator (Claude Sonnet, ~5 tools): ~15-20 seconds per turn.

### 7.2 The per-task timeout

A "task" is the agent's full execution: many turns until done.

Per-task timeout = number-of-expected-turns × per-turn-timeout × safety factor.

For Care Coordinator (~5 turns per task): 5 × 15 × 1.5 = ~120 seconds task timeout.

### 7.3 The "agent ran too long" failure

If task timeout fires:

- Agent's in-progress state is preserved (for postmortem; cross-link to [agent-engineering/error-and-partial-failure.md](../agent-engineering/error-and-partial-failure.md)).
- Task is marked failed.
- Caller receives structured error.
- Side effects from completed steps may be either committed or rolled back per the workflow's design.

### 7.4 The "soft" task timeout

For some workflows, the task continues past the soft timeout but the caller can stop waiting:

- Agent task is durable workflow (Temporal); continues in background.
- Caller's polling endpoint shows "still running."
- Eventual completion delivered via callback / webhook.

Cross-link to [ai-architecture-reference-architecture / integration-architecture / callback-and-webhook-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/callback-and-webhook-patterns.md).

### 7.5 The progressive timeout (per-step budget shrinking)

As the agent progresses:

- Step 1 starts with full budget.
- After step 1, remaining = total - step 1's actual duration.
- Step 2 starts with remaining.
- ...

If remaining falls below the next step's expected duration, the agent can abort early rather than time out mid-step.

### 7.6 The "agent stuck in a loop" detection

Loop detection is separate from timeout:

- Count of LLM calls per task.
- Number of identical or near-identical model outputs.
- Number of times the agent has called the same tool with the same args.

If these exceed thresholds, abort even if timeout hasn't fired. Cross-link to [agent-engineering/error-and-partial-failure.md](../agent-engineering/error-and-partial-failure.md).

---

## 8. Timeout as cost-control

A timeout is also a cost cap.

### 8.1 The cost-per-time calculation

For a streaming LLM call:

```
cost_per_second = output_tokens_per_second × cost_per_token
```

A timeout caps this:

```
max_cost_per_call = timeout_seconds × cost_per_second
```

For Claude Sonnet (~25 output tokens/sec at $0.015/1k tokens):

- 30-second timeout: max ~$0.011 output cost.
- 60-second timeout: max ~$0.023 output cost.

### 8.2 The runaway-output protection

Without a timeout, an LLM that's been given "max_tokens=8000" can generate for ~5 minutes; output cost up to $0.12. With a 30-second timeout, capped at ~$0.011.

The timeout is a hard cap; the model can't exceed it regardless of `max_tokens` setting.

### 8.3 The "tight timeout for cost" trade-off

Tighter timeouts:
- Lower cost cap.
- More false-positive timeouts on legitimate slow calls.

Wider timeouts:
- Higher cost cap.
- Fewer false-positives.

For cost-sensitive bulk workloads, tighter is better. For UX-sensitive real-time, wider is better. Tune per workload.

### 8.4 The combination with max_tokens

max_tokens and timeout together:

- max_tokens caps total output length.
- Timeout caps wall-clock time.
- Whichever binds first wins.

For predictable cost, set both.

### 8.5 The agent task cost cap

For agent tasks:

- Per-call timeout × max number of calls = max calls budget time.
- Per-call cost × max calls = max cost per task.

A task with 30s per-call × 10 calls × $0.04/call = $0.40 max per task. Cap aligns with task-level budget.

Cross-link to [agent-engineering/agent-cost-control.md](../agent-engineering/agent-cost-control.md).

### 8.6 The "very long timeout" red flag

A timeout >> P99 is a red flag:

- Cost cap is unnecessarily high.
- Stuck requests sit indefinitely.
- Operational signal to investigate why timeout was set so high.

Quarterly review of timeouts vs P99.

---

## 9. Connection-pool and network-layer timeouts

The lower-layer timeouts that compose with application timeouts.

### 9.1 The connection-setup timeout

For HTTP clients, separate from request timeout:

```python
client = httpx.AsyncClient(
    timeout=httpx.Timeout(
        connect=2.0,     # connection setup
        read=30.0,       # individual read
        write=2.0,       # individual write
        pool=5.0         # waiting for connection from pool
    )
)
```

- Connection timeout: TCP / TLS handshake.
- Read timeout: any single read from response.
- Write timeout: any single write to request.
- Pool timeout: waiting for a free connection in the pool.

Each has its own purpose; default settings often too generous.

### 9.2 The "pool exhausted" failure

If the connection pool is exhausted (all connections busy), new requests wait for the pool timeout. Higher pool timeout = more requests pile up.

Tighter pool timeout (e.g., 2-5 seconds) fails fast; surfaces as a capacity issue rather than queueing silently.

### 9.3 The TLS / DNS layer

DNS resolution timeouts, TLS handshake timeouts — usually invisible but can cause sporadic failures. Connection-setup timeout should be tight enough to surface DNS / TLS issues quickly.

### 9.4 The keep-alive interaction

HTTP keep-alive reuses connections, saving connection-setup latency:

- First request: full connect + TLS + request time.
- Subsequent requests: just request time.

Keep-alive matters for high-volume workloads; saves 50-200ms per request on the second+ call.

### 9.5 The provider's keep-alive support

Most providers support keep-alive; verify. Some restrict idle time (e.g., 5 minutes) before closing.

For low-volume workloads with sparse calls, keep-alive may not help (connection closes before next use).

### 9.6 The retry interaction at the network layer

Network-layer retries (HTTP client built-in) compose with application-layer retries:

- HTTP client may retry on connection errors.
- Application may retry on 5xx responses.

Both retry pathways multiply; effective retry count is higher than either alone. Cross-link to [retry-strategy.md](./retry-strategy.md).

---

## 10. Anti-patterns

### 10.1 The "default 30s for everything" timeout

**Pattern.** All AI calls have 30s timeout. Long-context calls fail at 30s. Agent tasks fail at 30s.

**Corrective.** Per-workload calibration per §4.1. Different timeouts for different call types.

### 10.2 The single timeout for streaming

**Pattern.** Streaming has one total-duration timeout. A stream that produces no tokens for 60s doesn't fire the timeout (because total <60s).

**Corrective.** Three timeouts (TTFT + inter-token + total) per §5.

### 10.3 The provider SDK default timeout

**Pattern.** Provider SDK has 10-minute default. Stuck calls sit for 10 minutes; resources held; cost accumulates.

**Corrective.** Override SDK defaults to your calibrated timeout per §4.2.

### 10.4 The "I'll calibrate later" deferral

**Pattern.** Default timeouts used initially; "we'll tune them after launch." Months pass; tuning never happens; false-positive timeouts cause customer complaints.

**Corrective.** Calibrate before launch using staging data; refine after first month of production data.

### 10.5 The timeout = max_tokens × tokens_per_sec calculation only

**Pattern.** Timeout computed strictly from expected output time; ignores latency variance and TTFT.

**Corrective.** Add margin per §2.4 (2x P99). Variance is real.

### 10.6 The agent timeout that's just per-call summed

**Pattern.** Agent task timeout = number of expected calls × per-call timeout. Doesn't account for tool call time, overhead, or variance in number of calls.

**Corrective.** Per-task timeout includes safety factor per §7.2.

### 10.7 The cascade that doesn't add up

**Pattern.** Total request budget is 5s; sum of component budgets is 8s. Components individually pass their timeouts; total fails.

**Corrective.** Cascade math per §3.1. Sum of components + overhead ≤ total budget.

### 10.8 The "no timeout means no failures" reasoning

**Pattern.** Timeouts removed because they cause "false failures." Stuck calls now sit indefinitely; resource exhaustion follows.

**Corrective.** False failures are signal that timeout is too tight, not that timeout is bad. Re-calibrate.

### 10.9 The connection pool exhaustion

**Pattern.** All connections busy; pool timeout is generous (30s); requests pile up; cascading failure.

**Corrective.** Tight pool timeout (2-5s) per §9.2. Surface capacity issue early.

### 10.10 The streaming timeout based on token count instead of time

**Pattern.** "Timeout if no token in N tokens." Counts tokens; ignores wall-clock. A slow generation (1 token / 10 sec) fits but isn't useful.

**Corrective.** Time-based timeouts; tokens are signal but time is the primary cap.

---

## 11. Findings (sprint-assignable)

### REL-TO-001 — Severity: Critical
**Finding.** Provider SDK default timeout used (typically 10 minutes); not calibrated.
**Recommendation.** Override defaults; set per-workload timeouts per §4.1.
**Owner.** AI platform, sprint N+1.

### REL-TO-002 — Severity: Critical
**Finding.** Streaming uses single total-duration timeout; doesn't catch stuck streams.
**Recommendation.** Three timeouts (TTFT + inter-token + total) per §5.
**Owner.** AI platform, sprint N+1.

### REL-TO-003 — Severity: Critical
**Finding.** Per-call timeout not calibrated to actual P99.
**Recommendation.** Calibrate from production data per §2; 2x P99 default per §2.4.
**Owner.** AI platform + observability, sprint N+1.

### REL-TO-004 — Severity: High
**Finding.** No per-agent-task timeout.
**Recommendation.** Per-task timeout per §7.2; cap on total task duration.
**Owner.** agent platform, sprint N+2.

### REL-TO-005 — Severity: High
**Finding.** Latency budget cascade not designed.
**Recommendation.** Cascade per §3.1; sub-budgets per component.
**Owner.** AI platform, sprint N+2.

### REL-TO-006 — Severity: High
**Finding.** Per-tool-call timeouts undocumented.
**Recommendation.** Policy per §6.2; each tool has documented timeout.
**Owner.** agent platform + tool implementers, sprint N+2.

### REL-TO-007 — Severity: High
**Finding.** Connection pool timeouts at default (generous); request queueing under load.
**Recommendation.** Tight pool timeout (2-5s) per §9.2.
**Owner.** AI platform, sprint N+2.

### REL-TO-008 — Severity: Medium
**Finding.** Quarterly P99 review not scheduled; timeouts drift.
**Recommendation.** Quarterly review per §2.6; recalibrate if > 30% drift.
**Owner.** AI platform + observability, sprint N+3.

### REL-TO-009 — Severity: Medium
**Finding.** Reasoning-model timeouts not adjusted for extended thinking.
**Recommendation.** Wider TTFT allowance per §5.7.
**Owner.** AI platform, sprint N+3.

### REL-TO-010 — Severity: Medium
**Finding.** Loop detection not implemented for agent tasks.
**Recommendation.** Loop detection per §7.6; abort even if timeout hasn't fired.
**Owner.** agent platform, sprint N+3.

### REL-TO-011 — Severity: Medium
**Finding.** Timeout doesn't compose with max_tokens for cost control.
**Recommendation.** Both set per §8.4; cost cap explicit.
**Owner.** AI platform, sprint N+3.

### REL-TO-012 — Severity: Medium
**Finding.** Provider-side idempotency not used; retries after timeout double-bill.
**Recommendation.** Idempotency-key per call; cross-link to [retry-strategy.md](./retry-strategy.md).
**Owner.** AI platform, sprint N+4.

### REL-TO-013 — Severity: Medium
**Finding.** Asymmetric cascade not used for long-running agents.
**Recommendation.** Pattern per §3.4 for multi-step workloads.
**Owner.** agent platform, sprint N+4.

### REL-TO-014 — Severity: Medium
**Finding.** Streaming UX doesn't surface timeout state.
**Recommendation.** Display per §5.6; UX-aware indicators.
**Owner.** product + AI platform, sprint N+4.

### REL-TO-015 — Severity: Low
**Finding.** Adaptive timeout not used for high-variance workloads.
**Recommendation.** Adaptive per §4.7 for Claude Opus extended thinking and similar.
**Owner.** AI platform, sprint N+5.

### REL-TO-016 — Severity: Low
**Finding.** Per-call timeout not tagged in cost telemetry.
**Recommendation.** Tag timeout used; surface in cost dashboard.
**Owner.** AI platform + observability, sprint N+5.

### REL-TO-017 — Severity: Low
**Finding.** TCP / TLS / DNS timeouts not separated.
**Recommendation.** Configure each layer per §9.1 and §9.3.
**Owner.** AI platform, sprint N+5.

### REL-TO-018 — Severity: Low
**Finding.** Keep-alive support not verified per provider.
**Recommendation.** Verify per §9.5; document; enable.
**Owner.** AI platform, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Override provider SDK default timeouts (§4.2).**
- [ ] **Calibrate per-call timeouts from production P99 (§2).** Default to 2x P99.
- [ ] **Implement streaming-aware three-timeout pattern (§5).**
- [ ] **Design latency budget cascade for each user-facing flow (§3).**
- [ ] **Set per-tool-call timeouts (§6).** Documented policy per tool.
- [ ] **Set per-agent-turn and per-agent-task timeouts (§7).**
- [ ] **Implement loop detection for agents (§7.6).** Abort early.
- [ ] **Tight connection pool timeouts (§9.2).**
- [ ] **Quarterly P99 review for drift (§2.6).**
- [ ] **Reasoning-model timeout adjustments (§5.7).**
- [ ] **Document timeout-as-cost-cap (§8) for cost-sensitive workloads.**
- [ ] **Adaptive timeout for high-variance (§4.7).**
- [ ] **UX-aware streaming indicators (§5.6).**

---

## 13. References

**In this folder.**
- [fallback-patterns.md](./fallback-patterns.md) — fallback on timeout; cascade with timeout policy.
- [retry-strategy.md](./retry-strategy.md) — retry decisions after timeout; companion.
- [circuit-breakers.md](./circuit-breakers.md) — circuit-breaker on repeated timeouts; companion.
- [degraded-mode-design.md](./degraded-mode-design.md) — degraded mode triggered by timeout.

**Elsewhere in this repo.**
- [agent-engineering/agent-loop-design.md](../agent-engineering/agent-loop-design.md) — agent loop within timeout budget.
- [agent-engineering/agent-cost-control.md](../agent-engineering/agent-cost-control.md) — cost cap via timeout.
- [agent-engineering/error-and-partial-failure.md](../agent-engineering/error-and-partial-failure.md) — partial failure on timeout.
- [cost-and-finops/cost-aware-rate-limiting.md](../cost-and-finops/cost-aware-rate-limiting.md) — rate-limit interaction.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / sync-vs-async-vs-streaming.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/sync-vs-async-vs-streaming.md) — integration-shape decision.
- [ai-architecture-reference-architecture / integration-architecture / integration-failure-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/integration-failure-patterns.md) — failure-mode taxonomy including timeout.
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — timeout under backpressure.

**External.**
- Anthropic / OpenAI / Google client SDK documentation for timeout APIs.
- Google SRE Book — chapter on timeouts and deadlines.
- httpx documentation — TCP / TLS / pool timeout primitives.
- Hystrix legacy patterns (deprecated; concepts apply).
