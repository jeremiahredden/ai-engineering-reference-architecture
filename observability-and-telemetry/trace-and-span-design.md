# Trace and Span Design for AI Systems

> **Audience.** Engineers building or refactoring observability for an AI feature. Tech leads who have been asked "why did the model say that?" and could not answer from the existing logs. **Scope.** The *engineering* foundation for AI-aware traces and spans — the structure that the per-call (`llm-call-instrumentation.md`) and per-agent-step (`agent-step-instrumentation.md`) docs apply. Not architectural placement of guardrails (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `guardrails-and-policy-architecture/` owns that). **Worked client.** Meridian Health.

---

## 1. Why this document exists

The single most common operational failure in production AI systems is *opacity*. The chat panel returned an answer that was wrong, or weird, or off-topic, and the team cannot reconstruct why. Logs show the request and the response; everything in between — the prompt that was actually built, the retrieval that ran, the chunks that came back, the tools the agent called, the budget state at each turn — is missing. Debugging becomes "rerun the same input and see what happens," which is unreliable for nondeterministic systems and impossible for past incidents whose state has since changed.

Traditional distributed-tracing solves this exact problem for HTTP services: every request gets a trace; the trace contains spans for each downstream call; spans carry attributes that explain what happened. The pattern adapts to AI systems with one substantive change: the spans and attributes are different because the operations are different. There is no canonical "Postgres query" span for an LLM call; there is no built-in OpenTelemetry semantic convention for "the retrieval client returned these doc IDs with these scores." Teams that adopt OpenTelemetry for AI without engineering the AI-specific span design end up with traces that capture latency but not the information needed to debug AI behavior.

This document is opinionated about three things:

1. **One trace per top-level user-visible interaction.** Not per HTTP request — the abstraction that matters for AI debugging is the user's interaction, which may span streaming responses, asynchronous callbacks, and human-in-the-loop hops.
2. **Span hierarchy mirrors the AI architecture.** A supervisor / worker agent has supervisor spans containing worker spans containing tool-call spans containing retrieval spans. A workflow has stage spans with embedded LLM-call spans. The span tree is the architecture, traced.
3. **Attributes capture what the system actually did, not just timings.** Latency-only attributes do not help debug AI behavior. The attributes that matter are *prompt version, model version, retrieved doc IDs, tool decisions, budget state at decision time, judge scores when scored, cost incurred*. These are the AI-specific equivalents of what request-method and status-code are to HTTP tracing.

Structure: (2) trace boundaries; (3) span hierarchy; (4) attribute taxonomy; (5) sampling strategy; (6) the OpenTelemetry-for-LLMs convention emerging in 2026; (7) sensitive-data handling in traces; (8) vendor tooling integration; (9) worked Meridian Care Coordinator example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. Trace boundaries

The first design decision is what defines a trace. The wrong answer is "one trace per HTTP request" — that produces traces that miss the boundaries that matter for AI debugging.

### 2.1 The interaction as the unit

A trace covers one *user-visible interaction*. For Meridian's chat panel: one chat turn (user types, system responds, regardless of how many model calls / retrievals / tool calls happen inside). For the async coordination task: one task end-to-end (the outer task, with per-patient sub-tasks as nested spans). For the side-effect path: one side-effect (the agent's draft creation through the human approval and the realized outcome).

The interaction is what the user perceives; the trace is what reconstructs it.

### 2.2 Trace ID propagation

The trace ID is generated at the point of user interaction (the chat panel, the API endpoint, the task initiation) and propagated downstream into every internal call. Standard W3C Trace Context propagation works as a starting point; AI-specific propagation adds the interaction-ID dimension (which can be the trace-ID itself, or a stable identifier the application carries alongside).

For Meridian: the trace ID propagates from the chat panel through the AI gateway, through the supervisor, through each worker, through each tool call, through each retrieval. Every span carries the same root trace ID; the parent-span ID gives the hierarchy.

### 2.3 Trace spans that cross async boundaries

