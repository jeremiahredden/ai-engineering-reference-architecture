# Query Rewriting

> **Audience.** Engineers adding or refactoring query rewriting in a RAG pipeline. Tech leads who have seen "conversational follow-ups perform much worse than first turns" and want a structured fix. **Scope.** The *engineering* practice of query rewriting — expansion, decomposition, context-aware rewriting, LLM-as-rewriter patterns, calibration, integration with retrieval. Pair with [retrieval-engineering.md](./retrieval-engineering.md), [eval-of-rag.md](../eval-engineering/eval-of-rag.md). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Query rewriting addresses a specific class of RAG failure: the user's literal query is not the optimal query for retrieval. The user says "what about pediatric patients?" — a clarifying follow-up to a prior turn — but the retrieval embeds those four words and returns content that has nothing to do with the original topic. The query needed rewriting; without it, recall on second-turn-and-later conversational queries collapses.

The discipline this document codifies: query rewriting is a deliberate engineering choice with measurable lift. Some workloads benefit substantially (conversational, multi-hop, shorthand-heavy); some don't (single-shot well-formed queries). The team measures and decides; the rewriter is itself a versioned, eval-validated artifact.

This document is opinionated about three things:

1. **Query rewriting is not always justified.** Single-turn well-formed queries don't need it. Adding a rewriter to those workloads adds latency and cost without quality lift.
2. **Conditional rewriting is the production default.** Rewrite on turn 2+; skip on turn 1. Rewrite on shorthand; skip on full queries. Per-class dispatch reduces cost.
3. **The rewriter prompt is itself code.** Versioned, eval-validated, monitored for drift. The same prompt-engineering discipline as the production system's primary prompts.

Structure: (2) the rewriting patterns; (3) when each applies; (4) implementation patterns; (5) calibration; (6) the rewriter as a prompt artifact; (7) integration with the retrieval pipeline; (8) cost / latency / quality; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The rewriting patterns

Several patterns; each fits a different problem.

### 2.1 Query expansion

Add synonyms, related terms, or alternative phrasings to the original query.

```
Original: "warfarin dosing for elderly patients"
Expanded: "warfarin coumadin anticoagulant dosing dose dosage elderly older geriatric patients"
```

**When right.** Lexical-mismatch workloads — the user's vocabulary doesn't match the corpus vocabulary; expansion increases the chance of a match.

**What it costs.** Adds an LLM call (or rule-based expansion); the expanded query is longer; the retrieval may have to handle the longer query.

### 2.2 Query decomposition (multi-hop)

Split a complex query into sub-queries; retrieve per sub-query; aggregate.

```
Original: "What is the interaction between drug A and drug B in patients with renal impairment?"
Decomposed: 
  - "drug A renal impairment dosing"
  - "drug B renal impairment dosing"
  - "drug A drug B interaction"
```

**When right.** Multi-hop questions where the answer requires combining information from multiple distinct sources.

**What it costs.** Multiple retrievals per query; the answer-aggregation logic; latency adds up.

### 2.3 Context-aware rewriting (conversational)

Rewrite the query using prior-turn context to produce a self-contained query.

```
Turn 1: "Tell me about warfarin dosing"
Turn 2 (original): "What about elderly patients?"
Turn 2 (rewritten): "What is warfarin dosing for elderly patients?"
```

**When right.** Conversational workloads where follow-up questions depend on prior turns.

**What it costs.** Adds an LLM call per turn 2+; the rewrite quality depends on the prompt and the model.

### 2.4 LLM-as-rewriter (general)

A general LLM prompt that takes the user query (and optionally context) and produces a rewritten query.

```
LLM prompt:
"Rewrite the user's query to be optimal for retrieval against a clinical-content corpus.
Add medical terminology where the user used colloquial terms.
Make it self-contained (don't rely on context the retriever won't see).
Return the rewritten query."
```

**When right.** When the rewriting logic is complex and benefits from a generative model.

**What it costs.** LLM-call cost per query; latency per rewrite.

