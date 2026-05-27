# Caching for Cost

> **Audience.** Engineers whose AI bill is growing 30%+ per quarter and the next obvious step is "cache more aggressively." Tech leads building cache infrastructure for AI workloads. SREs whose cache-hit-rate dashboard panel is empty. Anyone who's heard "we don't cache because of personalization" and is wondering if that's actually true. **Scope.** The *engineering* practice of caching to reduce AI cost: four cache tiers (prompt-prefix, response, semantic, retrieval); engineering patterns per tier; cache-hit-rate as SLI; cache invalidation discipline; multi-tenant isolation in caches. Not the architectural decision of when to cache (mostly: cache more than you think). Not the broader cost attribution (see [cost-attribution.md](./cost-attribution.md)). Not the rate-limit infrastructure (see [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Caching is the single highest-leverage cost lever in AI engineering. Most teams under-cache; the typical AI workload has 30-60% cacheable calls that aren't cached because:

- "We didn't think about it."
- "We tried it once and the invalidation was hard."
- "Our calls are too personalized to cache."
- "We don't have cache infrastructure."

Each of these is wrong for at least one of the four cache tiers available in 2026. The correct posture is "cache aggressively at every tier where it applies; engineer the invalidation discipline; measure hit rate; iterate."

The four tiers, in increasing order of complexity:

- **Prompt-prefix cache (provider-side).** Anthropic's prompt caching and OpenAI's prompt caching let you cache the static prefix of a prompt; subsequent calls reusing the prefix pay 10% (Anthropic) or 50% (OpenAI's cached_tokens at half-price) of the normal input cost. Near-free to enable; immediate 30-60% cost reduction for workloads with long stable prefixes.
- **Response cache (exact match).** Hash the (model + prompt + parameters) and cache the response. Returns instantly on cache hit; 100% cost savings on hits. Works for any workload where the same input produces the same response.
- **Semantic cache (similarity match).** Hash isn't enough — "What's the refund policy?" and "How do I get a refund?" should hit the same cache entry. Embedding-similarity matching: cache entries are embedded; new queries are embedded; similar queries hit. Higher hit rate than exact-match; risk of wrong-cache-hits.
- **Retrieval cache.** Cache the retrieved documents for a given query embedding. The LLM call still runs; retrieval doesn't. Saves vector store cost and latency; preserves the LLM's freshness.

A typical mature AI workload uses all four tiers in combination. The compound effect: 60-80% cost reduction is plausible for the right workload mix.

This document covers the engineering of each tier: what to cache, where, with what key, how to invalidate, how to measure. The architecture's role is acknowledging that caching is non-optional; this document is how to make it actually work.

This document is opinionated about four things:

1. **Provider-side prompt caching is the highest ROI cache to enable.** It's nearly free to configure; the savings are immediate; the only reason not to enable it is workload-specific (very short prompts where the discount doesn't apply, or prompts where every call has a different prefix).
2. **Exact-match response caching catches more than teams expect.** "But our queries are personalized" is usually overstated; some fraction of calls are non-personalized (system prompts, repeated FAQ-shaped queries, agent boilerplate). Measure before assuming caching doesn't apply.
3. **Semantic caching is powerful and risky.** Higher hit rate than exact-match; but wrong-cache-hits (a semantically-similar query that actually wanted a different answer) produce incorrect responses. Implement carefully; tune similarity threshold; monitor for false positives.
4. **Cache invalidation is the hard part; don't pretend otherwise.** The pattern that fails is "we'll figure out invalidation later." The pattern that works is "we choose what to cache based on how easy invalidation is."

Structure: (2) tier 1 — prompt-prefix cache; (3) tier 2 — response cache; (4) tier 3 — semantic cache; (5) tier 4 — retrieval cache; (6) cache-hit-rate as SLI; (7) cache invalidation discipline; (8) multi-tenant cache isolation; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption sequencing; (13) references.

---

## 2. Tier 1: Prompt-prefix cache (provider-side)

The lowest-effort, highest-immediate-return cache tier. Most teams should enable it on day one.

### 2.1 What it is

Major hosted providers (Anthropic, OpenAI, Google) offer prompt caching: the static prefix of a prompt is cached on the provider's side; subsequent calls with the same prefix pay a fraction of the normal input-token cost for the cached portion.

**Anthropic prompt caching (2026).** Cache writes cost 1.25x input rate; cache hits cost 0.1x input rate (90% discount on cached tokens). Cache TTL: 5 minutes default, 1 hour with extended-ttl flag.

**OpenAI prompt caching (2026).** Cached input tokens charged at 0.5x normal rate (50% discount). Cache TTL: ~5-10 minutes.

**Google Vertex (2026).** Context caching API; per-token storage cost; reads at reduced rate.

Different mechanics; same principle.

### 2.2 What to cache

The static prefix of the prompt: system prompt, few-shot examples, retrieved context that's stable, tool definitions. Anything that doesn't change between calls.

