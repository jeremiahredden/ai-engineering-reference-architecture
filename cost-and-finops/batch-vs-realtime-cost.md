# Batch vs Realtime Cost

> **Audience.** Engineers whose AI workload tolerates hours of latency but is paying real-time API rates. Tech leads whose nightly classification job costs $8k/month and could be $4k/month. Anyone whose data team is running an analytics enrichment workflow against the standard LLM endpoint when they could be using the batch endpoint. **Scope.** The *engineering* practice of using batch APIs to halve cost on batch-eligible workloads: the batch API offerings (2026); workload classification (what's batch-eligible); integration patterns (async, queue, callback); the 24-hour SLA reality; batch eval and quality discipline; cost accounting for batch. Not the architectural decision of sync-vs-async-vs-streaming (see [ai-architecture-reference-architecture / integration-architecture / sync-vs-async-vs-streaming.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/sync-vs-async-vs-streaming.md)). Not the general queue / callback patterns (see [callback-and-webhook-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/callback-and-webhook-patterns.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

In 2026, major hosted providers offer "batch" APIs at roughly 50% of the cost of their real-time APIs. The deal:

- You submit a batch of requests (up to thousands per submission).
- The provider processes them within a 24-hour window (often much faster, but no real-time guarantee).
- You pay ~50% of the per-token rate.
- No rate-limit pressure during the batch (within the batch quota).

For workloads where the user isn't waiting — overnight ingestion, analytics enrichment, bulk classification, training-data generation, backfill operations — batch APIs are a near-free 50% cost reduction.

The reality: most teams don't use them. Common reasons:

- Unaware. Batch APIs weren't on the team's radar when they shipped.
- Inertia. Existing code calls the real-time API; nobody wants to refactor.
- Misjudging eligibility. The team thinks "we need it within an hour" but actually 4-12 hours is fine.
- Integration friction. The team's pipeline expects synchronous response; batch APIs are async.
- Quality concerns. The team worries batch output is somehow worse (it isn't — same model, same quality).

Each of these is a solvable engineering problem. The savings justify the engineering investment for any workload above modest volume.

This document covers the engineering: which APIs are available, which workloads qualify, how to integrate them, what the SLAs actually mean, how to track cost on the batch path, and the common anti-patterns.

This document is opinionated about four things:

1. **Batch eligibility is broader than teams think.** The mental model "real-time is the default; batch is exceptional" produces under-use. The right framing: "any workload where no human is waiting is batch-eligible; default to batch."
2. **Batch APIs are the same model as real-time.** Output quality is identical; the only difference is latency. Concerns about "batch quality" are usually unfounded.
3. **The 24-hour SLA is the worst-case, not the typical-case.** Most batch jobs complete in minutes-to-hours. The 24-hour bound is the cap; design for it but expect faster.
4. **Integration friction is one-time.** Adding async / queue / callback paths is engineering work that pays off forever. The first batch integration is the slowest; subsequent uses are reuse.

Structure: (2) the batch API offerings in 2026; (3) workload classification; (4) the integration patterns; (5) the 24-hour SLA reality; (6) batch eval and quality assurance; (7) cost accounting for batch; (8) worked Meridian example; (9) anti-patterns; (10) findings; (11) adoption sequencing; (12) references. (One section less than typical — batch is straightforward.)

---

## 2. The batch API offerings (2026)

The current landscape. Subject to change; verify with each provider before committing.

### 2.1 Anthropic Batch API

**Pricing.** ~50% of standard input + output rates.

**Mechanics.** POST a JSONL file of requests (up to ~10k requests; check current limits). Anthropic processes asynchronously. Status polled or webhook-triggered. Results retrieved as JSONL.

**SLA.** 24-hour completion target; typical completion much faster (minutes to hours for small batches).

**Models.** All Claude models available via Batch.

**Rate limits.** Separate batch-specific quota; doesn't compete with real-time quota.

### 2.2 OpenAI Batch API

**Pricing.** 50% of standard rates.

**Mechanics.** Upload a JSONL file; create a batch job; poll for status; retrieve results.

**SLA.** 24-hour completion target.

**Models.** All major OpenAI models.

**Rate limits.** Separate quota; doesn't compete with real-time.

### 2.3 Google Vertex AI batch prediction

**Pricing.** Lower than online prediction (~25-50% depending on model).

**Mechanics.** Submit batch via UI or API; results to BigQuery or Cloud Storage.

**SLA.** Variable; typically completes in minutes-to-hours.

**Models.** Gemini models via batch prediction.

### 2.4 AWS Bedrock batch inference

**Pricing.** 50% of on-demand pricing for select models.

**Mechanics.** Upload JSONL to S3; submit job via Bedrock console / API; results to S3.

**SLA.** Completion time varies by job size and provider; typically hours.

**Models.** Subset of Bedrock models (check current support).

### 2.5 The "no batch API" cases

Not all providers / models offer batch. Where batch isn't available:

- Self-hosted models: the "batch" benefit applies to provider economics; for self-hosted, batching is purely an internal optimization (concurrent inference, GPU utilization).
- Smaller providers (white-label inference): some have batch options, some don't.
- Embedding-only APIs: some offer batch, some don't.

For workloads on self-hosted, the batching pattern is different — see §6.

### 2.6 The summary

| Provider | Discount | SLA | Workload fit |
| --- | --- | --- | --- |
| Anthropic Batch | ~50% | 24h | General Claude workloads |
| OpenAI Batch | 50% | 24h | General GPT workloads |
| GCP Vertex batch | 25-50% | variable | Gemini workloads |
| AWS Bedrock batch | 50% | variable | Select Bedrock models |
| Self-hosted | N/A | N/A | Internal batching |

---

## 3. Workload classification

What's batch-eligible. The framing.

### 3.1 The eligibility test

A workload is batch-eligible if:

1. No user is waiting in real-time for the result.
2. The result is acceptable when delivered within 24 hours.
3. The workload doesn't depend on context that will be stale by then (current price, current inventory).

If yes to all three: batch-eligible. Default to batch.

### 3.2 The typical batch workloads

**Overnight document ingestion.** Documents arriving during the day; processing accumulates; runs overnight; results available next morning.

**Bulk classification backfill.** Re-classifying historical data with a new model or new taxonomy. No urgency.

**Embedding generation for new corpus.** Embedding a million documents on first ingestion. Hours-to-days latency is fine; cost matters.

**Analytics enrichment.** Adding AI-derived fields (sentiment, summary, classification) to records flowing into the warehouse. Real-time isn't needed; warehouse latency is hours anyway.

**Training data generation.** Producing labeled data for fine-tuning. Latency is irrelevant; cost matters.

**Email / digest summarization.** Daily summary of overnight activity. Generated before morning; latency budget is hours.

**Compliance scanning.** Periodic re-scan of records for compliance signals. Latency tolerance is hours-to-days.

### 3.3 The "kind-of batch" workloads

Workloads that are partially batch:

**Customer-uploaded document analysis.** User uploads document; result returned in the UI. Real-time? Often the user can wait minutes; if so, batch-eligible with a "we'll email you when done" UX.

**Background suggestion generation.** User does something; system later suggests next actions. Real-time isn't needed; results can be ready by next session.

**Search index update.** New documents indexed for search; embedding-and-write is batch-friendly.

For these, the integration is: "submit batch on event; deliver result async (notification, dashboard refresh, etc.)."

### 3.4 The "not batch" workloads

**User-facing chat.** User is waiting; real-time required.

**Clinical decision support.** Clinician is in the room with the patient; real-time required.

**Authentication / authorization decisions.** Sub-second response required.

**Real-time interaction.** Live conversation, live voice; obvious.

**Streaming responses.** Anything where output is rendered as it generates.

### 3.5 The "I thought it was real-time but maybe not" check

Some workloads claimed as real-time aren't actually:

- "Email summary" — typically generated daily; batch.
- "Daily report" — generated overnight; batch.
- "Recommended next steps" — surfaced asynchronously; usually batch-eligible.
- "Background enrichment" — by definition async; usually batch.

For these, audit the claimed latency requirement; the actual user-facing latency may be hours.

### 3.6 The classification audit

For an established AI platform, classify each workload:

| Workload | Calls/day | Cost/month | Real-time required? | Batch-eligible? |
| --- | --- | --- | --- | --- |
| Care Coordinator | 5k | $30k | Yes | No |
| Patient API chat | 12k | $8k | Yes | No |
| Document ingestion | 40k | $25k | No | Yes |
| Embedding | 40k | $4k | No | Yes |
| Analytics enrichment | 80k | $15k | No | Yes |
| Compliance backfill | 200k weekly | $12k | No | Yes |
| Email summaries | 3k/day | $2k | No (eod) | Yes |

Then compute the savings potential: 50% of "Yes" workloads.

### 3.7 The "soft launch into batch" strategy

For an existing real-time workload that's batch-eligible:

1. Pilot: route 10% of traffic to batch path.
2. Validate quality (same model, but verify integration is correct).
3. Validate user-facing latency (does the async UX hold up?).
4. Ramp to 100%.

Avoid all-at-once flips; the integration patterns may have edge cases.

---

## 4. The integration patterns

How batch fits into the architecture.

### 4.1 The submit-poll pattern

Simplest integration:

```python
def process_documents_batch(documents: list[Document]):
    # Build batch file
    batch_file = build_jsonl([
        {
            "custom_id": doc.id,
            "params": {
                "model": "claude-sonnet-4-6",
                "messages": [{"role": "user", "content": doc.text}],
                "max_tokens": 1000
            }
        }
        for doc in documents
    ])

    # Submit
    batch_response = anthropic.batches.create(file=batch_file)
    batch_id = batch_response.id

    # Poll until complete
    while True:
        status = anthropic.batches.retrieve(batch_id)
        if status.processing_status == "ended":
            break
        time.sleep(60)

    # Retrieve results
    results = anthropic.batches.results(batch_id)
    for result in results:
        process_result(result)
```

Suitable for jobs you can wait on. Polling at 60s intervals; total elapsed is minutes-to-hours.

### 4.2 The fire-and-forget pattern with callback

For jobs where the submitting process doesn't wait:

```python
def submit_batch_with_callback(documents: list[Document], callback_url: str):
    batch_response = anthropic.batches.create(
        file=build_jsonl(...),
        webhook=callback_url  # if supported by provider
    )
    save_batch_state(batch_response.id, callback_url, documents)

def on_batch_webhook(batch_id):
    state = retrieve_batch_state(batch_id)
    results = anthropic.batches.results(batch_id)
    for result in results:
        process_result(result)
```

Submitter returns immediately; webhook fires on completion; downstream pipeline picks up. Suitable for long-running or scheduled batches.

### 4.3 The durable workflow pattern

For batches that are part of a larger workflow:

```python
@workflow.defn
class BatchEnrichmentWorkflow:
    @workflow.run
    async def run(self, documents: list[Document]):
        # Step 1: submit batch
        batch_id = await workflow.execute_activity(
            submit_batch_activity, documents,
            start_to_close_timeout=timedelta(minutes=5)
        )

        # Step 2: wait for completion (workflow framework handles waiting)
        await workflow.wait_for_external_event(
            "batch_completed", timeout=timedelta(hours=24)
        )

        # Step 3: process results
        results = await workflow.execute_activity(
            retrieve_results_activity, batch_id
        )

        # Step 4: downstream processing
        await workflow.execute_activity(
            store_enriched_documents, results
        )
```

Cross-link to [callback-and-webhook-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/callback-and-webhook-patterns.md) and [agent-loop-design.md](../agent-engineering/agent-loop-design.md) for durable-workflow patterns.

### 4.4 The "queue ingest, drain to batch" pattern

For continuous batch-eligible workloads:

```
Documents arrive → SQS / Kafka queue
                       ↓
        Batch builder (every 30 min or every 1k items, whichever first)
                       ↓
                  Anthropic Batch API
                       ↓
                  Results processor
                       ↓
                  Vector store + downstream
```

The queue accumulates items; the batch builder periodically drains and submits. Cost-efficient (large batches) and latency-bounded (worst-case wait is the build interval).

### 4.5 The hybrid real-time + batch pattern

Some workloads can serve cached responses real-time, then refresh asynchronously via batch:

```
User queries → Real-time path
                 ↓
        Cache hit? Return cached.
        Cache miss? Synchronous LLM call; cache; return.

                 ↓ (separately)

Batch refresh path:
  Periodic: re-process cached entries to refresh; batch API at 50%.
```

The batch refresh keeps cache content current while the real-time path stays fast.

### 4.6 The result-handling at scale

Batch results can be large (thousands of responses). Result handling:

- Stream results as they're available (don't load all into memory).
- Write to durable storage (S3, database) before processing.
- Idempotent processing (re-running on the same results doesn't duplicate effects).
- Error handling per result (some may have failed individually).

### 4.7 The per-request idempotency

Each request in a batch has a `custom_id`. Use it as the idempotency key:

- Set custom_id to document ID, request UUID, or workflow step ID.
- Result is matched to the custom_id.
- Downstream processing dedups on custom_id.

If the same batch is submitted twice (network retry, workflow retry), the custom_id allows dedup at the result-handling layer.

---

## 5. The 24-hour SLA reality

What the SLA actually means and how to design for it.

### 5.1 The provider's stated SLA

Major providers state: "batch jobs complete within 24 hours." The fine print:

- 24 hours is the worst case, not the median.
- Provider can extend during capacity pressure (rare).
- Failure-to-complete means refund + escalation; usually means provider's outage.

For typical workloads, batch completes in:

- Small batches (< 100 requests): minutes.
- Medium batches (100-1000 requests): minutes-to-hours.
- Large batches (1000-10000 requests): hours.
- Very large (during provider peak): hours-to-most-of-24-hours.

### 5.2 The design implications

Workloads must tolerate 24-hour latency. Specifically:

- Don't promise a user "results within 1 hour" for batch.
- Don't depend on batch completing within scheduled window (e.g., "must finish before 6 AM EST" requires real-time path).
- Build retry / fallback for batch SLA breach (rare, but design for it).

### 5.3 The "we need it faster than batch" decision

If the workload genuinely needs sub-hour latency:

- Real-time API (full cost).
- Hybrid: batch where possible, real-time for tail latency.
- Self-hosted with high concurrent inference (if scale justifies).

Don't push batch into roles it doesn't fit; the SLA is the SLA.

### 5.4 The early-completion behavior

Many batches complete early. The architecture should detect early completion:

- Polling with backoff (check every 1m initially; 5m after 10 min; 15m after 1 hour).
- Webhook on completion (if provider supports).

Early completion shouldn't be a surprise that delays downstream processing.

### 5.5 The chunking decision

A 100k-document batch can be split into ten 10k-document batches:

- **Pros.** Parallel submission; each batch independently sized; smaller batches complete faster.
- **Cons.** More batch IDs to track; more polling overhead; possibly more failed batches to retry.

For very large volumes, chunking is typical. Provider-specific batch size limits may force chunking anyway.

### 5.6 The "what if it doesn't complete" runbook

If a batch hasn't completed within the stated SLA:

1. Verify with provider's status page (provider-wide issue?).
2. Verify with provider's API (batch status).
3. Escalate to provider support if status seems stuck.
4. Consider re-submitting (with deduplication discipline).
5. Fall back to real-time API for time-sensitive portions.

Document the runbook; rare but real.

---

## 6. Batch eval and quality assurance

Batch output is the same model as real-time; quality is the same. But integration changes; eval should verify integration.

### 6.1 The "same model, same quality" baseline

Provider-side, batch calls run on the same model with the same parameters. There's no quality difference in the response itself.

Don't run extensive eval to compare batch vs real-time quality; that's a category error.

### 6.2 What to eval instead

- **Integration correctness.** Are requests being submitted correctly? Are results being parsed correctly?
- **Idempotency.** If a batch is re-submitted, are downstream effects deduplicated?
- **Per-request failure handling.** What happens when one request in a batch fails (e.g., content policy refusal)? Is it surfaced or silently dropped?
- **End-to-end latency.** Is the pipeline (submission → result handling → downstream) operating within SLA?

These are integration concerns; the eval suite verifies the pipeline.

### 6.3 The self-hosted batching variant

For self-hosted models, "batching" is different — concurrent inference on the GPU. Continuous batching (vLLM-style) can pack many requests through the same inference cycle, improving throughput.

This doesn't have a 50% discount (you own the infrastructure either way), but it improves utilization and reduces effective per-request cost.

Eval for self-hosted batching: throughput (requests per second), latency distribution, GPU utilization.

### 6.4 The "we batched the wrong thing" failure

Sometimes the batch architecture batches the wrong workload. Symptoms:

- A user is actually waiting and they're getting batch latency.
- The pipeline depends on batch results before a deadline that batch can't guarantee.

Mitigation:

- Annual workload review.
- Per-workload classification (§3.6).
- Move workloads back to real-time when their latency requirements change.

### 6.5 The "batch failed silently" failure

A batch is submitted; never completes; nobody notices. Downstream effects don't happen; data is missing; downstream consumers either fail or operate on stale data.

Mitigation:

- Per-batch monitoring (completion or non-completion).
- Alerts on batches that don't complete within expected time.
- Downstream consumers detect missing data and alert.

### 6.6 The eval for cost-savings

Validate the cost-savings hypothesis:

- Pre-migration: cost on real-time API for these workloads.
- Post-migration: cost on batch + cost of any real-time fallback.
- Net savings: should be ~50% for fully-batch-eligible workloads.

If net savings is much less than 50%, investigate.

---

## 7. Cost accounting for batch

Batch usage shows up differently in cost telemetry.

### 7.1 The batch rate as a separate model entry

In the model catalogue (cross-link to [model-catalogue-and-registry.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/model-strategy/model-catalogue-and-registry.md)), batch is a separate entry:

```yaml
model_id: anthropic:claude-sonnet:4-6
endpoints:
  realtime: https://api.anthropic.com/v1/messages
  batch: https://api.anthropic.com/v1/messages/batches
cost_per_input_token:
  realtime: 0.000003
  batch: 0.0000015  # 50% of realtime
cost_per_output_token:
  realtime: 0.000015
  batch: 0.0000075
```

Cost calculations use the appropriate rate based on which endpoint was used.

### 7.2 The cost-bearing event tag

Each cost-bearing event records whether it was batch or real-time:

```python
@dataclass
class CostBearingEvent:
    tenant_id: str
    feature: str
    timestamp: datetime
    model: str
    endpoint: str  # "realtime" or "batch"
    input_tokens: int
    output_tokens: int
    cost_usd: float
```

Dashboards can break down by endpoint.

### 7.3 The batch / real-time ratio dashboard

A dashboard panel: per-workload, fraction of calls on batch vs real-time. Surfaces workloads that should be more batched.

### 7.4 The chargeback differentiation

Customer-facing chargeback may differentiate batch usage:

```
Workload         Calls    Avg cost   Total
─────────────────────────────────────────────
Realtime calls   12,000   $0.04      $480
Batch calls      40,000   $0.02      $800
                                     ─────
                                     $1,280
```

Or aggregate, depending on customer preference.

### 7.5 The "we forgot to update the rate" trap

When batch rates change (provider price update), the rate table must update. If the table lags:

- Attribution at the old rate; chargeback off.
- Reconciliation drift increases.

Subscribe to provider rate updates; automate or scheduled-update.

### 7.6 The cost projection with batch shift

When planning a workload migration to batch:

```
Current real-time cost: $20k/month
Projected batch cost: $10k/month (50%)
Migration savings: $10k/month
Migration cost: 2 weeks engineering × loaded $5k/week = $10k one-time
Payback: 1 month
```

Most migrations have rapid payback. Use the math to justify engineering work.

---

## 8. Worked Meridian example

Meridian's batch migration moved ~$30k/month of spending from real-time to batch.

### 8.1 The pre-migration state

Before batch adoption (2025), Meridian ran all workloads on real-time APIs:

```
Workload                     Calls/month   Cost/month   Real-time required?
──────────────────────────────────────────────────────────────────────────
Care Coordinator             150k          $30k         Yes
Patient API chat             360k          $8k          Yes
Document ingestion           1.2M          $25k         No
Document embedding           1.2M          $4k          No
Analytics enrichment         2.4M          $15k         No
Compliance backfill          800k          $12k         No
Email summaries              90k           $2k          No (overnight)
```

Total: ~$96k/month. Batch-eligible workloads: $58k/month (Document ingestion + Embedding + Analytics + Compliance + Email).

### 8.2 The migration

Q4 2025: migrated the four largest batch-eligible workloads to Anthropic Batch (for LLM calls) and direct batch processing for embeddings.

Migration sequence:

1. **Email summaries.** Easiest; clear async UX; 1 week to migrate. Saved $1k/month.
2. **Compliance backfill.** Already async; trivial migration; 3 days. Saved $6k/month.
3. **Document ingestion.** Required pipeline refactor (queue + batch builder pattern); 2 weeks. Saved $12.5k/month.
4. **Document embedding.** Bulk embedding API on Anthropic / OpenAI not at 50%; instead moved to self-hosted (separate decision, not strictly batch). Saved $2k/month plus operational benefits.
5. **Analytics enrichment.** Pipeline already async; trivial migration; 1 week. Saved $7.5k/month.

Total saved: ~$29k/month.

### 8.3 The Care Coordinator and Patient API stayed real-time

Both have user-facing latency requirements:

- Care Coordinator: clinician is waiting; needs real-time response.
- Patient API chat: user is in the UI; real-time streaming.

These stayed on real-time. Combined: $38k/month at full rate; no batch path.

### 8.4 The pipeline pattern: document ingestion

```
EHR pushes documents → SQS queue
                         ↓
              Batch builder (every 30 min)
                         ↓
              Anthropic Batch API (classification + extraction)
                         ↓
              Results processor → Vector store + warehouse
```

Batch builder accumulates items every 30 minutes; submits batch; downstream pipeline runs on completion. End-to-end latency: 30-90 minutes worst case; typical 30-45 min.

The user-facing UX: "documents are processed within an hour of upload." Clinicians don't notice; the data is in the EHR / vector store within their normal workflow time.

### 8.5 The Q1 2026 batch outage incident

Anthropic Batch had a 4-hour incident in February 2026; submissions queued and didn't process during the window.

Meridian's pipeline:

- Detected the stall (batches not completing within expected time).
- Alerted on-call.
- Engineer made the call: wait it out (analytics enrichment is hours-tolerant) or fall back to real-time.
- Decision: wait. The downstream pipeline tolerated the delay.

After the outage cleared, batches completed; pipeline caught up; no data lost.

Cost impact: zero (we waited). Time impact: 6-hour data lag in the warehouse.

### 8.6 The migration cost vs savings

Migration engineering: ~4 weeks total across 5 workloads (1 engineer FTE).
Migration one-time cost: ~$20k loaded.
Ongoing savings: $29k/month.
Payback: < 1 month.

Three-year savings (if maintained): ~$1M.

### 8.7 The cost dashboards

Per-feature batch-vs-realtime panel surfaces:

```
Document ingestion:    100% batch
Document embedding:    100% self-hosted
Analytics enrichment:  98% batch (2% real-time for time-sensitive enrichment)
Email summaries:       100% batch
Compliance backfill:   100% batch
Care Coordinator:      0% batch (100% real-time as expected)
Patient API chat:      0% batch (100% real-time as expected)
```

Drift toward real-time on batch-designated workloads triggers investigation.

### 8.8 The customer-facing communication

For the analytics enrichment customer, the slight latency increase (real-time → batch) was communicated:

- Before: ~5 seconds per record.
- After: enrichment available in dashboard within 30-60 minutes of data arrival.

Customer agreed; the dashboard latency was acceptable.

### 8.9 The runbook update

Operations runbooks were updated:

- Batch submission failures: how to retry; how to escalate.
- Batch SLA breach (>24h): provider escalation; real-time fallback for critical paths.
- Per-request failures within a batch: handling for content-policy refusals, oversized inputs.

---

## 9. Anti-patterns

### 9.1 The "real-time by default for everything" reflex

**Pattern.** All workloads use real-time API regardless of need. 50% cost premium paid on workloads where users aren't waiting.

**Corrective.** Workload classification per §3; default to batch for batch-eligible.

### 9.2 The "we can't batch because of personalization" misjudgment

**Pattern.** Team rejects batch because each call is "personalized." Personalization is per-request; batch is just a delivery mechanism; the per-request content is unchanged.

**Corrective.** Audit eligibility criteria per §3.1. Personalization doesn't preclude batch.

### 9.3 The batch-then-poll loop that ties up resources

**Pattern.** Code submits batch, polls every 1 second for 24 hours. Wastes resources; usually completes much sooner.

**Corrective.** Backoff polling (1m, 5m, 15m) or webhook-based completion notification.

### 9.4 The batch result without idempotency

**Pattern.** Batch completes; results processed; pipeline retried because of an issue; results processed again; downstream sees duplicates.

**Corrective.** Custom_id as idempotency key per §4.7. Downstream dedups.

### 9.5 The "batch quality is worse" myth

**Pattern.** Team avoids batch because of perceived quality difference. There isn't one — same model, same parameters.

**Corrective.** Eval the integration (correctness of submission and result handling) not the model (no difference).

### 9.6 The "we'll move to batch next quarter" deferral

**Pattern.** Migration always slips. Real-time cost continues. The payback period assumed at engineering planning is real but unrealized.

**Corrective.** Commit the migration; payback within 1-2 months for most workloads.

### 9.7 The forgotten-batch monitoring gap

**Pattern.** Batch submission silently fails (network error, quota issue). Nobody notices; downstream data is missing.

**Corrective.** Per-batch monitoring; alerts on non-completion.

### 9.8 The cost-not-attributed-to-batch trap

**Pattern.** Cost dashboards don't differentiate batch from real-time. Decisions made without knowing the mix.

**Corrective.** Endpoint tag in cost-bearing event per §7.2; dashboard panel per §7.3.

### 9.9 The "batch fits the deadline" mistake

**Pattern.** Workload has hard deadline (must finish by 6 AM EST). Batch SLA is 24 hours. Sometimes batch completes by 6 AM; sometimes doesn't. Failed deadline.

**Corrective.** Don't depend on batch for hard deadlines. Use real-time or self-hosted for time-sensitive paths.

### 9.10 The batch that's actually multiple sequential batches

**Pattern.** Workload needs N batches in sequence (batch A's output feeds batch B). Total latency is 2x batch SLA = 48 hours. Workflow doesn't account for this.

**Corrective.** Plan workflow latency including sequential dependencies. Sometimes the right answer is parallelize where possible; sometimes the workload needs real-time for some steps.

---

## 10. Findings (sprint-assignable)

### COST-BATCH-001 — Severity: Critical
**Finding.** Batch-eligible workloads running on real-time API at full cost.
**Recommendation.** Workload classification per §3.6; migrate batch-eligible to batch API; expected ~50% cost savings.
**Owner.** AI platform + feature teams, sprint N+1.

### COST-BATCH-002 — Severity: High
**Finding.** No batch / real-time differentiation in cost telemetry.
**Recommendation.** Endpoint tag in cost-bearing events per §7.2; dashboard panel.
**Owner.** observability-eng + AI platform, sprint N+2.

### COST-BATCH-003 — Severity: High
**Finding.** Batch submissions lack monitoring; silent failures possible.
**Recommendation.** Per-batch monitoring per §5.6 and §9.7; alerts on non-completion.
**Owner.** SRE + AI platform, sprint N+2.

### COST-BATCH-004 — Severity: High
**Finding.** Batch results processed without idempotency.
**Recommendation.** Use custom_id as idempotency key per §4.7; downstream dedup.
**Owner.** AI platform, sprint N+2.

### COST-BATCH-005 — Severity: High
**Finding.** Batch poll uses tight loop wasting resources.
**Recommendation.** Backoff polling per §5.4; or webhook-based notification.
**Owner.** AI platform, sprint N+2.

### COST-BATCH-006 — Severity: Medium
**Finding.** Workloads classified once; classification not reviewed annually.
**Recommendation.** Annual workload classification review per §3.6 and §6.4.
**Owner.** AI platform + product, sprint N+3.

### COST-BATCH-007 — Severity: Medium
**Finding.** No queue-ingest-drain-to-batch pattern.
**Recommendation.** Pattern per §4.4 for continuous batch-eligible workloads.
**Owner.** AI platform, sprint N+3.

### COST-BATCH-008 — Severity: Medium
**Finding.** Batch failures (per-request, within a batch) not handled.
**Recommendation.** Per-request error handling per §6.2; surface and retry.
**Owner.** AI platform, sprint N+3.

### COST-BATCH-009 — Severity: Medium
**Finding.** Batch chunking absent for large submissions; submissions exceeding provider limits.
**Recommendation.** Chunking per §5.5; size based on provider limits.
**Owner.** AI platform, sprint N+3.

### COST-BATCH-010 — Severity: Medium
**Finding.** Batch SLA breach runbook absent.
**Recommendation.** Runbook per §5.6; verify quarterly.
**Owner.** SRE + AI platform, sprint N+3.

### COST-BATCH-011 — Severity: Medium
**Finding.** Batch and real-time rates not separately tracked in catalogue.
**Recommendation.** Separate entries / endpoints per §7.1.
**Owner.** AI platform, sprint N+4.

### COST-BATCH-012 — Severity: Medium
**Finding.** Provider rate-update process is manual / informal.
**Recommendation.** Subscribe to provider release notes; automate rate table update where possible.
**Owner.** AI platform + FinOps, sprint N+4.

### COST-BATCH-013 — Severity: Medium
**Finding.** Customer chargeback doesn't show batch usage.
**Recommendation.** Optional batch / real-time breakdown per §7.4 if customer requested.
**Owner.** product + AI platform, sprint N+5.

### COST-BATCH-014 — Severity: Medium
**Finding.** Self-hosted batching (continuous batching) not utilized.
**Recommendation.** vLLM continuous batching or equivalent for self-hosted workloads per §6.3.
**Owner.** AI platform, sprint N+4.

### COST-BATCH-015 — Severity: Low
**Finding.** Batch result storage retention policy absent.
**Recommendation.** Retain results in durable storage for audit window (90 days typical).
**Owner.** AI platform + compliance, sprint N+5.

### COST-BATCH-016 — Severity: Low
**Finding.** Batch / real-time ratio not surfaced per workload.
**Recommendation.** Dashboard panel per §7.3.
**Owner.** observability-eng, sprint N+5.

### COST-BATCH-017 — Severity: Low
**Finding.** Sequential batch chains not optimized.
**Recommendation.** Identify parallel-able steps; pipeline parallelization per §9.10.
**Owner.** AI platform, sprint N+6.

### COST-BATCH-018 — Severity: Low
**Finding.** Batch migration payback not tracked.
**Recommendation.** Track migration cost vs ongoing savings; report quarterly.
**Owner.** FinOps + AI platform, sprint N+6.

---

## 11. Adoption sequencing checklist

- [ ] **Audit workloads (§3.6).** Classify each as batch-eligible / real-time-required.
- [ ] **Compute potential savings.** Volume × current rate × 50% per batch-eligible workload.
- [ ] **Pick first migration.** Smallest, simplest batch-eligible workload (e.g., email summaries).
- [ ] **Implement integration pattern (§4).** Submit-poll for simple; queue + builder for continuous.
- [ ] **Add custom_id idempotency (§4.7).**
- [ ] **Monitor batch submissions (§5).** Status; expected completion; alerts on non-completion.
- [ ] **Update cost telemetry (§7).** Endpoint tag; dashboard panel.
- [ ] **Migrate workload; validate end-to-end.** Pilot with sample; ramp to full.
- [ ] **Repeat for next batch-eligible workload.**
- [ ] **Build SLA breach runbook (§5.6).**
- [ ] **Annual workload classification review.**
- [ ] **Track migration savings.** Verify ~50% per migrated workload.

---

## 12. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — attribution tracks batch usage separately.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) — dashboards surface batch / real-time ratio.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) — batch and real-time charged at different rates.
- [caching-for-cost.md](./caching-for-cost.md) — caching + batching combine for cost.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — routing decisions include batch eligibility.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) *(companion)* — batch quota is separate from real-time.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — batch cost included in budget tracking.

**Elsewhere in this repo.**
- [agent-engineering/agent-loop-design.md](../agent-engineering/agent-loop-design.md) — agents can't generally batch; understand the constraint.
- [rag-engineering/ingestion-pipeline-engineering.md](../rag-engineering/ingestion-pipeline-engineering.md) — ingestion is often the batch use case.

**Sibling repos.**
- [ai-architecture-reference-architecture / integration-architecture / sync-vs-async-vs-streaming.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/sync-vs-async-vs-streaming.md) — the shape decision; batch is a flavor of async.
- [ai-architecture-reference-architecture / integration-architecture / event-driven-ai-integration.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/event-driven-ai-integration.md) — event-driven patterns that wrap batch submissions.
- [ai-architecture-reference-architecture / integration-architecture / callback-and-webhook-patterns.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/callback-and-webhook-patterns.md) — webhook integration for batch completion.

**External.**
- Anthropic Batch API documentation.
- OpenAI Batch API documentation.
- Google Vertex AI batch prediction documentation.
- AWS Bedrock batch inference documentation.
- vLLM documentation on continuous batching.