### 2.5 Rule-based rewriting

Hard-coded rules (lookup tables, pattern matching, regex).

```
If query contains "ICD-10 code [A-Z][0-9]{2}": preserve the code verbatim.
If query is a single drug name: prepend "drug information about ".
```

**When right.** Workloads with a small set of well-understood query patterns.

**What it costs.** Maintenance of the rule set; new patterns require code changes.

### 2.6 The pattern choice

| Workload | Recommended pattern |
|---|---|
| Single-shot well-formed queries | None (skip rewriting) |
| Conversational multi-turn | Context-aware rewriting |
| Multi-hop questions common | Decomposition |
| Vocabulary mismatch between users and corpus | Expansion (often LLM-based) |
| Workload with stable query patterns | Rule-based |
| Heterogeneous workload | LLM-as-rewriter with prompt covering multiple patterns |

Most production workloads use one or two patterns selectively (conditional on the query); few use rewriting on every query.

---

## 3. When each applies

The rewriting decision is per-query.

### 3.1 The decision criteria

Per query, the pipeline decides: rewrite or skip?

- **Turn position.** Turn 1: usually no rewrite (the query is self-contained). Turn 2+: usually rewrite (conversational context likely).
- **Query length.** Very short (1-3 words): often shorthand; rewrite likely helps. Long (10+ words): usually self-contained; skip.
- **Query class.** Multi-hop questions: decompose. Simple lookups: skip.
- **Production traffic class.** A classifier (per [tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md)) may identify the query class and dispatch.

### 3.2 The conditional dispatch

```python
def maybe_rewrite(query, context):
    if should_rewrite(query, context):
        return rewrite(query, context)
    return query

def should_rewrite(query, context):
    if not context.previous_turns:
        # First turn; usually skip
        return False
    if len(query.split()) < 4:
        # Short query; likely shorthand; rewrite
        return True
    if context.classifier_indicates_multihop:
        return True
    return False
```

The conditional logic is workload-specific; calibrated per section 5.

### 3.3 The eval validation

Per [eval-of-rag.md](../eval-engineering/eval-of-rag.md):

- Eval cases include conversational subsets, multi-hop subsets.
- Measure retrieval recall with vs without rewriting on each subset.
- Adopt rewriting for subsets where it lifts recall; skip for subsets where it doesn't.

### 3.4 The lift expectations

Typical lifts on workloads that benefit:

- Conversational follow-ups: 15-25 percentage points recall lift on turn 2+.
- Multi-hop questions: 10-20 points lift via decomposition.
- Vocabulary mismatch: 5-10 points via expansion.

Workloads without these issues: 0-2 points lift; not worth the cost.

---

## 4. Implementation patterns

The engineering for each pattern.

### 4.1 LLM-as-rewriter implementation

The standard pattern:

```python
async def llm_rewrite(query: str, context: ConversationContext) -> str:
    messages = build_rewriter_prompt(query, context)
    response = await llm_client.call(
        provider="anthropic",
        model="claude-haiku-4-5",
        prompt_version="conversational_rewriter@1.2.0",
        messages=messages,
        max_tokens=100,
        context=current_call_context(),
    )
    return parse_rewrite(response.content)
```

The rewriter uses a cheap-tier model (Haiku-class typically); the rewrite is a structured-output task.

### 4.2 The rewriter prompt

A typical rewriter prompt:

```
You rewrite user queries to be optimal for retrieval against {corpus_description}.

Given the user's query and the prior conversation context, produce a rewritten
query that:
- Is self-contained (doesn't rely on context the retriever won't see).
- Uses appropriate domain vocabulary.
- Preserves the user's intent.
- Is concise (under 30 words).

PRIOR CONVERSATION:
{previous_turns}

USER QUERY:
{query}

Return only the rewritten query (no preamble, no explanation).
```

The prompt is versioned per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md); changes go through eval validation.

### 4.3 The decomposition implementation

For decomposition:

```python
async def decompose(query: str) -> list[str]:
    messages = build_decomposer_prompt(query)
    response = await llm_client.call(
        provider="anthropic",
        model="claude-haiku-4-5",
        prompt_version="multihop_decomposer@1.0.0",
        messages=messages,
        max_tokens=200,
        context=current_call_context(),
    )
    sub_queries = parse_decomposition(response.content)
    return sub_queries
```

The decomposer returns a structured list of sub-queries; each is then retrieved separately; results are aggregated.

### 4.4 The expansion implementation

Simple LLM-based expansion:

```python
async def expand(query: str) -> str:
    messages = build_expansion_prompt(query)
    response = await llm_client.call(...)
    return f"{query} {response.content}"  # original + expansion
```

Or rule-based:

```python
def rule_based_expand(query: str) -> str:
    expanded = query
    for term, synonyms in SYNONYM_DICT.items():
        if term in query.lower():
            expanded += " " + " ".join(synonyms)
    return expanded
```

### 4.5 The rewrite failure handling

If the rewriter fails (LLM error, parse failure, timeout):

- Fall back to the original query (don't fail the retrieval).
- Log the rewrite failure for investigation.
- The retrieval proceeds with the unmodified query.

The failure mode is graceful: rewrite is an enhancement, not a requirement.

### 4.6 The latency budget

Rewriter latency is part of the retrieval pipeline budget:

- LLM rewriter on Haiku-tier: ~200-500ms.
- Rule-based rewriter: ~5ms.
- The pipeline's total latency budget per [retrieval-engineering.md](./retrieval-engineering.md) section 2.2 accommodates.

For tight-latency workloads (< 500ms total), LLM rewriting may not fit; rule-based or skip.

---

## 5. Calibration

The rewriter's effectiveness is measured and tuned.

### 5.1 The lift measurement

Per [eval-of-rag.md](../eval-engineering/eval-of-rag.md):

- Run eval cases with rewrite vs without.
- Compare retrieval recall, precision, MRR.
- Per case class: rewrite lift on conversational cases, multi-hop cases, simple-lookup cases.

The per-class lift informs the conditional dispatch (per section 3.2).

### 5.2 The rewriter prompt iteration

The rewriter prompt is iterated based on eval:

- Identify cases where rewriting failed to lift quality.
- Read the rewrite output; identify why it didn't help.
- Update the prompt to address the pattern.
- Re-eval; verify the lift improvement.

Typical iteration: 2-4 prompt revisions per rewriter to stabilize at production quality.

### 5.3 The conditional-skip calibration

The skip thresholds (turn position, query length, query class) are calibrated:

- For each threshold candidate: measure cost (% of queries that rewrite) and quality (recall on rewritten cases).
- Pick thresholds that produce the best cost/quality joint.

### 5.4 The drift detection

Rewriter quality can drift:

- Workload composition changes; the rewriter encounters new patterns.
- Underlying LLM model updates; rewrite quality shifts.
- Corpus changes; the optimal rewrite for the same query may shift.

Quarterly re-eval per section 5.1; drift detected → prompt iteration or skip-threshold recalibration.

### 5.5 The over-rewrite anti-pattern

A rewriter that's too aggressive can hurt:

- The rewrite hallucinates terms not in the original query.
- The rewrite changes the user's intent.
- The rewrite is verbose; the embedding represents the verbose rewrite poorly.

Calibration includes detecting over-rewrite (eval cases where rewriting causes regressions).

---

## 6. The rewriter as a prompt artifact

The rewriter is a prompt; it follows the prompt-engineering discipline.

### 6.1 The prompt versioning

Per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) and [prompt-versioning.md](../prompt-engineering/prompt-versioning.md):

- The rewriter prompt is a versioned artifact (`conversational_rewriter@1.2.0`).
- Changes go through PR review; eval validation; release-pinning.
- Rollback is straightforward (point back at prior version).

### 6.2 The prompt ownership

The rewriter prompt has an owner team (typically ai-platform-eng). The owner:

- Reviews PRs touching the rewriter.
- Responds to rewriter-related quality issues.
- Schedules quarterly recalibration.

### 6.3 The prompt observability

Each rewrite is a trace span per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md):

- `ai.llm.prompt.version`: which rewriter version was used.
- `ai.retrieval.query_rewrite_original`: the original query.
- `ai.retrieval.query_rewrite_result`: the rewritten query.
- `ai.retrieval.query_rewrite_strategy`: which pattern (expansion, decomposition, etc.).

The trace supports diagnostic investigation.

### 6.4 The prompt-as-API contract

The rewriter has a contract:

- Input shape: query string + conversation context.
- Output shape: rewritten query string (or list of sub-queries for decomposition).
- Latency: < target.

Changes to the contract are versioned; consumers know what to expect.

---

## 7. Integration with the retrieval pipeline

The rewriter sits in the retrieval pipeline.

### 7.1 The placement

Per [retrieval-engineering.md](./retrieval-engineering.md) section 2.1:

```
User query
    │
    ▼
[Query embedding]
    │
    ▼
[Query rewriting] ← this stage
    │
    ▼
[Parallel retrieval] (BM25 + vector)
    │
    ▼
...
```

Rewriting happens after embedding (the embedder embeds the original or rewritten? — see section 7.3) and before retrieval.

### 7.2 The decomposition complication

When the rewriter decomposes the query into N sub-queries:

- Each sub-query is retrieved separately.
- Results are aggregated (RRF across sub-query result sets, or simple concatenation with deduplication).
- The aggregated set is reranked.

The pipeline accommodates the fan-out; latency adds up (N retrievals instead of 1).

### 7.3 The embed-original vs embed-rewritten question

For vector retrieval, the embedded query matters:

- **Embed original.** Vector search uses the original embedding; rewrite affects only BM25 / other retrievers.
- **Embed rewritten.** Vector search uses the rewritten embedding.

Either pattern is valid; the choice is workload-specific. Most teams embed the rewritten query (the rewrite is intended to improve retrieval; using it for vector aligns).

### 7.4 The wrapper integration

The retrieval-client wrapper (per [retrieval-scope-enforcement.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/retrieval-scope-enforcement.md)) invokes the rewriter as part of the retrieval call:

```python
def retrieve(self, query, context, ...):
    if self.should_rewrite(query, context):
        rewritten = self.rewriter.rewrite(query, context)
        # Continue with rewritten query
    else:
        rewritten = query
    # ... rest of retrieval pipeline
```

The rewriter is internal to the wrapper; calling code doesn't see it directly.

### 7.5 The observability integration

The rewrite step is its own trace sub-span (per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md)). The trace shows:

- Whether rewriting was applied.
- Which strategy.
- The original and rewritten queries.
- The rewrite latency.

Per-query investigation distinguishes "retrieval failed because of bad rewrite" from "retrieval failed because of bad retriever."

---

## 8. Cost / latency / quality

The three-dimensional optimization.

### 8.1 The cost dimension

Rewriter cost per query:

- LLM rewriter on Haiku-tier: ~$0.0005-0.001 per rewrite.
- Rule-based rewriter: free.
- LLM rewriter on Sonnet-tier or Opus-tier: more expensive; rarely justified.

For high-volume workloads with conditional rewrite (50% of queries rewrite): cost is ~$0.0003 amortized per query.

### 8.2 The latency dimension

- LLM rewriter latency: 200-500ms.
- Rule-based: 5ms.

The rewriter is the first stage in retrieval; its latency is added to total retrieval latency.

### 8.3 The quality dimension

The retrieval recall lift (or downstream answer quality) per section 5.1.

### 8.4 The joint optimization

The team's choice optimizes the joint:

- High-conversational workload: rewriting is essential; cost and latency justified.
- High-volume, single-shot: skip rewriting; the cost/latency isn't worth marginal lift.
- Mixed workload: conditional dispatch; rewriting for the cases that benefit.

### 8.5 The rule-based vs LLM trade-off