The variable suffix (current user message, current request) is not cached.

For a typical agent prompt:

```
[Cached prefix: ~6000 tokens]
System prompt
Few-shot examples
Tool definitions

[Variable suffix: ~500 tokens]
Current user message
```

The 6000 cached tokens cost 600 token-units (10% of normal) on cache hits, saving 5400 token-units per call.

### 2.3 The configuration

Anthropic:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    system=[
        {
            "type": "text",
            "text": large_static_system_prompt,
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[
        {"role": "user", "content": current_user_message}
    ]
)
```

The `cache_control` marker tells Anthropic to cache. Subsequent calls with the same system content hit the cache.

### 2.4 The break-even

Cache hits cost 0.1x; cache writes cost 1.25x. Break-even is at 2.5 hits per write (1.25 + 0.1×N = 1×N + 1.25; solve for N=2.5 ish actually let me recompute: cache write costs 1.25 per write; cache hits cost 0.1 per hit. For N total calls with the same prefix, cost is 1.25 + 0.1×(N-1) instead of 1×N. Break-even when 1.25 + 0.1(N-1) = N, i.e., 1.25 - 0.1 = 0.9N, N ≈ 1.28). So break-even is 2 calls; any prefix that's used 2+ times is profitable to cache.

For typical agent prompts (each prefix used hundreds of times across users), savings are dramatic.

### 2.5 The 5-minute TTL trap

Anthropic's default TTL is 5 minutes. If your traffic has gaps > 5 minutes, the cache expires; the next call writes again (1.25x cost). For bursty workloads, this is fine. For sparse workloads, cache lookups fail repeatedly.

**Mitigation.** Extended TTL (1 hour) for sparse-but-predictable workloads. Costs slightly more per write; pays off if the prefix is reused.

### 2.6 The "prefix changed between calls" trap

If the cached prefix differs by even one token between calls, the cache misses. Common sources of unintended cache miss:

- Timestamp in the system prompt ("Today is 2026-05-27").
- User ID in the prefix.
- Retrieved-context that differs (because it should).

**Mitigation.** Stable prefix design. Move variable content out of the cached portion. Timestamps move to the user message; user IDs are passed as variables; retrieved-context is either separately cached (not in the prefix) or accepted as cache-miss.

### 2.7 The agent-loop bonus

In agent loops, the prefix grows over turns (tool calls, tool results accumulate). The prefix caching can be applied at multiple boundaries:

- Cache after the system prompt.
- Cache after the first tool result.
- Cache after the second tool result.

Each cache boundary lets the next turn benefit from caching of everything up to that point. Anthropic supports up to 4 cache breakpoints.

### 2.8 The "is provider caching enabled" SLO

Track the fraction of input tokens that are cache hits:

```
cache_hit_token_fraction = cache_hit_tokens / total_input_tokens
```

For workloads expected to benefit from prompt caching, this should be > 50%. If lower, the prefix isn't stable enough, or the TTL is expiring.

---

## 3. Tier 2: Response cache (exact-match)

The classic cache. Same input → same response, served from cache.

### 3.1 What it is

Hash the request (model + prompt + parameters + tenant_id), lookup in a cache, return if hit.

```python
def cached_llm_call(context: RequestContext, prompt: str, model: str, **params) -> str:
    cache_key = compute_cache_key(context.tenant_id, prompt, model, params)
    cached = response_cache.get(cache_key)
    if cached:
        return cached

    response = llm_call(prompt, model, **params)
    response_cache.set(cache_key, response, ttl=appropriate_ttl)
    return response
```

100% cost saving on hits; ~1ms latency vs seconds for a fresh call.

### 3.2 When it works

- Same input is asked frequently (FAQ shapes, repeated workflows).
- Output doesn't need to be unique per call (no "be creative" element).
- The input naturally produces the same output (deterministic).

Typical fits:

- Classification calls (each unique document needs to be classified once; future identical documents reuse).
- Boilerplate generation (the system's standard response to a specific query class).
- Lookup-style queries (extracting structured fields from a known document).

### 3.3 When it doesn't work

- Conversational chat (every turn has different conversation history).
- Personalized recommendations (per-user output).
- Time-sensitive responses (changes with current state).
- Creative generation (variation is desired).

For these, response cache is rare hit. Other tiers may apply.

### 3.4 The cache key composition

```python
def compute_cache_key(tenant_id, prompt, model, params) -> str:
    key_parts = [
        tenant_id,  # tenant isolation
        model,      # different model = different response
        prompt,     # the prompt itself
        params.get("temperature", 0),  # temperature affects output
        params.get("max_tokens", 4096),
        params.get("system", ""),  # system prompt
        # Note: temperature 0 = deterministic; higher temperature = non-deterministic
    ]
    return hashlib.sha256("|".join(map(str, key_parts)).encode()).hexdigest()
```

Tenant ID in the key prevents cross-tenant hits (cross-link to §8).

### 3.5 The temperature consideration

At `temperature=0`, the model is mostly deterministic; same input produces same output. At higher temperatures, output varies between calls.

**Pattern.** For cacheable workloads, set `temperature=0` and cache. Sacrifices output diversity for cache hits.

**Pattern.** For non-deterministic workloads, cache anyway and accept that the cached response is "an" answer to this prompt, not "the" answer.

### 3.6 The TTL

Response cache TTL depends on what's being cached:

- Static facts (e.g., "what does ICD-10 code I10 mean"): days to weeks.
- Document classification: until the document changes.
- Workflow outputs: hours to days.
- Anything dependent on current state: minutes.

Set per-workload. Default conservatively (e.g., 1 hour); extend where appropriate.

### 3.7 The storage tier

- **In-memory LRU.** Fast; small; per-process. Good for hot keys.
- **Redis.** Fast; shared; medium-sized. Good for cross-process hits.
- **DynamoDB / Cassandra.** Slower; large; persistent. Good for long TTLs.

Tiered cache: in-memory L1 (fastest), Redis L2 (shared), DynamoDB L3 (persistent). Workload-appropriate.

### 3.8 The cache-hit-rate target

For workloads suited to response cache, target hit rate > 30%. Lower suggests the cache key includes more variability than necessary; investigate.

---

## 4. Tier 3: Semantic cache

Same intent, different wording. Higher hit rate than exact-match; risk of wrong cache hits.

### 4.1 What it is

Cache entries are stored with their query embeddings. New queries are embedded; similar embeddings hit the cache.

```python
def semantic_cached_llm_call(context: RequestContext, query: str, threshold=0.92) -> str:
    query_embedding = embed(query)

    cached_match = semantic_cache.find_similar(
        query_embedding,
        tenant_id=context.tenant_id,
        threshold=threshold
    )

    if cached_match:
        return cached_match.response

    response = llm_call(query)
    semantic_cache.store(query_embedding, query, response, context.tenant_id)
    return response
```

Higher hit rate than exact match (because "What's the refund policy?" and "How do I get a refund?" both hit).

### 4.2 The similarity threshold trade-off

Higher threshold (e.g., 0.95): fewer hits, fewer wrong hits.
Lower threshold (e.g., 0.85): more hits, more wrong hits.

Tuning depends on workload tolerance:

- FAQ-style chat: threshold ~0.92 (some wrong hits acceptable; user can clarify).
- Clinical queries: threshold ~0.97+ (wrong hits are dangerous; prefer cache miss to wrong cache hit).
- Internal copilot: threshold ~0.90 (lower precision OK).

### 4.3 The wrong-cache-hit risk

A semantic cache may return the response for a semantically-similar but actually-different query. Examples:

- "Will my insurance cover MRI?" cached → returned for "Will my insurance cover CT?" (different procedure).
- "What's our return policy?" cached → returned for "What's our exchange policy?" (different).

The cached response is wrong for the new query.

**Mitigation.** High similarity threshold. Domain-specific embedding model tuned for the workload. Human review of cache content; flagging wrong-hits.

### 4.4 The cache-store discipline

Stored cache entries must include enough metadata to verify a hit:

```python
@dataclass
class SemanticCacheEntry:
    query_embedding: list[float]
    query_text: str
    response: str
    tenant_id: str
    cached_at: datetime
    confidence: float  # quality score of the response
    embedding_model_version: str
```

When the embedding model changes, old entries' embeddings are no longer comparable; cache must be invalidated.

### 4.5 The hit-rate-vs-quality measurement

Track both:

- **Cache hit rate.** Fraction of queries hitting the cache.
- **Hit quality.** Sampling-based: human review of cache hits; what fraction are "correct" for the new query?

If hit rate is 50% but quality is 60% (40% of hits are wrong), the cache is harmful overall. Raise threshold or remove cache.

### 4.6 The "cache only high-quality responses" filter

Not every LLM response should be cached. Cache only:

- Responses with high model-confidence (no refusals, no uncertainty markers).
- Responses validated by some external check (passed schema validation, passed eval).
- Responses that have been served and not flagged by user.

Bad responses in the cache produce repeated bad responses on hits.

### 4.7 The semantic cache as RAG-without-LLM

Semantic cache can be seen as a special RAG: retrieve previous Q&A pairs; if similar enough, use the answer. The LLM call is skipped entirely.

For high-volume FAQ workloads, semantic cache hit rates of 40-70% are achievable.

---

## 5. Tier 4: Retrieval cache

Cache the retrieved documents for a given query embedding. The LLM call still runs; retrieval doesn't.

### 5.1 What it is

For RAG workloads, retrieval is one of two LLM-related costs (the LLM call + the vector store query). Caching retrieval reduces the vector store cost and latency.

```python
def cached_retrieve(context: RequestContext, query_embedding: Embedding, k=10) -> list[Chunk]:
    cache_key = compute_retrieval_cache_key(context.tenant_id, query_embedding, k)
    cached = retrieval_cache.get(cache_key)
    if cached:
        return cached

    results = vector_store.search(query_embedding, k=k, filter={"tenant_id": context.tenant_id})
    retrieval_cache.set(cache_key, results, ttl=appropriate_ttl)
    return results
```

### 5.2 When it helps

- Vector store latency is significant (5-50ms typical).
- Vector store has QPS cost.
- Query patterns have repeats.

Common for embedding-cache: the LLM call output may vary based on inputs, but the retrieved context for a given query is stable.

### 5.3 The cache key for retrieval

The cache key must capture:

- Tenant ID.
- Query embedding (or a hash of it).
- k (number of results).
- Filter parameters (date range, document type).

```python
def compute_retrieval_cache_key(tenant_id, embedding, k, **filters) -> str:
    embedding_hash = hash_embedding(embedding, precision=4)  # round to handle minor variations
    return f"{tenant_id}:{embedding_hash}:{k}:{stable_serialize(filters)}"
```

Embedding hashing with reduced precision allows near-identical queries (the embedding of "what's the refund policy" vs "what's the refund policy?" differs by tiny floating-point amounts) to hit the same cache entry.

### 5.4 The TTL for retrieval cache

Depends on how often the underlying corpus changes:

- Static corpus (reference docs, manuals): days.
- Slowly-changing (FAQs, policies): hours.
- Fast-changing (news, social media): minutes.

Document update events should invalidate affected cache entries (cross-link to §7).

### 5.5 The retrieval cache as a hot-key optimizer

In multi-tenant workloads, hot queries dominate. Caching their retrieval results:

- Reduces vector store load (hot queries don't repeat-hit the store).
- Reduces latency (cache lookups are faster than vector queries).
- Reduces per-tenant cost (if you're paying per-vector-query).

For workloads where 20% of queries account for 80% of volume, retrieval caching can reduce vector store load by 60%+.

### 5.6 The combination with semantic cache

Retrieval cache + response cache combination:

```
Query →
    Semantic cache hit? → Return cached response (no LLM, no retrieval).
    No → Retrieval cache hit? → LLM call with cached context.
    No → Retrieve from vector store → LLM call → Cache both response and retrieval.
```

Layered caching. Each layer catches some queries; the combined hit rate is much higher than either alone.

---

## 6. Cache-hit-rate as SLI

Cache effectiveness needs measurement to be managed.

### 6.1 The per-tier hit-rate SLI

Track hit rate per tier:

- Prompt-prefix cache hit rate: fraction of input tokens served from cache.
- Response cache hit rate: fraction of LLM-call requests served from cache.
- Semantic cache hit rate: fraction of queries hitting the semantic cache.
- Retrieval cache hit rate: fraction of retrievals served from cache.

Each is a separate SLI with separate targets.

### 6.2 The hit-rate dashboard

A dashboard panel per tier:

- Hit rate over time (last 7 days, hourly granularity).
- Hit rate by feature (different features have different cacheability).
- Hit rate by tenant (different tenants may have different patterns).

Trends reveal cache health. Drops trigger investigation.

### 6.3 The hit-rate target

Set targets per tier and per workload:

- Prompt-prefix on stable agent workloads: > 50%.
- Response cache on FAQ workloads: > 30%.
- Semantic cache on conversational workloads: > 20%.
- Retrieval cache on repetitive query workloads: > 40%.

Below target: investigate (key changes, TTL too short, content changing too fast).

### 6.4 The hit-rate-equivalent-cost-savings

Translate hit rate to cost savings:

```
cost_savings = hit_count × avg_call_cost
```

Where `avg_call_cost` is the cost of what would have been an LLM call (or vector query, for retrieval cache).

Surface this on cost dashboards: "caching saved $4,200 last month."

### 6.5 The miss-reason categorization

When a cache misses, categorize why:

- Cold cache (first time this key is seen).
- TTL expired (would have hit but entry was cleared).
- Key mismatch (request differs from cached request).
- Tenant mismatch (different tenant).

Categorization guides improvement. If TTL-expired is dominant, extend TTL. If key-mismatch is dominant, investigate which fields are unstable.

### 6.6 The hit-rate alerting

Alerts when hit rate drops:

- Per-tier hit rate < expected for > 1 hour.
- Hit rate dropped > 20% from rolling average.
- Cache size near capacity (eviction pressure rising).

Cross-link to [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md).

---

## 7. Cache invalidation discipline

The hard part. Wrong invalidation → stale responses; missed invalidation → wrong responses indefinitely; over-invalidation → cache thrash.

### 7.1 The invalidation patterns

**TTL-based.** Cache entries expire after a fixed duration. Simplest; works for stale-OK content.

**Event-driven.** Invalidation on specific events (document updated, prompt version changed). Most precise; most complex.

**Versioned.** Cache key includes a version (prompt_version, model_version, corpus_version). Version change invalidates all entries with the old version. Simpler than event-driven; precise enough for most cases.

**Hybrid.** Versioned + TTL: entries expire after TTL or version change, whichever first.

### 7.2 The "what triggers invalidation"

For each cache, document the invalidation triggers:

- Prompt-prefix cache: prefix change (system prompt, few-shot examples, tool defs).
- Response cache: prompt version change, model version change, deterministic-content change.
- Semantic cache: embedding model version change, response-quality flag, time-based expiry.
- Retrieval cache: document update, document deletion, vector store re-index.

Each trigger maps to invalidation action.

### 7.3 The document update propagation

For retrieval cache, document updates must invalidate affected cache entries:

```python
def on_document_updated(document_id, tenant_id):
    # All retrieval cache entries that returned this document are stale
    affected_entries = retrieval_cache.find_by_document_id(document_id, tenant_id)
    for entry in affected_entries:
        retrieval_cache.invalidate(entry.key)
```

Requires tracking which entries returned which documents — a reverse index.

Alternative: aggressive TTL on retrieval cache (e.g., 10 minutes); document updates wait up to TTL to propagate. Acceptable for slowly-changing corpora.

### 7.4 The prompt-version invalidation

When a prompt is updated, all response cache entries based on the old prompt are stale. Two patterns:

**Hard invalidation.** Delete all cache entries with the old prompt_version. Cache thrashes briefly; fresh entries build up.

**Soft invalidation.** Old entries remain but mark as "stale"; next access either uses them with disclaimer or triggers fresh call. Less thrash; risk of serving stale.

Hard is preferred for correctness; soft for performance-critical paths.

### 7.5 The model-version invalidation

Model upgrades (new Claude version, new GPT version) usually invalidate caches:

- Output may differ between versions.
- Some workloads benefit from re-running on new model.

Pattern: cache key includes `model_version`; upgrade is a version change; old entries expire naturally (or are deleted).

### 7.6 The "the wrong response was cached" recovery

Sometimes a wrong response gets cached and persists. Detection:

- User reports.
- Quality drift detection flags the response.
- Eval suite catches the regression.

Recovery:

- Find the cache entry.
- Invalidate.
- Investigate root cause (was the LLM wrong? was the input misinterpreted?).
- Fix the root cause.

The architecture must support: per-entry invalidation by key; bulk invalidation by pattern; pause caching during investigation.

### 7.7 The "don't cache low-confidence responses" filter

Some responses shouldn't be cached:

- Model refused or expressed uncertainty.
- Response failed schema validation.
- Response was flagged by safety filter.
- Response has user-feedback flag.

These responses are excluded from cache at write time. The "what to cache" decision is made per-response, not per-call.

---

## 8. Multi-tenant cache isolation

Caches must respect tenant boundaries. Cross-tenant cache hits are a multi-tenant leak.

### 8.1 The tenant_id in cache key

Every cache key includes tenant_id:

```python
cache_key = f"tenant:{tenant_id}:..."
```

Two tenants asking the same question hit different cache entries.

### 8.2 The "tenant-agnostic content" exception

Some content is genuinely cross-tenant: public reference data, common system responses. For these, a separate tenant-agnostic cache:

```python
cache_key = f"shared:..."  # not tenant-scoped
```

Restricted to content that has been explicitly classified as tenant-agnostic.

### 8.3 The cross-tenant cache hit risk

If tenant_id is missing from the key, tenant A's cached response is returned to tenant B. The leak.

**Mitigation.** Lint rules that require tenant_id in cache keys. Code review. Cache wrapper enforces inclusion.

### 8.4 The per-tenant cache size limits

Per-tenant cache size limits prevent one tenant from filling the cache with their entries:

```python
def cache_set(tenant_id, key, value):
    if tenant_cache_size(tenant_id) > tenant_cache_limit(tenant_id):
        evict_oldest_for_tenant(tenant_id)
    cache.set(key, value)
```

Per-tenant limits scale with tenant tier.

### 8.5 The cache-eviction policy

When the cache is full, eviction policy:

- LRU (least recently used): default; popular.
- LFU (least frequently used): biased toward hot content.
- TTL-only: no eviction beyond TTL; cache grows to size limit then refuses new writes.

For multi-tenant, ensure eviction is per-tenant (one tenant's heavy use doesn't evict another's hot content).

### 8.6 The compliance audit

Cache content includes responses, which may include PII. Audit:

- Cache encryption at rest.
- Cache access logging.
- Cache retention policy (don't keep PII in cache longer than necessary).
- Cache deletion on tenant offboarding.

Cross-link to [ai-architecture-reference-architecture / multi-tenancy-and-isolation / cross-tenant-leakage-prevention.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/cross-tenant-leakage-prevention.md).

---

## 9. Worked Meridian example

Meridian's caching reduced AI cost by ~$45k/month across the platform.

### 9.1 The caching deployment

```
Workload                          Tier 1 (prefix) | Tier 2 (response) | Tier 3 (semantic) | Tier 4 (retrieval)
─────────────────────────────────────────────────────────────────────────────────────────────────────────
Care Coordinator agent            ✓               | ×                 | ×                 | ✓
Patient API chat                  ✓               | ✓                 | ✓                 | ✓
Analytics warehouse copilot       ✓               | ×                 | ×                 | ✓
Document ingestion classify       ✓               | ✓                 | ×                 | ×
Document embedding                ×               | ✓                 | ×                 | ×
Billing-code workflow             ✓               | ✓                 | ×                 | ×
Internal IT copilot               ✓               | ✓                 | ✓                 | ✓
```

Each workload uses applicable tiers.

### 9.2 The hit rates by tier

```
Tier                        Hit rate    Cost saved/month
────────────────────────────────────────────────────────
Prompt-prefix (Anthropic)   55%         $18,000
Response cache              22%         $8,500
Semantic cache              30%         $12,000
Retrieval cache             48%         $4,200 (vector store + latency)
                                        ─────────
                            Total       $42,700
```

### 9.3 The Care Coordinator agent prefix caching

The Care Coordinator's prefix is ~7000 tokens (system prompt + tool definitions + few-shot examples). Each agent task makes 5-10 LLM calls; all calls share the same prefix.

With caching:
- Cache write: 7000 tokens × 1.25 = 8750 token-units once per session.
- Cache hits: 7000 × 0.10 = 700 token-units per subsequent call.
- Without caching: 7000 × 1.0 = 7000 token-units per call.

For a 7-call session: 8750 + 6 × 700 = 12,950 vs uncached 7 × 7000 = 49,000 token-units. **74% savings** on input tokens for the cached portion.

Across all Care Coordinator usage: $18k/month saved on prefix caching alone.

### 9.4 The Patient API chat semantic cache

The chat receives many variations of common questions:
- "How do I access my records?"
- "Where do I see my records?"
- "Can I view my health records?"

All semantically similar. Semantic cache (threshold 0.92):
- Hit rate: 30%.
- Wrong-hit rate (sampled): < 1%.
- Cost saved: $12k/month.

The high hit rate reflects the FAQ-shaped nature of patient queries.

### 9.5 The document classification response cache

Each unique document hits the classifier once; classifications are cached by document hash. Documents are immutable (Meridian doesn't modify clinical notes), so the cache is permanent.

- Cache hit rate: 22% (varies; some workflows re-classify after updates to classification taxonomy).
- Cost saved: $8.5k/month.

### 9.6 The retrieval cache for repeated queries

The Care Coordinator agent often queries for the same patient context across the agent's steps. Retrieval cache (per-session):

- Hit rate: 48%.
- Latency saved: ~80ms per cache hit (Pinecone query latency).
- Cost saved: ~$4k/month in vector store cost; significant latency improvement.

### 9.7 The invalidation discipline

- Prompt-prefix: TTL 5 minutes (default); extended TTL 1 hour on certain prompts; auto-invalidate on system-prompt version change.
- Response cache: TTL 24 hours; invalidate on prompt version change (cache key includes prompt_version).
- Semantic cache: TTL 7 days; invalidate on embedding model version change; manual flag on quality issues.
- Retrieval cache: TTL 10 minutes per query; invalidate on document update (reverse index).

### 9.8 The wrong-cache-hit incident (Q1 2026)

A semantic cache miss-tuning incident: threshold lowered from 0.92 to 0.88 in an experiment. Wrong-hit rate jumped from < 1% to 8%. Detected within 4 hours by quality drift detection.

Mitigation: threshold restored; affected cache entries purged; quality returned to baseline.

Post-incident: semantic cache threshold changes now require eval-suite approval before deployment.

### 9.9 The dashboards

Cache dashboards per tier; per-feature breakdown; per-tenant breakdown. Monthly review of:
- Hit rate trends.
- Cost savings.
- Wrong-hit detections.

### 9.10 The infrastructure cost

- Redis cluster (cache layer): $1500/month.
- Vector index for semantic cache: $400/month.
- Engineering: ~5 weeks to build out the four tiers (1.5 engineers).

ROI: $45k/month savings vs ~$2k/month infrastructure cost. Net $43k/month.

---

## 10. Anti-patterns

### 10.1 The "we don't cache because of personalization" claim

**Pattern.** Team rejects caching because "every user is different." Reality: substantial fraction of calls are non-personalized (FAQ, system responses, repeated workflows). Measure cacheable fraction before claiming none exists.

**Corrective.** Audit a sample of calls; measure exact-match and semantic-match hit potential.

### 10.2 The cache that's never invalidated

**Pattern.** Cache entries set without TTL. Once written, they persist forever. Stale content accumulates; quality degrades over months.

**Corrective.** TTL on every cache entry. Invalidation triggers documented.

### 10.3 The cache with no hit-rate monitoring

**Pattern.** Cache is implemented; nobody knows if it's working. Hit rate could be 5% or 50%.

**Corrective.** Hit rate dashboard per §6. Targets per tier.

### 10.4 The semantic cache with no threshold tuning

**Pattern.** Semantic cache deployed with default threshold; never tuned to workload. Either too tight (low hit rate) or too loose (wrong hits).

**Corrective.** Workload-specific threshold; tuned with eval suite; monitored for wrong hits.

### 10.5 The cache key missing tenant_id

**Pattern.** Cache key is just hash of prompt. Tenant A's response served to tenant B.

**Corrective.** Tenant_id in every cache key per §8.1.

### 10.6 The "we'll cache the bad responses too" inclusion

**Pattern.** Every LLM response cached, including refusals, validation failures, and low-quality outputs.

**Corrective.** Filter at write per §7.7.

### 10.7 The cache that grows unboundedly

**Pattern.** No size limit. Cache memory grows; eventually hits OOM; process restarts; cache lost.

**Corrective.** Size limits + eviction policy. Per-tenant size limits in multi-tenant systems.

### 10.8 The prompt-prefix cache not enabled

**Pattern.** Provider supports prompt caching; team hasn't enabled it. Free 30-60% cost reduction left on the table.

**Corrective.** Enable provider-side caching as a day-one config. Cross-link to §2.

### 10.9 The "cache the entire LLM output and ignore stale-tolerance" misuse

**Pattern.** Time-sensitive workloads cache responses for hours. Users get stale data.

**Corrective.** TTL appropriate to workload's freshness needs. Time-sensitive workloads have short TTL or no cache.

### 10.10 The cache-bypass that always wins

**Pattern.** Workload has `force_fresh=true` flag for cache bypass. Some callers always set it. Cache is bypassed for those callers; hit rate is misleading.

**Corrective.** Audit usage of bypass flags; reduce; require justification.

---

## 11. Findings (sprint-assignable)

### COST-CACHE-001 — Severity: Critical
**Finding.** Provider-side prompt caching not enabled.
**Recommendation.** Enable Anthropic prompt caching / OpenAI prompt caching per §2; measure hit rate.
**Owner.** AI platform, sprint N+1.

### COST-CACHE-002 — Severity: Critical
**Finding.** No response cache for cacheable workloads (classification, FAQ).
**Recommendation.** Implement response cache per §3 with tenant-scoped keys.
**Owner.** AI platform, sprint N+1.

### COST-CACHE-003 — Severity: Critical
**Finding.** Cache keys missing tenant_id.
**Recommendation.** Add tenant_id to all cache keys per §8.1; lint rule.
**Owner.** AI platform + security, sprint N+1.

### COST-CACHE-004 — Severity: High
**Finding.** No cache hit-rate monitoring.
**Recommendation.** Per-tier hit-rate dashboard per §6.
**Owner.** observability-eng, sprint N+2.

### COST-CACHE-005 — Severity: High
**Finding.** Semantic cache not implemented for high-volume FAQ workloads.
**Recommendation.** Semantic cache per §4 with workload-tuned threshold.
**Owner.** AI platform, sprint N+2.

### COST-CACHE-006 — Severity: High
**Finding.** Retrieval cache absent for RAG workloads.
**Recommendation.** Retrieval cache per §5; per-session for agent workloads.
**Owner.** AI platform, sprint N+2.

### COST-CACHE-007 — Severity: High
**Finding.** Cache TTL not workload-tuned.
**Recommendation.** TTL per workload per §3.6; documented.
**Owner.** AI platform, sprint N+2.

### COST-CACHE-008 — Severity: High
**Finding.** Cache invalidation triggers undocumented.
**Recommendation.** Document triggers per §7.2; implement invalidation pipeline.
**Owner.** AI platform, sprint N+2.

### COST-CACHE-009 — Severity: Medium
**Finding.** Prompt-version not in cache key; prompt updates produce stale hits.
**Recommendation.** Include prompt_version in response cache key per §7.4.
**Owner.** AI platform, sprint N+3.

### COST-CACHE-010 — Severity: Medium
**Finding.** Bad responses (refusals, validation failures) being cached.
**Recommendation.** Quality filter at cache write per §7.7.
**Owner.** AI platform, sprint N+3.

### COST-CACHE-011 — Severity: Medium
**Finding.** Semantic cache wrong-hit rate not monitored.
**Recommendation.** Sampled human review per §4.5; track wrong-hit rate.
**Owner.** AI platform + product, sprint N+3.

### COST-CACHE-012 — Severity: Medium
**Finding.** Document update doesn't invalidate retrieval cache.
**Recommendation.** Reverse-index for retrieval cache; invalidation on document update per §7.3.
**Owner.** AI platform, sprint N+3.

### COST-CACHE-013 — Severity: Medium
**Finding.** Cache size unbounded; memory pressure occasional.
**Recommendation.** Size limit + eviction policy per §8.4 and §8.5.
**Owner.** AI platform, sprint N+3.

### COST-CACHE-014 — Severity: Medium
**Finding.** Per-tenant cache size not limited.
**Recommendation.** Per-tenant size cap per §8.4; tier-based.
**Owner.** AI platform, sprint N+4.

### COST-CACHE-015 — Severity: Medium
**Finding.** Tiered cache (L1/L2/L3) not implemented; single Redis layer at all sizes.
**Recommendation.** In-memory L1 for hot keys per §3.7; reduce Redis load.
**Owner.** AI platform, sprint N+4.

### COST-CACHE-016 — Severity: Low
**Finding.** Hit-rate translates not surfaced as cost savings on dashboards.
**Recommendation.** Compute and display cost saved per cache tier per §6.4.
**Owner.** observability-eng, sprint N+5.

### COST-CACHE-017 — Severity: Low
**Finding.** Cache-miss-reason categorization absent.
**Recommendation.** Categorize per §6.5; identify improvement levers.
**Owner.** AI platform, sprint N+5.

### COST-CACHE-018 — Severity: Low
**Finding.** Cache compliance audit not performed.
**Recommendation.** Cache encryption, access log, retention policy, tenant-offboarding deletion per §8.6.
**Owner.** security + AI platform, sprint N+6.

---

## 12. Adoption sequencing checklist

- [ ] **Enable provider-side prompt caching (§2).** Day-one config; immediate savings.
- [ ] **Audit calls for cacheability.** Sample; measure exact-match and semantic-match potential.
- [ ] **Implement response cache (§3).** Tenant-scoped keys; conservative TTL.
- [ ] **Implement retrieval cache (§5).** For RAG workloads.
- [ ] **Build hit-rate dashboards (§6).** Per tier.
- [ ] **Implement semantic cache (§4).** Workload-tuned threshold; sampled wrong-hit review.
- [ ] **Document invalidation triggers (§7).** Per cache; per workload.
- [ ] **Implement invalidation pipeline (§7.3, §7.4).** Document update propagation; version change handling.
- [ ] **Add cache-quality filter at write (§7.7).** Don't cache refusals / low-confidence.
- [ ] **Verify tenant_id in all cache keys (§8.1).** Lint + code review.
- [ ] **Per-tenant size limits (§8.4).** Tier-based.
- [ ] **Eviction policy per cache (§8.5).** Per-tenant eviction in multi-tenant.
- [ ] **Compliance audit (§8.6).** Encryption, access log, retention.
- [ ] **Pre-production test:** synthetic load; verify hit rates; verify no cross-tenant hits.
- [ ] **Monthly cache health review.** Hit rates, invalidations, wrong-hit incidents.

---

## 13. References

**In this folder.**
- [cost-attribution.md](./cost-attribution.md) — attribution tracks both cached and uncached calls.
- [cost-budget-circuit-breaker.md](./cost-budget-circuit-breaker.md) — caching reduces budget consumption.
- [cost-dashboards-and-alerts.md](./cost-dashboards-and-alerts.md) — cache hit rates surface here.
- [per-tenant-cost-control.md](./per-tenant-cost-control.md) — per-tenant cache isolation.
- [tier-routing-for-cost.md](./tier-routing-for-cost.md) — cache miss + tier routing combine for cost savings.
- [batch-vs-realtime-cost.md](./batch-vs-realtime-cost.md) *(companion)* — batch processing complements caching.
- [cost-aware-rate-limiting.md](./cost-aware-rate-limiting.md) *(companion)* — caching reduces rate-limit pressure.

**Elsewhere in this repo.**
- [rag-engineering/retrieval-engineering.md](../rag-engineering/retrieval-engineering.md) — retrieval patterns that compose with retrieval cache.
- [rag-engineering/query-rewriting.md](../rag-engineering/query-rewriting.md) — query normalization improves cache hit rate.
- [observability-and-telemetry/cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md) — broader cost dashboards include cache effectiveness.

**Sibling repos.**
- [ai-architecture-reference-architecture / multi-tenancy-and-isolation / cross-tenant-leakage-prevention.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/cross-tenant-leakage-prevention.md) — cache isolation as part of broader multi-tenant controls.
- [ai-architecture-reference-architecture / integration-architecture / backpressure-and-queueing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/integration-architecture/backpressure-and-queueing.md) — caching reduces upstream pressure.

**External.**
- Anthropic prompt caching documentation.
- OpenAI prompt caching documentation.
- Google Vertex AI context caching documentation.
- Redis caching patterns documentation.
- Semantic caching research (e.g., GPTCache, LangChain semantic cache).
- Cache invalidation literature ("two hard things in CS" — Phil Karlton's well-known quip applies).