The async coordination task starts as one trace (the user's "kick off this task" interaction) and produces per-patient sub-tasks that run asynchronously. The pattern:

- The outer trace records the task initiation and the fan-out decision.
- Each per-patient sub-task gets its own trace ID *for the per-patient work*, with an explicit parent-trace link back to the originating outer trace.
- The webhook completion event is recorded in both the per-patient trace (as the completion span) and the outer trace (as the aggregated completion).

This lets per-patient incidents be investigated in isolation (one patient's per-patient trace) while preserving the outer-task view for fleet-level questions.

### 2.4 Trace spans that cross human-in-the-loop boundaries

The chat panel side-effect path crosses an HITL boundary: the agent creates a draft, the human reviews (minutes to hours later), the human approves or rejects. The pattern:

- The original agent trace captures the draft creation as a span.
- The human-review event is captured as an event on the original trace (using OpenTelemetry events, which can attach to spans without requiring the span to remain open).
- The eventual approval / rejection / send action is similarly an event.
- For long-lived HITL flows (>24 hours), the original trace's spans are finalized; the human-action event creates a follow-up span that links back to the original trace via parent-span-ID.

The result: a complete lineage from agent action to realized outcome is queryable as one trace tree, even when the events span hours or days.

---

## 3. Span hierarchy

A well-designed span hierarchy makes the trace readable top-down — the reader follows what the system did at each layer of decomposition. The hierarchy mirrors the AI architecture.

### 3.1 Workflow with embedded LLM steps

```
trace: interaction
└── span: workflow:patient_question
    ├── span: classify (LLM call)
    ├── span: retrieve
    │   ├── span: retrieval:bm25
    │   └── span: retrieval:vector
    ├── span: answer (LLM call)
    └── span: format
```

Each stage is a span; LLM calls are spans; retrievals are spans. The hierarchy reads as "the workflow's `answer` step is what failed; the LLM call within it took 3.2s." This is the simplest hierarchy; everything else builds on it.

### 3.2 Single-agent loop (ReAct)

```
trace: interaction
└── span: agent:reAct
    ├── span: turn:1
    │   ├── span: llm_call
    │   └── span: tool_call:retrieve_drug_interactions
    ├── span: turn:2
    │   ├── span: llm_call
    │   └── span: tool_call:retrieve_clinical_guideline
    └── span: turn:3
        └── span: llm_call (final answer, no tool call)
```

Each turn is a span; per-turn LLM and tool calls are sub-spans. The trace reveals the loop's progression and where it terminated (the third turn had no tool call → final answer).

### 3.3 Supervisor / worker

```
trace: interaction
└── span: supervisor
    ├── span: worker:classifier
    │   └── span: llm_call
    ├── span: worker:clinical_knowledge
    │   ├── span: llm_call
    │   ├── span: tool_call:retrieve
    │   │   ├── span: retrieval:bm25
    │   │   ├── span: retrieval:vector
    │   │   └── span: rerank
    │   └── span: tool_call:drug_interaction_graph
    ├── span: worker:drafting
    │   └── span: llm_call
    └── span: supervisor:consolidation
        └── span: llm_call
```

The Meridian Care Coordinator's trace shape. The supervisor span contains worker spans; each worker contains its LLM call(s) and any tool calls; the supervisor's consolidation is a final span. The hierarchy is the architecture, made visible.

### 3.4 Async coordination

```
trace (outer): interaction
└── span: task:bulk_outreach
    ├── span: estimate_fan_out
    ├── span: orchestration:step_functions
    │   ├── (event: per-patient task started: trace-id=t-p1)
    │   ├── (event: per-patient task started: trace-id=t-p2)
    │   └── ... (14 events)
    └── (event: task completed: 14/14 patients processed)

trace (per-patient): t-p1
├── (event: parent-trace: outer trace ID)
└── span: per_patient_coordination
    └── ... (full supervisor / worker hierarchy as in 3.3)
```

The outer trace records the orchestration; per-patient traces record the per-patient work; the events link them.

### 3.5 Naming conventions

Span names follow `category:operation` format:

- `workflow:patient_question`, `workflow:coordination_task`
- `agent:reAct`, `agent:supervisor_worker`, `agent:plan_then_execute`
- `worker:classifier`, `worker:clinical_knowledge`, `worker:drafting`
- `turn:N` (with N as the turn number)
- `llm_call` (with model, provider, prompt_version in attributes)
- `tool_call:tool_name`
- `retrieval:retriever_name`
- `rerank`
- `embed`
- `cache_lookup`, `cache_write`

Consistent naming makes traces queryable across the platform — "show me all `worker:clinical_knowledge` spans for tenant X this hour" works because the name is standardized.

---

## 4. Attribute taxonomy

Span attributes are where the AI-specific debugging information lives. The attribute set is deliberate; arbitrary key/value pairs do not make traces readable.

### 4.1 Universal attributes (every AI span)

| Attribute | Meaning | Example |
|---|---|---|
| `ai.trace.interaction_id` | The user-visible interaction ID (often == trace ID) | `interaction-2026-05-25-a7b8` |
| `ai.tenant.id` | Tenant context | `mercy-cleveland` |
| `ai.feature.id` | Which AI feature is this part of | `care-coordinator` |
| `ai.user.id` | The acting user (when applicable) | `u-mercy-cleveland-rn-001234` |
| `ai.user.role` | The user's role | `rn` |
| `ai.session.id` | Session context (when applicable) | `session-2026-05-25-a7b8-3` |
| `ai.cost.usd` | Cost incurred in this span | `0.012` |

These attributes attach to every span. They support aggregation, attribution, and per-tenant / per-feature analysis.

### 4.2 LLM-call span attributes

Detailed in [llm-call-instrumentation.md](./llm-call-instrumentation.md). Briefly:

| Attribute | Meaning |
|---|---|
| `ai.llm.provider` | `anthropic` / `openai` / `google` / `azure-openai` / etc. |
| `ai.llm.model` | The model identifier |
| `ai.llm.model_version` | The specific pinned version |
| `ai.llm.prompt_version` | The versioned prompt artifact ID |
| `ai.llm.tokens.input` | Input token count |
| `ai.llm.tokens.input_cached` | Cached portion of input |
| `ai.llm.tokens.output` | Output token count |
| `ai.llm.latency.ttft_ms` | Time to first token |
| `ai.llm.latency.total_ms` | Total latency |
| `ai.llm.finish_reason` | `stop` / `length` / `tool_calls` / `content_filter` / etc. |
| `ai.llm.has_tool_calls` | Boolean — did the response include tool calls |

### 4.3 Tool-call span attributes

| Attribute | Meaning |
|---|---|
| `ai.tool.name` | The tool name |
| `ai.tool.version` | The tool registration version |
| `ai.tool.class` | `read-only` / `draft-only` / `side-effect-with-HITL` / `side-effect-immediate` |
| `ai.tool.arguments_hash` | Hash of arguments (not the args themselves, for sensitive tools) |
| `ai.tool.authorization.decision` | `allowed` / `denied` |
| `ai.tool.authorization.reason` | If denied, the reason code |
| `ai.tool.success` | Boolean |
| `ai.tool.error_type` | When failed |
| `ai.tool.proposal_id` | For side-effect-with-HITL tools |
| `ai.tool.draft_id` | For draft-only tools |

### 4.4 Retrieval span attributes

| Attribute | Meaning |
|---|---|
| `ai.retrieval.query` | The query string (or hash for sensitive content) |
| `ai.retrieval.query_rewrite` | The rewritten query (if rewriting was used) |
| `ai.retrieval.corpus_id` | Which corpus was queried |
| `ai.retrieval.corpus_version` | The corpus version |
| `ai.retrieval.retriever_type` | `bm25` / `vector` / `hybrid` / `graph` |
| `ai.retrieval.embedding_model` | When using vector retrieval |
| `ai.retrieval.embedding_model_version` | Pinned version |
| `ai.retrieval.top_k` | How many results were requested |
| `ai.retrieval.returned_count` | How many results were returned |
| `ai.retrieval.doc_ids` | Array of returned doc IDs |
| `ai.retrieval.doc_scores` | Array of returned scores |
| `ai.retrieval.tenant_filter` | The tenant filter applied (defense-in-depth visibility) |
| `ai.retrieval.latency_ms` | Latency |

### 4.5 Agent loop / turn span attributes

| Attribute | Meaning |
|---|---|
| `ai.agent.topology` | `single` / `supervisor_worker` / `plan_execute` / etc. |
| `ai.agent.turn.number` | The turn number within the loop |
| `ai.agent.budget.turns_remaining` | Turn budget state |
| `ai.agent.budget.cost_remaining` | Cost budget state |
| `ai.agent.budget.time_remaining_ms` | Time budget state |
| `ai.agent.budget.tool_calls_remaining` | Tool-call budget state |
| `ai.agent.decision` | `tool_call` / `final_answer` / `escalate` / `terminate_budget` |

### 4.6 Quality and eval span attributes (when sampled for online judging)

| Attribute | Meaning |
|---|---|
| `ai.quality.judge.scored` | Boolean — was this interaction judged? |
| `ai.quality.judge.score` | Judge score (when scored) |
| `ai.quality.judge.criteria` | Which rubric criteria failed (when failed) |
| `ai.quality.judge.model_version` | The judge model's version |

### 4.7 Circuit-breaker span attributes (when a circuit tripped)

| Attribute | Meaning |
|---|---|
| `ai.circuit.tripped` | Boolean |
| `ai.circuit.layer` | `interaction` / `session` / `tenant` / `feature` |
| `ai.circuit.threshold` | The threshold that was crossed |
| `ai.circuit.actual` | The actual value at trip time |

### 4.8 The attribute discipline

- **Named consistently across the platform.** Every team that writes AI spans uses the same attribute names. The shared naming is what makes cross-feature queries possible.
- **Reasonable cardinality.** High-cardinality attributes (full prompt text, full query text) go in events or in a separate retrieval-content store, not as span attributes. The trace backend's cardinality limits matter.
- **Documented.** The attribute taxonomy is a versioned artifact; new attributes go through a small review; deprecated ones are retired through the normal deprecation flow.

---

## 5. Sampling strategy

A complete observability stack samples — but the sampling decision matters more for AI workloads than for traditional ones, because the high-value debugging cases are often the unusual ones that a random sample misses.

### 5.1 Sampling tiers

| Class | Sampling rate | Rationale |
|---|---|---|
| Clinical / regulated / high-stakes interactions | 100% | Audit posture requires full traces; volume is low enough to afford it |
| Standard production interactions | 10% random + tail-based augmentation | Random sample for trend analysis + tail-based catches errors and slow tails |
| Background / batch jobs | 100% of failures + 1% of successes | Failures need full debugging context; successes need only trend signal |
| Eval and test traffic | 100% | Cost-effective at the volume; full traces are training data for the eval review process |

### 5.2 Tail-based sampling for AI

Tail-based sampling means the decision to keep a trace is made *after* the trace completes, based on its characteristics. Useful AI-specific keep-criteria:

- **Failed circuit breakers.** Always kept; even a brief circuit trip indicates a real production event.
- **Quality judge failures.** When the sampled online judge marks an interaction as failed, the trace is upgraded to keep.
- **Errors and timeouts.** Standard SRE keep-criteria.
- **Long tail.** Top 5% by total latency or by cost. These are often the most-debug-worthy.
- **User feedback.** Thumbs-down responses upgrade their trace to keep.
- **HITL rejections.** When a human reviewer rejects a draft, the trace that produced the draft is kept (it is now a training case for the eval review).

The 10% random + tail-based augmentation pattern typically retains ~25% of total traces with the keep-criteria above. Storage cost is meaningfully lower than 100% sampling while debugging signal is preserved.

### 5.3 What sampling should NOT lose

- **Cost telemetry.** Even un-sampled traces contribute to cost aggregations. Cost is computed at-call and recorded in a separate aggregation store; sampling decisions affect trace storage but not cost tracking.
- **Audit-required events.** Authorization decisions for tool calls, HITL approvals, side-effect realizations. These are written to the dedicated audit sink regardless of trace sampling.
- **SLI / quality metrics.** Pre-aggregated metrics (judge-pass-rate, latency percentiles, error rate) come from a separate metric pipeline; sampling does not affect them.

The sampling decision is about *trace detail retention*; it is not about *signal availability*.

---

## 6. OpenTelemetry-for-LLMs convention

The OpenTelemetry community in 2025-2026 has been standardizing semantic conventions for LLM operations. The state of the convention is partial; teams should adopt the published convention where it covers their case and extend it consistently where it does not.

### 6.1 Published convention (as of 2026-Q1)

OpenTelemetry's `gen-ai` semantic convention covers the LLM-call span shape with attributes like:

- `gen_ai.system`: the provider
- `gen_ai.request.model`: the model name
- `gen_ai.request.temperature`, `gen_ai.request.top_p`, etc.: model parameters
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`: token counts
- `gen_ai.response.finish_reasons`: finish reason

When emitting LLM-call spans, use these names. Vendor tooling (LangSmith, Braintrust, Phoenix, Datadog LLM Observability) increasingly recognizes them.

### 6.2 Extensions for AI-specific concerns

The published convention does not (as of 2026-Q1) cover:

- Retrieval spans with retriever / corpus / doc-IDs / scores
- Agent loop / turn structure
- Tool authorization decisions
- Cost attribution attributes
- Quality / judge attributes

For these, extend with a consistent prefix (`ai.retrieval.*`, `ai.agent.*`, `ai.tool.*`, `ai.cost.*`, `ai.quality.*`). Use the `gen_ai.*` names where they overlap (every LLM call still uses `gen_ai.usage.input_tokens`); use `ai.*` for the AI-specific extensions.

The naming discipline is: `gen_ai.*` for the published convention; `ai.*` for the extensions; never invent ad-hoc names.

### 6.3 Vendor-specific span enrichment

Some vendors (LangSmith, Braintrust) emit additional attributes useful for their tooling. Capture them when the vendor's value-add depends on them (e.g., LangSmith's run-tree visualization is much more useful when its specific attributes are populated), but do not let vendor-specific naming creep into the platform-wide convention. The vendor attributes ride alongside the standard ones, not in place of them.

---

## 7. Sensitive-data handling in traces

AI traces carry sensitive content by default — the user's question, the retrieved chunks, the model's output. For Meridian, this includes PHI; for other workloads, this can include PII, financial data, regulated content, intellectual property. The trace backend is not the right place to store unbounded sensitive content; the discipline is *what travels in the trace vs what travels in a redirected sink*.

### 7.1 The data-classification decision per attribute

Every attribute has a sensitivity classification:

- **Routine.** Safe to include in trace verbatim (model version, prompt version, tool name, latency, cost).
- **Sensitive (hashed).** Hashed in trace; full content stored in a separate access-controlled sink (queries, tool arguments, prompt content).
- **Sensitive (redirected).** Not in trace at all; written only to the access-controlled sink (PHI, PII, regulated content).

### 7.2 The redirected sink

A separate trace-content store with stricter access controls than the trace backend. Same trace ID linkage; the redirected content can be retrieved by trace ID when needed for debugging, with access logged.

Pattern: ai-platform-eng has access to traces for routine debugging; security-eng + auditors have additional access to the redirected sink for incident investigation; engineers do not have access to the redirected sink in production.

### 7.3 PHI redaction in the pipeline

For Meridian: the trace export pipeline applies PHI redaction before traces leave the application boundary. Patient identifiers, PHI tokens in prompt content, PHI in tool arguments are scrubbed or replaced with placeholders that link to the redirected sink.

The redaction is deterministic (same input produces same placeholder) so trace lineage is preserved; the redaction is auditable (the pipeline logs what it scrubbed); the redaction does not lose attribution (the trace still knows it is for patient `pt-44929`, just via a placeholder).

### 7.4 Trace retention policy

Traces have a retention SLO matching the regulatory requirement. For Meridian's clinical interactions: 7 years (HIPAA-aligned). For routine system traces: 30 days. The retention policy is enforced at the trace backend; old traces are deleted by policy, not by manual cleanup.

---

## 8. Vendor tooling integration

The AI observability vendor space is active and evolving. The integration pattern:

### 8.1 Instrument once, export to many

The application emits OpenTelemetry spans with the standard `gen_ai.*` and `ai.*` attributes. The OpenTelemetry exporter pipeline routes to:

- The team's chosen AI observability vendor (LangSmith, Braintrust, Phoenix, Helicone)
- The team's general APM (Datadog, New Relic, Honeycomb) for unified application observability
- Cold storage (S3, GCS) for long-retention compliance use

This pattern avoids vendor lock-in (switching vendors does not require re-instrumentation) and supports multi-destination routing (different consumers get different views of the same trace).

### 8.2 The vendor's value-add

Different vendors emphasize different value:

- **AI-eval tooling.** LangSmith, Braintrust — strong for running evals against traces, comparing prompt versions, building golden sets from production interactions.
- **Token / cost observability.** Helicone, Portkey — strong for per-call cost attribution and provider-level analytics.
- **General-purpose APM with LLM add-ons.** Datadog, New Relic, Honeycomb — strong for unified observability across the platform.
- **Open-source / self-hosted.** Phoenix, OpenInference — useful when vendor lock-in is a concern or when the workload requires self-hosted observability.

Pick the vendor whose value-add aligns with the team's primary observability need. Most teams end up using two or three for different purposes.

### 8.3 What to NOT delegate to the vendor

- **Trace ID generation and propagation.** Owned by the application; vendors consume.
- **Span structure and attribute taxonomy.** Owned by the platform; vendors render.
- **Sampling decision.** Owned by the platform's exporter; vendors receive sampled traces.
- **Sensitive-data redaction.** Owned by the platform's export pipeline; vendors never see sensitive content.

Delegating any of these to the vendor produces lock-in. Owning them in the platform makes vendor swap a configuration change.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 Trace shape

Every chat interaction produces one trace, with the supervisor / worker hierarchy from section 3.3. Streaming responses do not end the trace until the stream completes (the trace stays open across the streamed tokens).

For the async coordination task, the outer trace records the orchestration; per-patient traces record the per-patient work; events link them. For the HITL side-effect path, the agent trace finalizes when the draft is created, and a follow-up trace span links to the eventual human action.

### 9.2 Attribute population

Every span carries the universal attributes (interaction ID, tenant, feature, user, session, cost). LLM-call spans use `gen_ai.*` conventions plus the `ai.llm.*` extensions for prompt version. Tool-call spans use `ai.tool.*`. Retrieval spans use `ai.retrieval.*` with the full doc-ID list and scores (for retrieval debugging) and the tenant filter that was applied (for defense-in-depth visibility).

The Meridian platform has a versioned attribute schema (currently v2.4); changes go through ai-platform-eng review.

### 9.3 Sampling

Clinical interactions: 100% sampled (volume is ~3K/day across all hospitals; affordable at 100%). The audit posture requires it.

Internal eval and dogfooding traffic: 100% sampled (the trace is the eval feedback signal).

Async coordination tasks: 100% per-patient sampling. The volume is bounded; the regulatory posture requires it.

### 9.4 Vendor integration

The export pipeline routes to:
- **Datadog** for unified application observability (this is where on-call sees AI activity alongside service health).
- **LangSmith** for AI-specific debugging, prompt comparison, and eval-against-production-traces workflows.
- **S3 (encrypted, KMS-managed)** for 7-year compliance retention.

The redirected sink (for PHI-containing trace content) is a separate encrypted S3 bucket with access controls matching the regulatory posture. Routine debugging in Datadog and LangSmith sees redacted traces; security-eng has access to the redirected content when investigating incidents.

### 9.5 The trace-as-debugging-surface discipline

When a Care Coordinator quality regression is investigated, the playbook starts with the trace:

1. The failing interaction's trace is pulled by interaction ID.
2. The trace is read top-down: supervisor → which worker failed → which tool call or LLM call within the worker had the problem.
3. The failing span's attributes give the immediate context: prompt version, model version, retrieved doc IDs, budget state.
4. The likely cause is usually one of: prompt regression (compared against the prior version's trace for the same case class), corpus regression (compared against the doc-IDs the prior trace retrieved), model regression (the model version changed), or new edge case (the input pattern was not represented in the eval suite).
5. The fix lands; a regression case is added; the closing trace verifies the fix.

This is the discipline. The trace is the evidence; the existing trace is read; new experiments are not run.

---

## 10. Anti-patterns

### 10.1 "Logs only, no traces"

The team logs each LLM call as a structured log entry but does not maintain trace hierarchy. Reconstructing what happened across multiple calls requires correlating timestamps and IDs by hand.

**Corrective.** Adopt distributed tracing; emit spans; let the trace backend reconstruct the hierarchy. Logs are useful adjuncts but not substitutes.

### 10.2 "Trace per HTTP request, not per interaction"

The HTTP layer's tracing splits an SSE-streamed chat response into multiple short traces (one per chunk). The user's interaction is reassembled by the team manually.

**Corrective.** Define the trace at the interaction boundary; HTTP-level spans become children of the interaction span. The interaction span stays open across the streamed response.

### 10.3 "Attribute names ad-hoc per team"

Different teams use different attribute names for the same concept (`model`, `model_name`, `gen_ai.model`, `ai.model`). Cross-feature queries are impossible because the attribute schema does not align.

**Corrective.** Platform-wide attribute taxonomy; documented; versioned; new attributes go through review. Use `gen_ai.*` and `ai.*` consistently.

### 10.4 "All sampling random, none tail-based"

10% random sampling loses the 90% of failing interactions that did not happen to be sampled. Investigations have to "wait for it to happen again to a sampled user."

**Corrective.** Random sample for trend + tail-based augmentation for failures, slow tails, circuit trips, judge failures, HITL rejections, user feedback.

### 10.5 "Full retrieval content in trace attributes"

Retrieved chunks (1000+ tokens each) go into trace attributes. The trace backend's cardinality and storage limits are hit; queries become slow; cost balloons.

**Corrective.** Doc IDs in trace attributes; full content in a separate retrieval-content store linked by doc ID. The trace tells you *what* was retrieved; the content store tells you *what was in it* on demand.

### 10.6 "Vendor's instrumentation is the platform's instrumentation"

The team adopted a vendor that auto-instruments the LLM SDK. Spans appear in the vendor; when the team needs to add custom attributes or change the span structure, they discover the auto-instrumentation does not support it.

**Corrective.** Own the instrumentation; the vendor consumes standardized spans. Auto-instrumentation is a starting point, not an architecture.

### 10.7 "Sensitive content in traces with reactive cleanup"

PHI / PII / regulated content goes into traces; when discovered, the team writes a script to retroactively redact. Old traces stay sensitive; the redaction is incomplete; compliance posture is degraded.

**Corrective.** Sensitive-content classification per attribute; redaction in the export pipeline before traces leave the application boundary; never write sensitive content to the trace backend.

### 10.8 "Trace retention by default"

Traces are stored at the trace backend's default retention (often 14 days). Regulated workloads' requirements are not met; long-tail incident investigations cannot find the trace.

**Corrective.** Retention policy aligned with the regulatory requirement and the investigation horizon. Long retention can be in cheaper cold storage; routine retention in the hot trace backend.

---

## 11. Findings (sprint-assignable)

### OBS-TRACE-001 — Severity: Critical
**Finding.** Production AI feature emits logs but no distributed traces; cross-call reconstruction is manual.
**Recommendation.** Adopt OpenTelemetry tracing; emit spans per the hierarchy in section 3; integrate with existing APM.
**Owner.** ai-platform-eng + observability-eng, sprint N+1.

### OBS-TRACE-002 — Severity: Critical
**Finding.** Sensitive content (PHI / PII / regulated) is included in trace attributes; trace backend is not authorized to hold it.
**Recommendation.** Sensitivity classification per attribute (section 7.1); redaction in the export pipeline; redirected sink for sensitive content.
**Owner.** ai-platform-eng + security-eng, sprint N+1.

### OBS-TRACE-003 — Severity: High
**Finding.** Attribute names are inconsistent across teams / features; cross-feature queries are impossible.
**Recommendation.** Platform-wide attribute taxonomy; adopt `gen_ai.*` for published convention; extend with `ai.*` for AI-specific extensions.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-TRACE-004 — Severity: High
**Finding.** Trace boundary is per-HTTP-request; streaming interactions are split into multiple short traces.
**Recommendation.** Define trace at the interaction boundary; HTTP spans are children of the interaction span.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-TRACE-005 — Severity: High
**Finding.** Sampling is 100% random with no tail-based augmentation; storage cost is unsustainable.
**Recommendation.** Random sample + tail-based augmentation per section 5.2; reduce baseline rate; preserve high-value cases.
**Owner.** ai-platform-eng + observability-eng, sprint N+2.

### OBS-TRACE-006 — Severity: High
**Finding.** Random sampling at 10% loses most failed interactions; investigations cannot reconstruct.
**Recommendation.** Tail-based keep on failures, circuit trips, judge failures, HITL rejections, user feedback.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-TRACE-007 — Severity: High
**Finding.** Full retrieval content (1000+ tokens per chunk) goes into trace attributes; cardinality limits are exceeded.
**Recommendation.** Doc IDs in trace; full content in separate retrieval-content store linked by doc ID.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-TRACE-008 — Severity: High
**Finding.** Async / HITL trace lineage is broken — agent-side traces do not link to per-patient or human-action follow-up traces.
**Recommendation.** Parent-trace event linkage per section 2.3 / 2.4.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-TRACE-009 — Severity: Medium
**Finding.** Trace retention is the backend default (14 days); regulatory requirement is 7 years.
**Recommendation.** Tiered retention: hot in trace backend for 30 days; cold archive (encrypted S3) for the regulatory term.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### OBS-TRACE-010 — Severity: Medium
**Finding.** Vendor-specific span enrichment from one vendor leaks into platform attribute convention; vendor swap would be expensive.
**Recommendation.** Vendor attributes ride alongside standard ones; never replace; standardize the platform convention.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-TRACE-011 — Severity: Medium
**Finding.** Trace export uses vendor's SDK directly rather than OpenTelemetry; cannot multi-route to other consumers.
**Recommendation.** OpenTelemetry collector pipeline; route to multiple consumers; vendor SDK consumes from collector.
**Owner.** ai-platform-eng + observability-eng, sprint N+3.

### OBS-TRACE-012 — Severity: Medium
**Finding.** Per-tenant, per-feature attribution attributes are not consistently populated; per-segment analysis is incomplete.
**Recommendation.** Universal attributes (section 4.1) enforced at span creation; lint rule against spans missing the required attributes.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-TRACE-013 — Severity: Medium
**Finding.** Quality-judge sampling results are not attached to traces; correlation with system behavior is manual.
**Recommendation.** Add `ai.quality.judge.*` attributes to interactions that the online judge sampled.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-TRACE-014 — Severity: Medium
**Finding.** Circuit-breaker trips are logged but not surfaced as span attributes on the failing trace.
**Recommendation.** Add `ai.circuit.tripped` attributes per section 4.7; trace shows the trip in context.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-TRACE-015 — Severity: Medium
**Finding.** Span naming is inconsistent; queries across teams require alias maps.
**Recommendation.** Span naming convention (section 3.5); document; enforce via lint or convention review.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-TRACE-016 — Severity: Low
**Finding.** Trace backend cost is meaningful and growing; team has not implemented tail-based aggressive sampling.
**Recommendation.** Aggressive sampling on successful, low-cost, low-value interactions; full sampling on the keep-criteria.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### OBS-TRACE-017 — Severity: Low
**Finding.** Trace-content schema versioning is informal; consumers cannot tell which attribute schema a trace was emitted with.
**Recommendation.** `ai.trace.schema_version` attribute on every span; major bumps when attribute meanings change.
**Owner.** ai-platform-eng, sprint N+5.

### OBS-TRACE-018 — Severity: Low
**Finding.** Newcomer engineers cannot read traces effectively because the attribute taxonomy is undocumented.
**Recommendation.** Generate attribute documentation from the taxonomy schema; commit alongside code; include in onboarding.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team adding AI observability to a feature in production:

- [ ] **Sprint 0 — decide.** Choose the trace backend (existing APM, AI-specific vendor, or both). Pick the OpenTelemetry exporter path.
- [ ] **Sprint 1 — boundary.** Define trace boundaries at the interaction level. Trace ID generation at the chat panel / API endpoint / task initiation.
- [ ] **Sprint 1 — minimal spans.** Emit spans for the major operations: LLM call, retrieval, tool call. Use the published `gen_ai.*` convention for LLM-call attributes.
- [ ] **Sprint 2 — attribute taxonomy.** Define the platform attribute schema. Implement universal attributes. Document.
- [ ] **Sprint 2 — sensitive data.** Sensitivity classification per attribute; redaction pipeline; redirected sink.
- [ ] **Sprint 3 — span hierarchy.** Wrap the agent / workflow shape into the span hierarchy. Reading the trace top-down should match reading the architecture top-down.
- [ ] **Sprint 3 — sampling.** Random + tail-based. Keep criteria for failures, circuits, judges, HITL rejections, feedback.
- [ ] **Sprint 4 — vendor integration.** If using an AI observability vendor, integrate via OpenTelemetry collector; do not lock into the vendor SDK.
- [ ] **Sprint 4 — retention.** Tiered retention aligned with regulatory and investigation requirements.
- [ ] **Ongoing — discipline.** Every new attribute goes through schema review. Every new span type uses the naming convention. The trace is the primary debugging surface.

A team that completes this sequence has the foundation for the per-call (`llm-call-instrumentation.md`) and per-agent-step (`agent-step-instrumentation.md`) patterns that build on top.

---

## 13. References

- W3C Trace Context specification — the propagation standard.
- OpenTelemetry semantic conventions, including the `gen_ai.*` namespace (2024-2026).
- OpenInference, LangSmith run-tree, Braintrust trace model — vendor span conventions that align with or extend OpenTelemetry.
- Phoenix, Helicone, Portkey, Datadog LLM Observability documentation — vendor capabilities discussed in section 8.
- HIPAA Security Rule §164.312 — the audit / retention requirements that shape Meridian's trace design.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) for the per-call patterns.
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](./agent-step-instrumentation.md) for the agent-loop patterns.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](./) (coming) for the retrieval-specific attributes.
- This repo: [cost-and-finops/cost-attribution.md](../cost-and-finops/) (coming) for the cost-attribution pipeline that consumes trace attributes.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the architecture context.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/ai-gateway-pattern.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/) (coming) — the platform component that emits many of these traces.
