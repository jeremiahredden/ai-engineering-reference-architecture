# Circuit Breakers

> **Audience.** Engineers whose AI feature is melting down because the provider is having a bad hour and retries are making it worse. SREs whose incident response includes "manually turn off feature X" because there's no circuit-breaker. Anyone whose "we'll add a circuit-breaker later" deferral became "we'll add a circuit-breaker after this incident." **Scope.** The *engineering* practice of circuit-breakers for AI workloads: the classic states (closed / open / half-open); per-provider circuit-breaker (provider outage → fail fast → fall back); per-tool circuit-breaker (tool failing → stop trying); per-feature circuit-breaker (feature broken → degraded mode); integration with cost-budget circuit-breaker; threshold tuning; observability. Not the timeout (see [timeout-strategy.md](./timeout-strategy.md), companion). Not the retry policy (see [retry-strategy.md](./retry-strategy.md), companion). Not the fallback path (see [fallback-patterns.md](./fallback-patterns.md), companion). Not the cost-budget primitive (see [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md), specific). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Circuit-breakers are a well-known reliability pattern: when downstream is failing, stop calling it for a while; reduce load on the failing component; fail fast for the caller. The classic states (closed → open → half-open) are well-established.

For AI workloads, circuit-breakers gain specific applications:

- **Per-provider:** Anthropic / OpenAI is degraded; fail fast and route to fallback model.
- **Per-tool:** A tool the agent depends on is consistently failing; stop trying that tool; agent works around.
- **Per-feature:** The feature is broken in a way that affects user-visible quality; serve degraded mode; page on-call.
- **Cost-aware:** Per-feature or per-tenant cost is exceeded; fail fast on cost rather than on error rate.

The classic circuit-breaker pattern adapts; the AI-specific integrations are what make it useful for an AI platform.

Without circuit-breakers:

- Provider degradation → retries pile up → cost balloons → response times degrade for everyone.
- Tool failures → agent calls fail one by one → each failure costs another LLM call attempt.
- Feature quality regression → users see broken responses → support tickets.
- Cost budget exceeded → spend continues unchecked.

With circuit-breakers:

- Provider degradation → breaker opens → calls fail fast → fallback model takes over → bill stays bounded.
- Tool failures → breaker opens for that tool → agent's prompt updated to skip it → continued operation.
- Feature quality regression → breaker opens → degraded mode serves cached / templated responses → quality drift halted.
- Cost budget exceeded → breaker opens → spend halted.

This document covers the AI-specific circuit-breaker patterns; the threshold tuning that makes them actually useful; the integration with cost-budget circuit-breakers; and the observability that lets on-call understand what's happening.

This document is opinionated about four things:

1. **A circuit-breaker is for stopping bleeding, not for fixing the cause.** When it fires, the bleeding stops; root cause is investigated separately. Don't conflate.
2. **Per-provider circuit-breakers should be the default.** Any AI platform that calls external providers needs them; provider degradations are not rare events.
3. **Cost-as-circuit-breaker is non-optional past trial size.** Without it, the financial blast radius of any incident is unbounded.
4. **Half-open testing must be careful.** A premature half-open during recovery can re-trip the breaker immediately. Patience.

Structure: (2) the classic circuit-breaker states; (3) per-provider circuit-breaker; (4) per-tool circuit-breaker; (5) per-feature circuit-breaker; (6) integration with cost-budget circuit-breaker; (7) threshold tuning; (8) observability and runbook integration; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. The classic circuit-breaker states

The pattern, refreshed for the AI context.

### 2.1 The three states

**Closed (normal).** Calls flow through; failures are tracked; success rate is monitored.

**Open (tripped).** Calls fail fast without attempting the downstream. The downstream is given time to recover.

**Half-open (testing).** A trial call attempts the downstream. If succeeds, breaker returns to closed. If fails, breaker returns to open.

### 2.2 The state transitions

```
Closed:
  - Track failure rate.
  - If failure rate > threshold over window → Open.
  - Otherwise stay Closed.

Open:
  - All calls fail fast.
  - After timeout period → Half-open.

Half-open:
  - Trial call.
  - Success → Closed.
  - Failure → Open (timeout extends; subsequent half-opens delayed).
```

### 2.3 The signal: what counts as "failure"

For provider circuit-breaker:

- HTTP 5xx (server error).
- HTTP 429 (rate limited) — sometimes counted, sometimes not (debatable; rate-limit is intentional throttling, not "failure").
- Timeout.
- Network failure.

For tool circuit-breaker:

- Tool returns error.
- Tool returns invalid response.
- Tool times out.

For feature circuit-breaker:

- Quality regression detected (eval failure).
- User-feedback flag count exceeds threshold.

### 2.4 The window for tracking

Sliding window (last N seconds; e.g., 60 seconds):

```python
class SlidingWindow:
    def __init__(self, window_seconds):
        self.window = window_seconds
        self.events = []  # (timestamp, is_failure) tuples

    def record(self, is_failure):
        now = time.time()
        self.events.append((now, is_failure))
        self._evict_old(now)

    def failure_rate(self):
        now = time.time()
        self._evict_old(now)
        if not self.events:
            return 0.0
        failures = sum(1 for _, f in self.events if f)
        return failures / len(self.events)

    def _evict_old(self, now):
        cutoff = now - self.window
        self.events = [(t, f) for t, f in self.events if t >= cutoff]
```

Each consumer maintains a sliding window; tracks failure rate; trips when threshold exceeded.

### 2.5 The shared state for fleet-wide breakers

Per-process circuit-breakers don't coordinate across consumers. For fleet-wide effect, use shared state (Redis):

```python
def is_open(breaker_name):
    return redis.get(f"breaker:{breaker_name}:state") == "open"

def trip(breaker_name, open_for_seconds):
    redis.set(f"breaker:{breaker_name}:state", "open", ex=open_for_seconds)

def reset(breaker_name):
    redis.delete(f"breaker:{breaker_name}:state")
```

All consumers see the same state; coordinated behavior.

### 2.6 The "fail fast" behavior

When breaker is open:

- Don't attempt the downstream.
- Return error immediately (or trigger fallback per [fallback-patterns.md](./fallback-patterns.md)).
- Don't add to retry queue.

The point is to take load off the downstream, not just to fail in a different way.

### 2.7 The half-open trial

When opening period ends:

- Send one trial call (or limited probes).
- If success → close.
- If fail → re-open; extend timeout.

Limit probes; a flood of half-open trials defeats the purpose of the open period.

---

## 3. Per-provider circuit-breaker

The most common AI-specific application.

### 3.1 The setup

For each provider (Anthropic, OpenAI, Bedrock, etc.) and each model:

```yaml
provider_breakers:
  anthropic:claude-sonnet-4-6:
    failure_threshold_pct: 30  # trip if 30% of calls fail in window
    window_seconds: 60
    open_seconds: 30  # stay open for 30s
    half_open_probe_count: 1

  openai:gpt-4o:
    failure_threshold_pct: 30
    window_seconds: 60
    open_seconds: 30
    half_open_probe_count: 1
```

Per-provider thresholds; tune based on provider's typical reliability.

### 3.2 The behavior on trip

When provider circuit-breaker trips:

- All new calls to that provider/model fail fast.
- Fallback path activates (e.g., switch to alternative provider, smaller model, cached response, structured error).
- Alert fires; on-call notified.

Cross-link to [fallback-patterns.md](./fallback-patterns.md).

### 3.3 The provider 429 question

Is 429 a "failure" that should trip the breaker?

**Argument yes.** 429 means provider can't serve; behavior similar to outage; breaker should open.

**Argument no.** 429 is provider throttling; recovery is on a known schedule (Retry-After); breaker overlapping with rate-limit logic.

**Pragmatic.** Count 429 as failure if Retry-After is short (< 60s); don't count if long. Or: separate breaker for 429 vs 5xx.

### 3.4 The fleet-wide vs per-consumer

For provider breakers, fleet-wide is correct:

- Consumer A sees failures; opens breaker.
- Consumer B should also see breaker open (not discover failures independently).
- Coordinated behavior across the fleet.

Redis-backed; cross-link to §2.5.

### 3.5 The "all providers down" scenario

What if all providers are down (or breakers all open)?

- Fallback to cached responses where possible.
- Surface structured errors to callers.
- Page on-call (this is a major incident).

Cross-link to [fallback-patterns.md](./fallback-patterns.md) §4 (the fallback ladder).

### 3.6 The provider-side maintenance

Providers occasionally announce planned maintenance. The breaker can be pre-tripped:

- Read provider status / planned-maintenance feed.
- Trip the breaker proactively for the maintenance window.
- Reset when maintenance ends.