Rule-based:
- Free; instant.
- Limited to patterns hard-coded.
- Maintenance overhead as new patterns appear.

LLM:
- Flexible; handles new patterns.
- Costs; latency.

Many workloads use a hybrid: rule-based for the common cases (instant); LLM for the rest (flexible).

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The rewriting pattern

Meridian's Care Coordinator uses context-aware LLM-based rewriting for turn 2+ conversational queries.

- First turn queries: no rewriting (queries are well-formed clinical questions from the chat panel).
- Turn 2+: LLM rewriter on Haiku-tier converts the follow-up into a self-contained query.

### 9.2 The rewriter prompt

```
You rewrite a clinical follow-up question into a self-contained query for
retrieval against the clinical-guidelines corpus.

Use the prior conversation turn to fill in context. Use clinical terminology
where appropriate. Keep the rewrite concise.

PRIOR TURN:
{previous_turn}

USER FOLLOW-UP:
{query}

Return only the rewritten query.
```

Prompt version: `conversational_rewriter@1.2.0`. Pinned per release.

### 9.3 The dispatch

The conditional logic:

```python
def should_rewrite(query, context):
    if not context.previous_turns:
        return False  # turn 1
    if context.classifier_indicates_complete_query:
        return False  # already self-contained
    return True
```

The classifier (one of the Haiku classifier worker's outputs) flags queries that are already self-contained even on turn 2+.

In production: ~38% of all queries get rewritten (turn 2+, not already self-contained).

### 9.4 The eval lift

Eval measurement on the conversational subset:

- Without rewriting: recall = 64%.
- With rewriting: recall = 86%.
- Lift: 22 percentage points.

The lift justifies the cost; rewriting is adopted.

### 9.5 The cost / latency

- Rewriter call: ~$0.0006 per call (Haiku, ~100 output tokens).
- Daily volume: ~6K interactions × 38% rewrite = ~2,280 rewrite calls.
- Daily rewriter cost: ~$1.40.
- Latency added: ~280ms p95 to retrieval (only on rewriting queries).

The cost is small; the latency is within the broader retrieval budget.

### 9.6 The drift management

Quarterly: eval re-run; lift on conversational subset measured; prompt iterated if needed.

Recent revisions:
- v1.0.0 (initial): 18-point lift.
- v1.1.0 (added clinical-terminology guidance): 20-point lift.
- v1.2.0 (improved short-followup handling): 22-point lift.

### 9.7 The decomposition consideration

Multi-hop questions in clinical workload are rare (~3% of traffic). The team evaluated decomposition:

- Decomposition lift on multi-hop subset: 12 points.
- Decomposition cost: 2-3 LLM calls + multiple retrievals per query.
- Decision: not yet adopted; the lift on 3% of traffic doesn't justify the engineering investment.

Revisit if multi-hop traffic grows.

### 9.8 The failure handling

Rewriter failures (~0.1% of attempts):
- LLM timeout: fall back to original query.
- Parse failure: fall back to original.
- Failures logged; investigated weekly.

The fallback pattern means rewriter failures don't fail retrieval.

### 9.9 The platform discipline

- Rewriter prompt versioned per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md).
- Conditional dispatch eval-validated.
- Quarterly recalibration scheduled.
- Trace observability records rewrite per query.

---

## 10. Anti-patterns

### 10.1 "Rewrite every query"

Rewriting on every query; cost and latency added to queries that don't benefit.

**Corrective.** Conditional dispatch per section 3.2.

### 10.2 "No rewriting for conversational workload"

Conversational queries handled as-is; recall on turn 2+ collapses.

**Corrective.** Context-aware rewriting per section 2.3.

### 10.3 "Rewriter prompt not versioned"

Rewriter prompt is a string in code; changes ship without eval validation.

**Corrective.** Versioned prompt per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md).

### 10.4 "No rewrite-lift measurement"

Rewriter added; lift never measured; cost paid without verified benefit.

**Corrective.** Eval-measured lift per section 5.1.

