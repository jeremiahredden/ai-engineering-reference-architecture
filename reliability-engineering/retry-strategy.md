# Retry Strategy

> **Audience.** Engineers whose retry logic was inherited from a non-AI service and is now producing $200/day in retry storms during provider degradations. SREs wondering why their "3 retries with exponential backoff" produces shockingly high bills. Anyone whose agent task retries from step 1 after step 5 failed. **Scope.** The *engineering* practice of retry strategy for AI workloads: the retry decision tree (which errors retry; which don't); idempotency analysis; exponential backoff with jitter; provider-aware backoff (Retry-After header); cached retry pattern to avoid double-billing; bounded retry budgets; workload-specific retry policies. Not the timeout that triggers retry (see [timeout-strategy.md](./timeout-strategy.md), companion). Not the fallback path when retries exhaust (see [fallback-patterns.md](./fallback-patterns.md), companion). Not the circuit-breaker that pauses retries (see [circuit-breakers.md](./circuit-breakers.md), companion). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Retry policy is a primitive reliability concern, but AI workloads change the rules in three specific ways:

- **Retries cost real money.** A retry on an LLM call may incur full call cost; aggressive retry policies during provider degradations produce shocking bills.
- **Many AI operations are non-idempotent.** Tool calls with side effects (sending email, writing to a system of record, updating customer state) cannot be naively retried; the side effect may already have happened.
- **Output variance means same-input-different-output.** A retry of an LLM call may produce a different response than the first attempt. "Same effective result" — the basis of conventional idempotency — doesn't apply at the LLM-output layer.

These properties combine to make naive retry policies (3 retries with exponential backoff for any error) a major cost and correctness liability. The right retry strategy is more nuanced:

- Specific error classes get specific retry behavior.
- Idempotency is analyzed per-operation, not assumed universally.
- Retry budgets are bounded at multiple levels (per-request, per-feature, per-tenant).
- Cached retry patterns avoid double-billing on infrastructure failures.
- Provider-aware backoff respects the provider's explicit signals.

This document covers the decision tree, the idempotency analysis, the backoff mechanics, and the cost-protection patterns that distinguish AI retry strategy from conventional service retry.

This document is opinionated about four things:

1. **The default retry pattern is wrong for AI.** "Retry 3 times with exponential backoff" was correct for stateless web requests; for LLM calls with cost and side-effect implications, it's a footgun.
2. **Retry is a per-operation decision, not a platform-wide policy.** Each tool, each LLM call, each pipeline step has its own retry profile based on idempotency and cost.
3. **Provider 429s must be honored, not retried.** Retrying through 429 counts against the limit and worsens the problem. Respect `Retry-After` always.
4. **Retries during provider degradation are the source of unbounded cost incidents.** Retry storms during a provider's bad hour can double or triple the bill. Cost-aware retry caps and circuit-breakers are non-optional.

Structure: (2) the retry decision tree by error class; (3) idempotency analysis; (4) exponential backoff with jitter; (5) provider-aware backoff (Retry-After); (6) the cached retry pattern; (7) bounded retry budget; (8) workload-specific retry policy; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The retry decision tree by error class

Each error class gets different retry treatment.

### 2.1 Network / transient errors

Connection refused, DNS failure, TLS handshake failure, network timeout before LLM call started.

**Retry.** Yes; the LLM call hasn't happened; retry is free (no cost). Bounded (2-3 retries) with exponential backoff.

### 2.2 Provider 5xx errors

Provider responds with 500, 502, 503, 504. The LLM call may or may not have processed.

**Retry.** Yes, but with caution. Idempotency-key on the call so the provider can dedup if the original did process. Backoff with jitter. Bounded (2-3 retries).

### 2.3 Provider 429 (rate limit)

Provider responds with 429 + Retry-After.

**Retry.** Yes, but respect Retry-After. Pause until the time elapses; then retry. Aggressive retry through 429 worsens the problem (each attempt counts toward limit).

### 2.4 Provider 4xx (other than 429)

400 (bad request), 401 (unauthorized), 403 (forbidden), 404, 422 (validation).

**Retry.** No. These indicate caller / config errors that retry won't resolve. Log; surface as failure; investigate.

### 2.5 Timeout (no provider response)

Client-side timeout fired before provider responded.

**Retry.** Sometimes. The LLM call may have processed (you may be billed); retry with idempotency-key. Bounded; consider widening timeout instead of retrying.

### 2.6 Provider response received, output is invalid

LLM responded with 200 OK but output is malformed (JSON parse failure, schema violation, truncated).

**Retry.** Sometimes. The model's non-determinism may produce a valid output on retry. Bounded (1-2 retries); accept failure after.

### 2.7 Provider response received, output is refusal

LLM declined to respond (safety filter, refusal message).

**Retry.** Rarely. A refusal usually indicates the model evaluated and chose; retry produces same. Specific cases (false-positive content filter) may justify one retry.

### 2.8 Provider response received, output is wrong but plausible

Output passes schema; semantics are incorrect (hallucinated entity, wrong number, factually wrong).

**Retry.** No. The model's output is "successful" by HTTP and schema; the wrongness is content-level. Retry won't fix; correction is via prompt engineering or output validation.

### 2.9 Tool-call failure

LLM emitted a tool call; tool's execution failed.

**Retry.** Depends on the tool (§3). Idempotent tools: yes. Non-idempotent: no.

### 2.10 Tool-call success but side effect already taken

LLM called a tool with side effect; tool succeeded; orchestration logic failed before recording the success. From the caller's perspective, the call "failed"; from the system's, it succeeded.

**Retry.** Never naively. The side effect has happened; retry produces double-execution. Either: idempotency at the tool boundary; or accept that the operation completed.

### 2.11 The decision matrix

| Error class | Retry? | Conditions |
| --- | --- | --- |
| Network transient | Yes | Bounded; exponential backoff |
| Provider 5xx | Yes | Idempotency-key; bounded; backoff |
| Provider 429 | Yes | Respect Retry-After |
| Provider 4xx | No | Investigate; fix; do not retry |
| Timeout | Sometimes | Idempotency-key; bounded |
| Invalid output | Sometimes | Bounded; accept failure |
| Refusal | Rarely | Only specific cases |
| Wrong-but-plausible | No | Retry won't help |
| Tool failure (idempotent) | Yes | Bounded |
| Tool failure (non-idempotent) | No | Side effect already taken |

---

## 3. Idempotency analysis

Whether retry is safe depends on idempotency. The analysis.

### 3.1 What "idempotent" means

An operation is idempotent if executing it twice produces the same effective system state as executing it once.

- A database write that uses upsert semantics: idempotent.
- An HTTP POST that creates a new record each call: non-idempotent.
- An email send: non-idempotent.

### 3.2 LLM calls are not idempotent at the response layer

Same input → different output (temperature > 0 or model non-determinism). So an LLM call isn't idempotent in the strict sense.

But the *side effects* of an LLM call can be made idempotent:

- The LLM's output is recorded once (cached by request ID).
- The side effects (DB writes, downstream calls) use idempotency keys.

The system as a whole becomes idempotent even though the LLM call itself isn't.

### 3.3 Provider-side idempotency

Major providers support an `Idempotency-Key` header (or similar). When you retry with the same key, the provider returns the cached response from the first call — no re-execution, no extra cost.

```python
response = anthropic.messages.create(
    model="...",
    messages=[...],
    extra_headers={"Idempotency-Key": f"req-{request_id}"}
)
```

Use for retry safety. The provider's cache keeps the original response available.

### 3.4 Tool-side idempotency

For tools with side effects:

- Tools that support idempotency-key headers: pass one.
- Tools without idempotency support: either wrap in an idempotency layer, or don't retry.

Per-tool documentation should declare idempotency posture.

### 3.5 The "side effect already taken" check

When unsure whether the side effect happened (timeout, connection lost mid-call):

- Query the system of record: did the write happen?
- Check the audit log: is there a record of execution?
- If the side effect happened, treat as success; don't retry.
- If not, retry is safe.

Adds latency but prevents double-execution.

### 3.6 The idempotency-key strategy

Choose the key carefully:

- **Request-level key.** Per-request UUID; survives across retries within the same request.
- **Workflow-level key.** Per-workflow-step; survives across workflow retries.
- **Composite key.** {tenant_id, request_id, step_id} for fine-grained dedup.

The key must be stable across retries (the same value passed each time) but unique across distinct requests (a new value for each new request).

### 3.7 The cache-the-LLM-response pattern

Within the consumer, cache the LLM response keyed by request ID:

```python
def call_llm_with_dedup(request_id, prompt):
    cached = response_cache.get(request_id)
    if cached:
        return cached
    
    response = llm_call(prompt, idempotency_key=request_id)
    response_cache.set(request_id, response, ttl=3600)
    return response
```

On retry within the same request, cache hits avoid the provider call entirely. Free retry.

### 3.8 The non-idempotent tool wrapper

For non-idempotent tools, wrap with an idempotency layer:

```python
def idempotent_send_email(idempotency_key, to, body):
    if email_sent_log.has(idempotency_key):
        return  # already sent; skip
    
    send_email(to, body)
    email_sent_log.record(idempotency_key)
```

The wrapper makes the tool effectively idempotent. The log is the source of truth.

---

## 4. Exponential backoff with jitter

The standard backoff pattern, AI-specific tuning.

### 4.1 The basic formula

```
backoff_ms = min(max_backoff, base_backoff × 2^retry_attempt) × jitter
```

Where:
- `base_backoff`: initial delay (typically 100-1000 ms).
- `max_backoff`: ceiling (typically 30-60 seconds).
- `2^retry_attempt`: exponential growth.
- `jitter`: random factor in [0.5, 1.5] to avoid synchronized retries.

### 4.2 The jitter rationale

Without jitter, all consumers retry at the same time:

- Attempt 1 fails → all retry at T+1s.
- Attempt 2 fails → all retry at T+2s.

If the provider was recovering, the synchronized retry storm knocks it back over. Jitter spreads attempts.

### 4.3 The "decorrelated jitter" variant

```
backoff_ms = random_uniform(base_backoff, prev_backoff × 3)
```

More aggressive jitter; even better at de-synchronization. Reference: AWS Architecture Blog "Exponential Backoff and Jitter" (2015).

### 4.4 The "respect Retry-After" override

If the provider returned `Retry-After: 30`, the exponential backoff is overridden:

```python
def compute_backoff(retry_attempt, retry_after_seconds):
    if retry_after_seconds:
        return retry_after_seconds * 1000
    return min(60_000, 100 * (2 ** retry_attempt)) * random_uniform(0.5, 1.5)
```

Provider knows best; respect their signal.

### 4.5 The "first retry is fast" pattern

Many transient errors resolve on the first retry. The first retry can be faster than the exponential schedule would suggest:

- Attempt 1: 100ms.
- Attempt 2: 500ms (faster than exponential 200ms).
- Attempt 3: 2000ms.
- Attempt 4: 8000ms.

Fast first retry catches the easy wins; subsequent retries back off properly.

### 4.6 The "compute backoff per-attempt" rather than pre-computed

Compute backoff at each retry based on current state, not a pre-allocated schedule. Allows reading the latest `Retry-After` header at each attempt.

### 4.7 The "don't retry too fast" lower bound

Some providers consider attempts within X ms as the same request. Retrying within 100ms may not actually re-attempt; the same response is returned.

Minimum backoff: usually 100-500ms.

---

## 5. Provider-aware backoff (Retry-After)

Honoring the provider's signal.

### 5.1 The Retry-After header

When the provider returns 429 (or sometimes 503), the response includes `Retry-After`:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

The integer is seconds; sometimes an HTTP date.

### 5.2 The fleet-wide pause

When 429 fires, pause not just this consumer but the fleet:

```python
def on_429(provider, retry_after_seconds):
    redis.set(f"provider_pause:{provider}", "1", ex=retry_after_seconds)

def can_call_provider(provider):
    return not redis.get(f"provider_pause:{provider}")
```

All consumers see the pause; resume together after Retry-After.

Cross-link to [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md).

### 5.3 The fleet-wide pause limit

Don't pause the entire fleet for very long Retry-Afters (e.g., 5 minutes). The architecture may want to:

- Route to fallback model.
- Queue requests.
- Surface 429 to callers.

Above a threshold (e.g., 60 seconds), fleet pause becomes counterproductive.

### 5.4 The "headroom-aware" pre-call check

Before calling, check headroom:

```python
def should_call(provider, model, estimated_tokens):
    headroom_tpm = redis.get(f"provider_headroom_tpm:{provider}:{model}") or 0
    return estimated_tokens < float(headroom_tpm) * 0.9
```

If headroom is low, defer / queue rather than risk 429.

### 5.5 The retry-after spike during provider incidents

During a major incident, Retry-After values can be very long (5+ minutes). Workload behavior:

- Real-time workloads: fall back to alternative model.
- Batch workloads: queue and wait.
- Best-effort workloads: drop / shed.

Per-workload policy on what to do with long Retry-After.

### 5.6 The "ignore Retry-After at your peril" warning

Some teams ignore Retry-After and retry immediately. This:

- Counts toward the rate limit again.
- Prolongs the 429 condition.
- Worsens the incident.

Always respect Retry-After.

---

## 6. The cached retry pattern

Avoiding double-billing on retries.

### 6.1 The scenario

LLM call A is sent; provider processes it (you're billed); response is lost (network blip, consumer crash, timeout); consumer retries with new connection.

Without idempotency: provider processes B; you're billed twice; response sent.

With provider idempotency-key + cache: provider returns A's cached response; B not processed; you're billed once.

### 6.2 The cache layers

Three layers of "don't re-bill":

- **Provider-side.** Idempotency-Key header; provider caches.
- **Consumer-side response cache.** Cache by request ID; on retry, return cached.
- **Application-level transactional log.** Records which calls were billed; doesn't trust retry signal.

All three are useful; each catches a different scenario.

### 6.3 The provider-side cache TTL

Idempotency-key cache TTL varies by provider (typically 24 hours for Anthropic; ask provider). Within the TTL, retry hits the cache.

Beyond TTL: retry incurs new call.

### 6.4 The consumer-side response cache

```python
@cached(ttl=300)
def cached_llm_call(request_id, prompt, **kwargs):
    return llm_call(prompt, idempotency_key=request_id, **kwargs)
```

Caches the response in the consumer's process / shared Redis. Subsequent calls with the same request_id return cached.

5-minute TTL is typical; tune based on retry timing patterns.

### 6.5 The "call-attempted" log

For debugging:

```python
def llm_call_with_log(request_id, prompt):
    log.info("llm_call_attempt", request_id=request_id, prompt_hash=hash(prompt))
    try:
        return cached_llm_call(request_id, prompt)
    except Exception as e:
        log.warn("llm_call_failed", request_id=request_id, error=str(e))
        raise
```

Logs every attempt; useful for "did the call happen?" questions during incidents.

### 6.6 The "$X saved by caching retries" metric

Track:

```
metric: cached_retry_savings_usd
  Each retry that hits the cache is saved cost.
  Sum over a period reflects retry-storm protection.
```

During provider degradations, this metric should spike (many cached retries avoided real calls).

---

## 7. Bounded retry budget

Multiple levels of retry budget bound the total.

### 7.1 The per-request retry budget

Maximum retries for any single LLM call: typically 2-3.

```python
@retry(max_attempts=3, backoff=exponential)
def call_with_retry(prompt):
    return llm_call(prompt)
```

After 3 attempts, give up. Surface as failure.

### 7.2 The per-feature retry budget

A feature may have many concurrent requests; each retry. Per-feature retry budget caps total:

```python
def call_within_feature_retry_budget(feature, prompt):
    if feature_retry_counter.exceeded(feature):
        return llm_call(prompt)  # no retry; budget exhausted
    
    response, retry_count = call_with_retry(prompt)
    feature_retry_counter.add(feature, retry_count)
    return response
```

When feature retry budget is exhausted (e.g., 1000 retries in last hour), no more retries for that feature until budget resets.

### 7.3 The per-tenant retry budget

Similar to per-feature; ensures one tenant's retry storm doesn't exhaust shared budget.

### 7.4 The aggregate retry budget

Platform-wide: total retries per hour. If exceeded, retries paused globally.

### 7.5 The retry budget recovery

Budgets refill over time:

- Per-request: resets each request.
- Per-feature: token-bucket refill (e.g., 100 retries/min).
- Per-tenant: same as feature.
- Aggregate: token-bucket.

### 7.6 The "circuit-breaker on retry storm" trigger

When retry rate is very high (e.g., > 30% of all calls), it indicates provider degradation or systemic issue. Circuit-breaker fires:

- Stop retrying for N minutes.
- Surface 429s / failures to callers directly.
- Alert on-call.

Prevents retry storms from compounding the incident. Cross-link to [circuit-breakers.md](./circuit-breakers.md).

---

## 8. Workload-specific retry policy

Different workloads have different retry profiles.

### 8.1 User-facing chat

- Real-time; latency matters.
- Retry: 1 attempt at most (UX can't tolerate multi-second waits for retries).
- Fallback: smaller model on first failure.

### 8.2 Bulk classification (batch)

- Latency-tolerant; cost-sensitive.
- Retry: 3 attempts with longer backoff.
- Fallback: queue for later if still failing.

### 8.3 Agent task

- Long-running; cost-aware.
- Retry: per-step; not whole-task restart.
- Idempotency: per-step idempotency keys.
- Cross-link to [agent-engineering/error-and-partial-failure.md](../agent-engineering/error-and-partial-failure.md).

### 8.4 Real-time clinical decision support

- Safety-critical; latency-sensitive.
- Retry: 1 attempt; if fails, escalate to human.
- No fallback to weaker model (would change quality posture).

### 8.5 Background enrichment

- No user waiting.
- Retry: aggressive (5+ attempts) with backoff.
- Fallback: skip / queue for next batch.

### 8.6 The per-workload retry config

```yaml
workloads:
  patient-chat:
    max_retries: 1
    backoff_base_ms: 200
    fallback: smaller_model_on_failure
    
  clinical-decision-support:
    max_retries: 1
    backoff_base_ms: 500
    fallback: escalate_to_human
    
  document-classification:
    max_retries: 3
    backoff_base_ms: 1000
    fallback: queue_for_later
    
  care-coordinator-agent:
    max_retries: 2
    backoff_base_ms: 500
    per_step_retry: true
    fallback: continue_with_partial
```

Per-workload retry config is documented; consumed by the retry layer.

### 8.7 The retry-policy override per call

Some calls override the default:

- "This specific call must not be retried" → no_retry flag.
- "This specific call needs aggressive retry" → max_retries=5.

Override is per-call; documented; tested.

---

## 9. Worked Meridian example

Meridian's retry strategy survived two provider incidents in the last 12 months.

### 9.1 The retry config per workload

```yaml
care-coordinator:
  default_max_retries: 2
  per_step_retry: true
  on_429: pause_fleet
  on_5xx: retry_with_backoff
  on_timeout: retry_once_with_idempotency_key
  fallback: continue_with_partial (cross-link to agent error-handling)

patient-api-chat:
  default_max_retries: 1
  on_429: fall_back_to_haiku
  on_5xx: fall_back_to_haiku
  on_timeout: no_retry
  fallback: cached_response_if_available

document-classification:
  default_max_retries: 3
  on_429: pause_and_retry
  on_5xx: retry_with_backoff
  fallback: queue_for_later

bulk-embedding:
  default_max_retries: 5
  on_429: pause_and_retry
  on_5xx: retry_with_backoff
  fallback: queue_for_later

analytics-warehouse-copilot:
  default_max_retries: 2
  on_429: queue
  on_5xx: retry_with_backoff
  fallback: cached_or_default
```

Each workload's retry profile reflects its tolerance.

### 9.2 The provider-aware backoff infrastructure

Redis-backed fleet pause:

```python
def call_provider(provider, prompt, request_id):
    if redis.get(f"provider_pause:{provider}"):
        raise ProviderPaused()
    
    try:
        return anthropic.messages.create(
            model="...",
            messages=[...],
            extra_headers={"Idempotency-Key": request_id}
        )
    except RateLimitError as e:
        retry_after = e.response.headers.get("Retry-After", 30)
        redis.set(f"provider_pause:{provider}", "1", ex=int(retry_after))
        raise
```

When 429 fires, fleet pauses for Retry-After seconds. All consumers see the pause.

### 9.3 The Q1 2026 provider degradation

Anthropic had a 12-minute incident; elevated 429 rate.

- 429s fired across consumers.
- Fleet pause activated for Retry-After (45 seconds initially; grew to 120 seconds).
- Patient API chat fell back to Haiku (per workload config).
- Document classification queued.
- Care Coordinator: 8 in-flight tasks paused mid-step; resumed after.

Total cost impact: ~$120 (mostly latency; cached retries saved most of the cost).

Without the strategy:
- Retries would have continued through 429s, prolonging the incident.
- Each retry would have incurred cost.
- Estimated: $2-4k extra cost during the 12-minute window.

### 9.4 The Q3 2025 retry-storm incident

Before the cost-aware retry policy:

- A timeout config bug caused 30s timeouts to fire on calls that would have completed in 35s.
- Retry policy retried every timeout.
- Each retry produced a new full LLM call.
- Volume: 2000 calls/hour got 3 retries each.
- Effective: 8000 calls/hour vs intended 2000.
- Cost overage: $800 over the window.

Resolution: widened timeouts (now calibrated to P99); added idempotency-key; added cached retry pattern. Subsequent timeout incidents cost < $50.

### 9.5 The per-feature retry budget

- Care Coordinator: 500 retries/hour. Rarely hit.
- Document classification: 2000 retries/hour. Occasionally hit during provider issues.
- Bulk embedding: 5000 retries/hour. Hit during the Q1 incident; protected against runaway.

When a feature exhausts its budget:
- New requests fail-without-retry.
- Customer-success aware via alert.
- Budget refills over the next hour.

### 9.6 The retry savings metric

Tracked monthly:

- Cached retries (provider idempotency-key hit): saves $1-3k/month routine; ~$8k during incidents.
- Provider pause prevents 429-storm cost: ~$2k/month.

Total estimated retry-policy savings: $3-5k/month routine; $10-20k during major incidents.

### 9.7 The runbook integration

The retry strategy has a runbook entry:

- Symptoms: cost spike correlated with provider error rate.
- Investigation: check retry metrics; identify if storm.
- Mitigation: pause retries (or tighten retry config); cross-link to [cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md).

### 9.8 What the strategy costs

- Engineering: ~2 weeks initial build (1 engineer); ongoing maintenance minimal.
- Infrastructure: Redis cluster shared with rate-limit infrastructure.
- Operational: occasional tuning during incidents.

### 9.9 The lessons learned

- Default retry policies from non-AI services are wrong.
- Provider idempotency-key is non-optional; eliminates double-billing.
- Per-workload retry config is more important than generic policy.
- Fleet-wide pause is much better than per-consumer retry.

---

## 10. Anti-patterns

### 10.1 The "retry 3 times on any error" default

**Pattern.** Generic retry policy applied to all errors. 4xx errors retry (wasted); refusals retry (wasted); same-input-different-output expected (wasted retries).

**Corrective.** Per-error-class decision per §2.

### 10.2 The retry without idempotency-key

**Pattern.** Retry on timeout; no idempotency-key. Provider re-processes; double-billed. Storm during outage = massive bill.

**Corrective.** Provider idempotency-key per §3.3.

### 10.3 The retry through 429

**Pattern.** Retry policy ignores 429; retries immediately. Each retry counts toward rate limit; prolongs the 429 condition; worsens the storm.

**Corrective.** Respect Retry-After per §5; fleet-wide pause per §5.2.

### 10.4 The retry of the entire agent task

**Pattern.** Agent task fails at step 5 of 7; retry policy restarts from step 1. Steps 1-4 re-run (expensive); same failure at 5 (likely).

**Corrective.** Per-step retry per §8.3 and [agent-engineering/error-and-partial-failure.md](../agent-engineering/error-and-partial-failure.md).

### 10.5 The retry on bad output

**Pattern.** LLM produces a response that's wrong but not invalid; retry policy fires; same wrong output (or different wrong output); waste.

**Corrective.** Don't retry on content-level wrongness per §2.8. Fix at prompt / validation layer.

### 10.6 The retry without backoff

**Pattern.** Retries fire immediately on failure; provider gets hammered; consumer's failure rate stays high.

**Corrective.** Exponential backoff with jitter per §4.

### 10.7 The unbounded retry storm during incidents

**Pattern.** Provider has bad hour; consumer retries thousands of times; cost balloons; nothing protects.

**Corrective.** Bounded budgets per §7; circuit-breaker on retry storm per §7.6.

### 10.8 The "all consumers retry at once" thundering herd

**Pattern.** Without jitter, retries synchronize; provider gets bursty load; storm prolongs incident.

**Corrective.** Jitter per §4.2; decorrelated jitter per §4.3.

### 10.9 The non-idempotent tool retry

**Pattern.** Send-email tool times out; retry; email sent twice; customer confused.

**Corrective.** Idempotency at tool boundary per §3.8; check before resending.

### 10.10 The "retries are free" assumption

**Pattern.** Team treats retries as free (low cost per retry, why not retry more). At scale during incidents, retry costs dominate the bill.

**Corrective.** Track retry cost; cache retries; bound retry budget.

---

## 11. Findings (sprint-assignable)

### REL-RTY-001 — Severity: Critical
**Finding.** Generic retry policy applied to all errors; not per-error-class.
**Recommendation.** Decision tree per §2; per-error-class behavior.
**Owner.** AI platform, sprint N+1.

### REL-RTY-002 — Severity: Critical
**Finding.** Provider idempotency-key not used; retries double-bill.
**Recommendation.** Idempotency-key per §3.3; cross-link to provider documentation.
**Owner.** AI platform, sprint N+1.

### REL-RTY-003 — Severity: Critical
**Finding.** No fleet-wide pause on 429.
**Recommendation.** Redis-backed pause per §5.2.
**Owner.** AI platform + SRE, sprint N+1.

### REL-RTY-004 — Severity: Critical
**Finding.** Agent task retries from step 1 on any failure.
**Recommendation.** Per-step retry per §8.3.
**Owner.** agent platform, sprint N+1.

### REL-RTY-005 — Severity: High
**Finding.** No exponential backoff with jitter.
**Recommendation.** Backoff per §4; jitter to avoid thundering herd.
**Owner.** AI platform, sprint N+2.

### REL-RTY-006 — Severity: High
**Finding.** No per-workload retry config.
**Recommendation.** Config per §8.6; documented per workload.
**Owner.** AI platform + feature teams, sprint N+2.

### REL-RTY-007 — Severity: High
**Finding.** No bounded retry budget per feature.
**Recommendation.** Per-feature budget per §7.2.
**Owner.** AI platform, sprint N+2.

### REL-RTY-008 — Severity: High
**Finding.** Consumer-side response cache absent.
**Recommendation.** Cache per §6.4; saves cost on consumer-side retries.
**Owner.** AI platform, sprint N+2.

### REL-RTY-009 — Severity: High
**Finding.** Retries continue through 429 (Retry-After ignored).
**Recommendation.** Respect Retry-After per §5; pause for advised time.
**Owner.** AI platform, sprint N+2.

### REL-RTY-010 — Severity: Medium
**Finding.** Non-idempotent tools retried naively; double side-effects.
**Recommendation.** Idempotency at tool boundary per §3.8.
**Owner.** tool implementers, sprint N+3.

### REL-RTY-011 — Severity: Medium
**Finding.** Bad-output retry produces same waste.
**Recommendation.** Don't retry content-level wrongness per §2.8.
**Owner.** AI platform, sprint N+3.

### REL-RTY-012 — Severity: Medium
**Finding.** Circuit-breaker on retry storm absent.
**Recommendation.** Pause retries on high storm rate per §7.6; cross-link to [circuit-breakers.md](./circuit-breakers.md).
**Owner.** AI platform, sprint N+3.

### REL-RTY-013 — Severity: Medium
**Finding.** Per-tenant retry budget not implemented.
**Recommendation.** Per-tenant per §7.3.
**Owner.** AI platform, sprint N+3.

### REL-RTY-014 — Severity: Medium
**Finding.** Cached-retry savings not tracked.
**Recommendation.** Metric per §6.6; surface on cost dashboard.
**Owner.** observability-eng, sprint N+4.

### REL-RTY-015 — Severity: Medium
**Finding.** Decorrelated jitter not used; synchronized retry waves.
**Recommendation.** Decorrelated per §4.3.
**Owner.** AI platform, sprint N+4.

### REL-RTY-016 — Severity: Low
**Finding.** Per-call retry-policy override mechanism absent.
**Recommendation.** Override pattern per §8.7.
**Owner.** AI platform, sprint N+5.

### REL-RTY-017 — Severity: Low
**Finding.** First retry uses exponential schedule; usually too slow.
**Recommendation.** Fast first retry per §4.5.
**Owner.** AI platform, sprint N+5.

### REL-RTY-018 — Severity: Low
**Finding.** Long Retry-After (5+ min) pauses fleet; counterproductive for real-time.
**Recommendation.** Fleet pause threshold per §5.3; fall back instead of waiting.
**Owner.** AI platform, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Implement per-error-class decision tree (§2).**
- [ ] **Add provider idempotency-key to every LLM call (§3.3).**
- [ ] **Implement Redis-backed fleet pause on 429 (§5.2).**
- [ ] **Refactor agent retries to per-step (§8.3).**
- [ ] **Implement exponential backoff with jitter (§4).**
- [ ] **Define per-workload retry config (§8.6).**
- [ ] **Implement bounded retry budgets per feature / tenant (§7).**
- [ ] **Implement consumer-side response cache (§6.4).**
- [ ] **Audit non-idempotent tools; add idempotency wrappers (§3.8).**
- [ ] **Add cached-retry savings metric (§6.6).**
- [ ] **Implement circuit-breaker on retry storm (§7.6).**
- [ ] **Decorrelated jitter (§4.3).**
- [ ] **Fast first retry (§4.5).**
- [ ] **Long-Retry-After fallback (§5.3).**

---

## 13. References

**In this folder.**
- [timeout-strategy.md](./timeout-strategy.md) — timeouts that trigger retries; companion.
- [fallback-patterns.md](./fallback-patterns.md) — fallback when retries exhaust.
- [circuit-breakers.md](./circuit-breakers.md) — circuit-breaker that pauses retries; companion.
- [degraded-mode-design.md](./degraded-mode-design.md) *(coming)* — degraded mode after retry exhaustion.

**Elsewhere in this repo.**
- [agent-engineering/error-and-partial-failure.md](../agent-engineering/error-and-partial-failure.md) — agent partial-failure handling; per-step retry.
- [cost-and-finops/cost-aware-rate-limiting.md](../cost-and-finops/cost-aware-rate-limiting.md) — rate-limit awareness composes with retry policy.
- [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md) — retry storms as cost incidents.
- [cost-and-finops/caching-for-cost.md](../cost-and-finops/caching-for-cost.md) — response cache that catches retries.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — fleet-wide pause integrates with backpressure.
- [ai-architecture-reference-architecture / integration-architecture / integration-failure-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/integration-failure-patterns.md) — failure-mode taxonomy.

**External.**
- AWS Architecture Blog: "Exponential Backoff and Jitter" (2015) — foundational reference.
- Google SRE Book — chapter on overload and cascading failures.
- Anthropic / OpenAI provider documentation on idempotency-key and Retry-After.
- httpx, AnthropicSDK retry primitives.