Cleaner than reactive breaker firing.

---

## 4. Per-tool circuit-breaker

For agent workloads, individual tools may fail.

### 4.1 The setup

For each tool the agent uses:

```yaml
tool_breakers:
  fetch_patient:
    failure_threshold_pct: 25
    window_seconds: 60
    open_seconds: 60

  external_eligibility_check:
    failure_threshold_pct: 30
    window_seconds: 120
    open_seconds: 180  # external services; longer recovery

  search_clinical_notes:
    failure_threshold_pct: 20
    window_seconds: 60
    open_seconds: 60
```

Per-tool; tuned to tool's typical reliability.

### 4.2 The behavior on trip

When tool breaker trips:

- Agent receives "tool unavailable" response when invoking that tool.
- Agent's prompt (next turn) is updated: "Tool X is currently unavailable; work around if possible."
- The agent's reasoning can adapt (e.g., ask user for the information instead of fetching).

### 4.3 The "agent can't proceed without this tool" handling

Some tools are non-substitutable:

- Care Coordinator needs `fetch_patient`; without it, the task can't run.
- If `fetch_patient` breaker trips, the task fails (not the agent's fault).

For these:

- Tool breaker fires.
- Agent fails the task.
- Caller is notified.
- Fallback (next task in queue or human escalation) applies.

### 4.4 The slow tool vs failing tool distinction

A slow tool (each call takes 30s) is different from a failing tool (each call returns error). The breaker:

- Failing tool: failure rate triggers.
- Slow tool: latency-based breaker; opens when P99 exceeds threshold.

Tools can have both kinds of breakers.

### 4.5 The "tool that occasionally fails" tolerance

Some tools are inherently flaky (third-party APIs). Breaker:

- Don't open on 1-2 failures; require sustained failure rate.
- After open, longer recovery period (give the underlying issue time to clear).

Tune thresholds per tool's known reliability.

### 4.6 The tool dependencies

A tool may depend on others:

- `fetch_patient` depends on the database.
- If the database is down, multiple tools fail simultaneously.

The breaker can be at the downstream-resource level (one breaker for "the database") rather than per-tool.

---

## 5. Per-feature circuit-breaker

For AI features whose quality may regress, a feature-level breaker.

### 5.1 The setup

For each AI feature:

```yaml
feature_breakers:
  care-coordinator:
    quality_threshold: 0.85  # eval pass rate
    quality_window: 1_hour
    failure_threshold_pct: 5  # error rate
    failure_window_seconds: 60
    open_seconds: 300  # 5 min

  patient-api-chat:
    quality_threshold: 0.90
    quality_window: 30_min
    failure_threshold_pct: 3
    failure_window_seconds: 60
    open_seconds: 120
```

Per-feature; quality + reliability signals combined.

### 5.2 The quality signal

Quality is harder to define than reliability. Sources:

- **Live eval.** A judge model evaluates sampled responses.
- **User feedback.** Thumbs-down rate exceeds threshold.
- **Schema validation failure rate.** Structured-output failures.
- **Output sanity check.** Hallucinated entity rate (cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / per-tenant-prompt-and-context.md §6](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-prompt-and-context.md)).

When quality signal drops past threshold, the breaker fires.

### 5.3 The behavior on trip

When feature breaker trips:

- The feature serves degraded mode (cached responses, simpler model, structured "feature degraded" message).
- Alert fires; engineering team paged.
- New deploys to the feature are blocked until investigation.

Cross-link to [degraded-mode-design.md](./degraded-mode-design.md) *(coming)*.

### 5.4 The "quality regression detected by judge" signal

A live eval judge samples responses; assigns pass/fail:

```python
def quality_judge_loop(feature):
    while True:
        sampled_responses = get_recent_responses(feature, sample_rate=0.05)
        for resp in sampled_responses:
            score = judge_model.evaluate(resp)
            quality_window.record(score)
        
        if quality_window.pass_rate() < threshold:
            trip_breaker(f"feature:{feature}:quality")
        
        sleep(60)
```

Continuous monitoring; trips when sustained quality drops.

### 5.5 The "user feedback signal" caveat

User feedback (thumbs-down) is noisy and slow:

- Most users don't give feedback.
- Negative feedback often delayed.
- Signal-to-noise ratio is low.

Use as supplementary signal, not primary. Live eval (§5.4) is more reliable.