### 10.5 "Rewriter on expensive tier"

Rewriting uses an Opus-tier LLM; cost dominates the rewriter ROI.

**Corrective.** Haiku-tier rewriter typically sufficient; eval-validated.

### 10.6 "Rewriter fails the retrieval on failure"

When the rewriter fails, the whole retrieval call fails; the user sees an error.

**Corrective.** Fall back to original query on rewriter failure per section 4.5.

### 10.7 "Over-rewrite — hallucinated context"

Rewriter adds context the user didn't intend; retrieval pivots to the hallucinated direction; user gets a wrong answer.

**Corrective.** Calibration includes detecting over-rewrite; prompt iterated to be conservative.

### 10.8 "Decomposition without aggregation"

Decomposition produces sub-queries; sub-queries are retrieved; the aggregation strategy is "concat with no dedup"; the context is messy.

**Corrective.** RRF or score-normalized aggregation across sub-queries; deduplication.

---

## 11. Findings (sprint-assignable)

### QREWRITE-001 — Severity: High
**Finding.** Conversational follow-up retrieval recall is low; no query rewriting in place.
**Recommendation.** Context-aware rewriting per section 2.3; conditional on turn 2+.
**Owner.** ai-platform-eng, sprint N+1.

### QREWRITE-002 — Severity: High
**Finding.** Rewriting applied universally; cost and latency added to queries that don't benefit.
**Recommendation.** Conditional dispatch per section 3.2.
**Owner.** ai-platform-eng, sprint N+2.

### QREWRITE-003 — Severity: High
**Finding.** Rewriter lift not measured; the rewriter's value is unknown.
**Recommendation.** Eval measurement per section 5.1; adopt or remove based on measured lift.
**Owner.** ai-platform-eng, sprint N+2.

### QREWRITE-004 — Severity: High
**Finding.** Rewriter prompt is inline string; not versioned.
**Recommendation.** Versioned prompt per [prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md).
**Owner.** ai-platform-eng, sprint N+2.

### QREWRITE-005 — Severity: High
**Finding.** Rewriter failure fails the retrieval; user sees an error.
**Recommendation.** Fall back to original query per section 4.5.
**Owner.** ai-platform-eng, sprint N+2.

### QREWRITE-006 — Severity: High
**Finding.** Rewriter uses expensive tier (Sonnet or Opus); cost not justified.
**Recommendation.** Move to Haiku-tier per section 4.1; eval-validate.
**Owner.** ai-platform-eng + finops, sprint N+2.

### QREWRITE-007 — Severity: Medium
**Finding.** Rewriter prompt not eval-validated; changes ship without verification.
**Recommendation.** Eval gate per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md).
**Owner.** ai-platform-eng, sprint N+3.

### QREWRITE-008 — Severity: Medium
**Finding.** Over-rewrite detection absent; hallucinated context may regress recall.
**Recommendation.** Eval cases for over-rewrite per section 5.5.
**Owner.** ai-platform-eng, sprint N+3.

### QREWRITE-009 — Severity: Medium
**Finding.** Decomposition for multi-hop queries not implemented; multi-hop recall is bounded.
**Recommendation.** Evaluate decomposition per section 2.2; adopt if lift justifies.
**Owner.** ai-platform-eng, sprint N+3.

### QREWRITE-010 — Severity: Medium
**Finding.** Decomposition aggregation logic uses naive concat; context is messy.
**Recommendation.** RRF aggregation per section 7.2.
**Owner.** ai-platform-eng, sprint N+3.

### QREWRITE-011 — Severity: Medium
**Finding.** Rewriter observability absent; per-query rewrite trace invisible.
**Recommendation.** Trace attributes per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) section 3.2.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### QREWRITE-012 — Severity: Medium
**Finding.** Conditional-skip thresholds uncalibrated; rewriting on or off too aggressively.
**Recommendation.** Calibration per section 5.3.
**Owner.** ai-platform-eng, sprint N+3.