### 5.6 The "deploy-correlated regression" pattern

If quality drops after a deploy:

- Breaker may fire shortly after the deploy.
- Automated rollback or alert.
- Investigation focuses on the deploy.

Cross-link to [cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md) §9.

### 5.7 The "false trip" risk

Quality signals can produce false trips:

- Sample size too small.
- Judge model's variance.
- Edge-case responses skewing the rate.

Tune thresholds carefully; require sustained signal (not single-sample spikes).

---

## 6. Integration with cost-budget circuit-breaker

The cost-budget circuit-breaker is documented separately in [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md). This section covers its composition with reliability circuit-breakers.

### 6.1 The composition

A single LLM call may pass through multiple circuit-breakers:

```
Pre-call checks:
  - Per-feature reliability breaker (open?)
  - Per-tenant cost-budget breaker (exceeded?)
  - Per-feature cost-budget breaker (exceeded?)
  - Provider reliability breaker (open?)
  - Provider rate-limit breaker (paused?)
  
If any breaker is open → fail fast with structured error.
```

Multiple breakers; multiple reasons for failure; structured error indicates which.

### 6.2 The "cost breaker fires first" pattern

Often the cost breaker fires before the reliability breaker:

- Provider has trouble; latency rises but isn't outright error.
- Cost-per-call rises (longer calls, retries).
- Cost-budget breaker fires.
- Reliability breaker hasn't fired yet (errors haven't crossed threshold).

This is the right order: cost protection should be eager; reliability protection slower.

### 6.3 The combined error envelope

When multiple breakers are open:

```json
{
  "status": "service_unavailable",
  "circuit_breakers_open": [
    "provider:anthropic:sonnet (since 14:23 for 60s)",
    "feature:care-coordinator:cost (daily budget exhausted)"
  ],
  "retry_after_ms": 60_000,
  "alternative_actions": [
    "Wait for provider breaker to close",
    "Use fallback feature (patient-api-chat)"
  ]
}
```

Caller sees the breakdown.

### 6.4 The "everything is open" cascade

If many breakers are open simultaneously, fallback paths may also be blocked:

- Care Coordinator breaker open.
- Fallback to Haiku model: Haiku provider breaker also open.
- Fallback to cached response: cache miss.

The architecture needs a final fallback (structured error to caller; manual escalation).

### 6.5 The shared signal pattern

Some breakers feed others:

- Provider breaker opens → many features impacted → some features' reliability breakers will also open if they depend on that provider.
- Cost-budget breaker for feature X → impacts cost for feature Y (if they share infrastructure).

Document the dependencies; expect cascading failures.

---

## 7. Threshold tuning

The hard part. Thresholds determine the breaker's effectiveness.

### 7.1 The "too sensitive" failure mode

Breaker trips on small fluctuations:

- Normal variance is interpreted as failure.
- Breaker opens; legitimate traffic is blocked.
- Breaker oscillates between open and closed.

Symptom: high breaker open rate; many false trips.

### 7.2 The "too lax" failure mode

Breaker doesn't trip until things are severely broken:

- Failures accumulate before breaker fires.
- Heavy load on the failing downstream.
- Cost / time wasted on doomed retries.

Symptom: breaker rarely fires; downstream gets hammered during real incidents.

### 7.3 The tuning approach

Start conservative; tune toward sensitivity:

- Initial threshold: 50% failure rate over 60-second window.
- Observe production data.
- If breaker never fires during real incidents: lower threshold (more sensitive).
- If breaker fires on noise: raise threshold (less sensitive).

### 7.4 The "noise floor" baseline

What's the normal failure rate?

- Most production AI workloads see 0.1-1% baseline failure (transient network issues).
- Spikes during provider issues: 5-30%.

Threshold should be above baseline + some margin (e.g., 5-10% for sensitive; 20-30% for conservative).

### 7.5 The per-provider tuning

Different providers have different reliability:

- Mature hosted providers (Anthropic, OpenAI direct): 0.1% baseline; threshold 5-10%.
- Smaller white-label providers: 0.5% baseline; threshold 10-15%.
- Self-hosted: variable; threshold based on observed reliability.

### 7.6 The window-size tuning

Short window (10 seconds): reacts fast to spikes; more sensitive to noise.

Long window (5 minutes): less sensitive; slower to react.

For provider breakers, 60-second window is typical.

### 7.7 The minimum-sample-size requirement

Don't trip on small samples:

- If only 5 calls in the window and 2 fail (40% failure), don't trip.
- Require minimum sample size (e.g., 20 calls) before threshold evaluation.

Prevents false positives from low-traffic periods.

### 7.8 The "open period" duration

How long should the breaker stay open?

- Short (30-60s): assumes downstream recovers quickly.
- Long (5-15 min): for known-slow recoveries (provider maintenance).

Default 60s; tune based on observed recovery patterns.

### 7.9 The half-open probe rate

How often does half-open test?

- One probe at a time.
- After failure: re-open; extend timeout.
- Don't flood with probes; defeats the purpose.

### 7.10 The threshold review cycle

Quarterly review:

- Per-breaker fire rate.
- False-positive rate (estimated).
- False-negative rate (real incidents that should have tripped earlier).
- Adjust thresholds.

---

## 8. Observability and runbook integration

A breaker that fires without anyone knowing is just a silent failure.

### 8.1 The metrics

Per-breaker:

- State (closed / open / half-open).
- Time since last state change.
- Trip count (last hour, last day, last week).
- Failure rate during closed periods.
- Mean time to recovery (MTTR) — how long does it stay open.

### 8.2 The dashboard

A "circuit-breaker status" dashboard:

- All breakers; current state; recent activity.
- Drill-down: per-breaker history; trip events.

When on-call gets paged for a feature, the dashboard shows whether breakers are open.

### 8.3 The alert design

When a breaker trips:

- Notification to relevant team.
- Page if customer-facing impact.
- Slack message with breaker context (which breaker, how long, why).

Cross-link to [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md).

### 8.4 The runbook per breaker

Each breaker links to a runbook:

- What this breaker means.
- What downstream is affected.
- How long it typically stays open.
- What to investigate.
- How to manually reset (if appropriate).

### 8.5 The "manually reset the breaker" capability

For edge cases, operators can reset:

- After verifying the downstream is healthy.
- Force-close the breaker.
- Resume traffic.

This is rare; usually the breaker should close on its own. But the capability is needed for unusual cases (e.g., breaker tripped on a transient anomaly; downstream is fine; want to resume immediately).

### 8.6 The "breaker fired but on-call doesn't know" failure

Alert routing:

- Each breaker has an owner team.
- Alert routes to that team.
- Escalation if not acknowledged.

Cross-link to alerting patterns.

### 8.7 The post-incident review

When a breaker fires during a real incident:

- Did it fire fast enough?
- Did it open at the right threshold?
- Did the fallback work?
- Should the threshold be tuned?

Post-incident review of the breaker behavior; tune.

---

## 9. Worked Meridian example

Meridian's circuit-breaker stack handled multiple provider incidents in 2025-2026.

### 9.1 The breaker catalog

```
Provider breakers:
  anthropic:claude-sonnet-4-6 (US): threshold 25% / 60s window / 60s open
  anthropic:claude-haiku-4-5 (US): threshold 25% / 60s window / 60s open
  anthropic:claude-opus-4-7 (US): threshold 25% / 60s window / 90s open (premium)
  cohere:command-r-plus (CA): threshold 30% / 60s window / 60s open
  
Tool breakers (Care Coordinator):
  fetch_patient: threshold 20% / 60s / 60s
  search_clinical_notes: threshold 20% / 60s / 60s
  external_eligibility_check: threshold 30% / 120s / 180s
  draft_referral (LLM-internal): managed via provider breaker
  
Feature breakers:
  care-coordinator: quality < 0.85 over 1h, OR errors > 5% over 60s, open 5 min
  patient-api-chat: quality < 0.90 over 30 min, OR errors > 3% over 60s, open 2 min
  
Cost breakers (cross-link to cost-and-finops):
  per-feature: care-coordinator $1500/day
  per-tenant: tier-specific
```

### 9.2 The Q1 2026 Anthropic incident

Anthropic had a 12-minute degradation; elevated errors and 429s.

Timeline:
- 14:23 — Provider error rate climbs from 0.1% to 8% over 2 minutes.
- 14:25 — Sonnet provider breaker fires (threshold 25%, exceeded at 28%).
- 14:25 — Patient API chat falls back to Haiku via fallback ladder.
- 14:25 — Care Coordinator pauses in-flight tasks (per workflow); new tasks queue.
- 14:25 — Document classification queues (won't urgent-process).
- 14:35 — Provider error rate back to 0.5% (recovering).
- 14:36 — Half-open probe succeeds.
- 14:36 — Breaker closes; normal traffic resumes.

Outcome:
- Patient API chat: continued operating on Haiku for ~13 min; UX impact minimal.
- Care Coordinator: 8 in-flight tasks paused; resumed; minor latency to clinicians.
- Document classification: queued ~3k documents; processed after.
- Total cost during incident: ~$80 extra (mostly from Haiku fallback).
- Without breakers: estimated $2-3k extra cost from retry storm + provider load.

### 9.3 The Q2 2026 external eligibility check incident

An external eligibility-check API (provided by an insurance partner) had a multi-hour outage.

Timeline:
- 09:15 — Eligibility checks start failing (network issues at partner).
- 09:16 — Tool breaker fires (threshold 30%, exceeded at 35%).
- 09:16 — Care Coordinator's agent receives "eligibility tool unavailable" responses.
- 09:16 — Agent's prompt adapts: "Eligibility check is unavailable; proceed without; mark eligibility as 'TBD' for human review."
- 09:16 — Care Coordinator continues operating in degraded mode.
- 11:30 — Partner restored; eligibility tool starts succeeding.
- 11:31 — Half-open probe succeeds.
- 11:31 — Tool breaker closes.

Outcome:
- Care Coordinator continued operating; ~140 tasks completed during the 2.25-hour incident with "eligibility TBD" markers.
- After restoration, the marked-TBD tasks went through eligibility review (manual).
- Customer-facing impact: minimal; clinicians used the agent normally; eligibility was a downstream concern.
- Without breaker: each eligibility tool call would have timed out; agent task latency would have ballooned; significant disruption.

### 9.4 The Q4 2025 feature quality regression

A Care Coordinator prompt change inadvertently produced a quality regression.

Timeline:
- Day 0 — Deploy ships at 14:00.
- Day 0 14:00 → 16:00 — Care Coordinator's live-judge quality score drops from 0.92 to 0.78.
- Day 0 16:05 — Feature quality breaker fires (threshold 0.85, sustained below for > 30 min).
- Day 0 16:05 — Care Coordinator serves cached / templated responses (degraded mode).
- Day 0 16:05 — Engineering team paged; deploy log reviewed.
- Day 0 16:20 — Deploy identified as cause; revert PR opened.
- Day 0 16:30 — Revert deployed.
- Day 0 17:30 — Quality score back to 0.91 (verified by judge over 1 hour).
- Day 0 17:30 — Manual reset of breaker (after confirming healthy).

Outcome:
- ~25 minutes from deploy to detection.
- ~30 minutes from detection to revert.
- Degraded mode during incident: cached / templated responses; clinicians notified.
- Without breaker: regression would have persisted until next-day eval review; many more affected tasks.

### 9.5 The threshold-tuning history

Initial thresholds (2024) → tuned over time:

- Provider breaker: started at 50% / 60s. Tuned down to 25% / 60s after observing real incidents at lower failure rates.
- Tool breaker: started at 30% / 60s. Specific tools tuned individually.
- Feature quality breaker: started at threshold 0.80. Tuned up to 0.85 after observing that 0.80 produced false trips on edge-case responses.

Each tuning was evidence-based; tracked.

### 9.6 The dashboard

A circuit-breaker dashboard shows all breakers' current state. Engineering checks during incidents; FinOps checks during cost reviews; SRE checks during normal monitoring.

### 9.7 The runbook integration

Each breaker has a runbook entry:

- "When provider:anthropic:sonnet fires" → runbook covers fallback verification, expected duration, when to escalate.
- "When feature:care-coordinator:quality fires" → runbook covers deploy review, judge-sample inspection, revert procedure.

### 9.8 What the breaker stack costs

- ~3 weeks initial build (1 engineer).
- Ongoing: ~5% of platform team's time for tuning + runbook maintenance.
- Infrastructure: shared Redis with rate-limit infrastructure.

### 9.9 What the breaker stack prevents

- Provider degradation cost: ~$2-3k per major incident saved.
- Tool failure cascades: ~1-2 incidents per quarter contained.
- Quality regressions: typically caught and reverted within an hour vs day-long undetected.
- Composite estimated value: $30-50k/year in avoided incident cost + reduced engineering response burden.

### 9.10 The post-incident review

After each significant breaker firing:

- Did the breaker fire at the right time?
- Was the threshold appropriate?
- Did fallback work?
- Documentation update.

Continuous improvement; thresholds refined over time.

---

## 10. Anti-patterns

### 10.1 The missing circuit-breaker

**Pattern.** No circuit-breaker at all. Provider degradation → retries pile up → cost balloons → manual intervention required.

**Corrective.** Provider breaker per §3 as a starting point; expand from there.

### 10.2 The per-process breaker

**Pattern.** Each consumer has its own breaker; doesn't share state with other consumers. Each one discovers failures independently.

**Corrective.** Fleet-wide via shared state per §2.5.

### 10.3 The "too sensitive" breaker

**Pattern.** Threshold so low that normal variance trips it. Breaker opens frequently; legitimate traffic blocked.

**Corrective.** Tune to above noise floor per §7.4. Quarterly review.

### 10.4 The "too lax" breaker

**Pattern.** Threshold so high that real failures don't trip it. Provider hammered; cost balloons.

**Corrective.** Lower threshold; observe production data; tune to actual incident rates.

### 10.5 The breaker without observability

**Pattern.** Breaker fires; nobody knows. Investigation includes "wait, is the breaker open?" check.

**Corrective.** Dashboard + alerts + runbook per §8.

### 10.6 The "flood of half-open probes"

**Pattern.** Half-open allows many probes simultaneously. Probe storm re-trips the breaker; oscillation.

**Corrective.** One probe at a time per §2.7.

### 10.7 The breaker without fallback

**Pattern.** Breaker opens; fail-fast happens; no fallback. Caller gets generic error; UX suffers.

**Corrective.** Fallback path defined per §3.2; cross-link to [fallback-patterns.md](./fallback-patterns.md).

### 10.8 The "cost breaker missing" gap

**Pattern.** Reliability breakers exist; cost breaker doesn't. Reliability incident is bounded; cost incident is not.

**Corrective.** Cost-budget breaker per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).