### QREWRITE-013 — Severity: Medium
**Finding.** Embed-original vs embed-rewritten not decided deliberately.
**Recommendation.** Eval-validate both per section 7.3; pick based on measurement.
**Owner.** ai-platform-eng, sprint N+4.

### QREWRITE-014 — Severity: Medium
**Finding.** Rewriter recalibration not scheduled; drift may accumulate.
**Recommendation.** Quarterly recalibration per section 5.4.
**Owner.** ai-platform-eng, sprint N+4.

### QREWRITE-015 — Severity: Medium
**Finding.** Rule-based pattern not used for high-frequency simple rewrites; LLM cost incurred unnecessarily.
**Recommendation.** Hybrid rule-based + LLM per section 8.5.
**Owner.** ai-platform-eng, sprint N+4.

### QREWRITE-016 — Severity: Low
**Finding.** Rewriter prompt does not include corpus context; rewrites lack domain vocabulary.
**Recommendation.** Include `{corpus_description}` in prompt per section 4.2.
**Owner.** ai-platform-eng, sprint N+5.

### QREWRITE-017 — Severity: Low
**Finding.** Rewriter latency exceeds budget for low-latency workloads.
**Recommendation.** Rule-based or skip for low-latency paths per section 4.6.
**Owner.** ai-platform-eng, sprint N+5.

### QREWRITE-018 — Severity: Low
**Finding.** Documentation thin; new engineers don't understand the rewriter's role.
**Recommendation.** Documentation alongside the rewriter.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team adding query rewriting:

- [ ] **Sprint 0 — design.** Identify the workload pattern needing rewriting (conversational, multi-hop, vocabulary). Choose rewriting pattern(s).
- [ ] **Sprint 1 — first rewriter.** Implement context-aware rewriting (most common case); LLM-based on Haiku-tier.
- [ ] **Sprint 1 — prompt as artifact.** Versioned prompt per prompts-as-code.
- [ ] **Sprint 1 — fallback.** Graceful fallback to original query on rewriter failure.
- [ ] **Sprint 2 — eval validation.** Measure lift per section 5.1; adopt or revise.
- [ ] **Sprint 2 — conditional dispatch.** Skip rewriting for queries that don't benefit (turn 1, simple lookups).
- [ ] **Sprint 3 — observability.** Trace attributes; per-query rewrite visible.
- [ ] **Sprint 3 — eval gate.** Rewriter changes go through eval gate.
- [ ] **Sprint 4 — decomposition.** If multi-hop is a meaningful workload share, add decomposition.
- [ ] **Sprint 5 — calibration cadence.** Quarterly recalibration; over-rewrite detection.
- [ ] **Ongoing — discipline.** Rewriter as a versioned, eval-validated artifact.

A team that completes this sequence has query rewriting that lifts retrieval on the cases that need it. A team that ships universal rewriting pays cost without benefit on the cases that don't.

---

## 13. References

- This repo: [rag-engineering/retrieval-engineering.md](./retrieval-engineering.md) — where the rewriter sits in the pipeline.
- This repo: [rag-engineering/rag-failure-modes-and-debugging.md](./rag-failure-modes-and-debugging.md) — diagnostic patterns for rewriting issues.
- This repo: [eval-engineering/eval-of-rag.md](../eval-engineering/eval-of-rag.md) — eval discipline for measuring lift.
- This repo: [prompt-engineering/prompts-as-code-discipline.md](../prompt-engineering/prompts-as-code-discipline.md) — rewriter prompt as an artifact.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — query-rewrite attributes.
- This repo: [cost-and-finops/tier-routing-for-cost.md](../cost-and-finops/tier-routing-for-cost.md) — classifier dispatch.
- Sibling repo: [ai-architecture-reference-architecture/context-and-prompt-architecture/long-context-vs-rag.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/tree/main/context-and-prompt-architecture) (coming) — the architectural context.
- HyDE (Hypothetical Document Embeddings) — alternative pattern for query rewriting.
- LangChain query-construction documentation; LlamaIndex query-transform documentation.