### 10.9 The breaker that needs manual reset every time

**Pattern.** Breaker opens; doesn't close automatically; requires on-call to manually reset.

**Corrective.** Half-open testing per §2.7; auto-close when downstream recovers.

### 10.10 The "we'll add the breaker later" deferral

**Pattern.** No breaker at launch; "we'll add when we need it." First major incident is the trigger; rushed implementation under pressure.

**Corrective.** Provider breaker at minimum from day one. Expand as needed.

---

## 11. Findings (sprint-assignable)

### REL-CB-001 — Severity: Critical
**Finding.** No provider circuit-breaker; provider degradations produce retry storms.
**Recommendation.** Per-provider breaker per §3 with fleet-wide state per §2.5.
**Owner.** AI platform + SRE, sprint N+1.

### REL-CB-002 — Severity: Critical
**Finding.** No per-tool circuit-breaker for agent workloads.
**Recommendation.** Per-tool breaker per §4.
**Owner.** agent platform, sprint N+1.

### REL-CB-003 — Severity: Critical
**Finding.** No circuit-breaker observability.
**Recommendation.** Dashboard + alerts + runbook per §8.
**Owner.** observability-eng + SRE, sprint N+1.

### REL-CB-004 — Severity: High
**Finding.** No per-feature quality circuit-breaker.
**Recommendation.** Per-feature breaker per §5 with live-judge signal.
**Owner.** AI platform, sprint N+2.

### REL-CB-005 — Severity: High
**Finding.** Cost-budget circuit-breaker missing.
**Recommendation.** Cross-link to [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md).
**Owner.** AI platform + FinOps, sprint N+2.

### REL-CB-006 — Severity: High
**Finding.** Thresholds set arbitrarily; not based on production data.
**Recommendation.** Tune per §7; quarterly review.
**Owner.** SRE, sprint N+2.

### REL-CB-007 — Severity: High
**Finding.** Half-open testing floods with probes.
**Recommendation.** Single probe per §2.7.
**Owner.** AI platform, sprint N+2.

### REL-CB-008 — Severity: High
**Finding.** Breaker opens but no fallback defined.
**Recommendation.** Fallback per §3.2; cross-link to [fallback-patterns.md](./fallback-patterns.md).
**Owner.** AI platform, sprint N+2.

### REL-CB-009 — Severity: Medium
**Finding.** Per-provider breakers not differentiated by model.
**Recommendation.** Per-provider × per-model per §3.1.
**Owner.** AI platform, sprint N+3.

### REL-CB-010 — Severity: Medium
**Finding.** 429 handling vs reliability handling not separated.
**Recommendation.** Distinct breaker behavior per §3.3.
**Owner.** AI platform, sprint N+3.

### REL-CB-011 — Severity: Medium
**Finding.** Live-judge quality signal not implemented.
**Recommendation.** Sampled judge per §5.4.
**Owner.** AI platform + eval, sprint N+3.

### REL-CB-012 — Severity: Medium
**Finding.** Minimum sample size not enforced; false trips on low traffic.
**Recommendation.** Sample-size requirement per §7.7.
**Owner.** AI platform, sprint N+3.

### REL-CB-013 — Severity: Medium
**Finding.** Per-breaker runbook absent.
**Recommendation.** Runbook per breaker per §8.4.
**Owner.** SRE, sprint N+3.

### REL-CB-014 — Severity: Medium
**Finding.** Threshold tuning not on quarterly cadence.
**Recommendation.** Quarterly review per §7.10.
**Owner.** SRE, sprint N+4.

### REL-CB-015 — Severity: Medium
**Finding.** Manual reset capability absent.
**Recommendation.** Operator-reset per §8.5; audited.
**Owner.** SRE, sprint N+4.

### REL-CB-016 — Severity: Low
**Finding.** Cascading breaker dependencies not documented.
**Recommendation.** Document per §6.5; understand the cascade.
**Owner.** AI platform, sprint N+5.

### REL-CB-017 — Severity: Low
**Finding.** Provider-side maintenance not pre-tripping breaker.
**Recommendation.** Proactive breaker per §3.6 if provider publishes maintenance schedule.
**Owner.** AI platform, sprint N+5.

### REL-CB-018 — Severity: Low
**Finding.** Post-incident review of breaker behavior not standard.
**Recommendation.** Review per §8.7; tune thresholds.
**Owner.** SRE, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Implement provider circuit-breaker (§3).** Fleet-wide via Redis.
- [ ] **Implement per-tool circuit-breaker (§4).** Per-tool thresholds.
- [ ] **Implement per-feature circuit-breaker (§5).** Reliability + quality signals.
- [ ] **Implement live-judge quality signal (§5.4).**
- [ ] **Connect to cost-budget circuit-breaker (§6).**
- [ ] **Build circuit-breaker dashboard (§8.2).**
- [ ] **Define runbook per breaker (§8.4).**
- [ ] **Define fallback per breaker (§3.2).**
- [ ] **Configure half-open probe rate (§2.7).**
- [ ] **Enforce minimum sample size (§7.7).**
- [ ] **Quarterly threshold tuning (§7.10).**
- [ ] **Pre-production chaos test:** simulate provider degradation; verify breaker fires; verify fallback works; verify recovery.
- [ ] **Post-incident review** of breaker behavior; tune.

---

## 13. References

**In this folder.**
- [timeout-strategy.md](./timeout-strategy.md) — timeouts that contribute to breaker signal; companion.
- [retry-strategy.md](./retry-strategy.md) — retry policy that interacts with breaker (don't retry through open breaker); companion.
- [fallback-patterns.md](./fallback-patterns.md) — fallback paths when breaker fires.
- [degraded-mode-design.md](./degraded-mode-design.md) *(coming)* — degraded mode is the typical "breaker open" behavior.

**Elsewhere in this repo.**
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — cost-as-circuit-breaker primitive.
- [cost-and-finops/cost-incident-runbook.md](../cost-and-finops/cost-incident-runbook.md) — incidents where breakers fire.
- [cost-and-finops/cost-dashboards-and-alerts.md](../cost-and-finops/cost-dashboards-and-alerts.md) — alert integration.
- [observability-and-telemetry/quality-drift-detection.md](../observability-and-telemetry/quality-drift-detection.md) — quality signal that feeds feature breaker.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — fleet-wide rate-limit pause overlaps with provider breaker.
- [ai-architecture-reference-architecture / integration-architecture / integration-failure-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/integration-failure-patterns.md) — failure-mode taxonomy.

**External.**
- Hystrix documentation (legacy; concepts apply).
- Resilience4j documentation (circuit-breaker primitives in Java).
- Polly documentation (.NET).
- "Release It!" by Michael Nygard — circuit-breaker pattern's canonical reference.
- Google SRE Book — chapter on overload and cascading failures.
