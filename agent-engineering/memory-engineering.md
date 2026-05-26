# Memory Engineering

> **Audience.** Engineers building or refactoring an agent's memory layer — short-term context strategy, long-term store, per-conversation episodic memory, cross-conversation semantic memory, and the retention/forgetting policy. Tech leads making memory architecture decisions. **Scope.** The engineering depth on memory: taxonomy, implementation patterns, retention policy, failure modes. Not the RAG retrieval pipeline (see [rag-engineering/](../rag-engineering/)) — though episodic and semantic memory share retrieval primitives. Not memory as architecture decision (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)). **Worked client.** Meridian Health.

---

## 1. Why this document exists

"Memory" in agent systems is a category, not a thing. Four distinct memory types — short-term context, long-term store, episodic per-conversation, semantic cross-conversation — solve four distinct problems with four distinct implementations. Teams that conflate them produce systems that are simultaneously over-built (long-term semantic memory for a use case that only needs a running summary) and under-built (no episodic memory at all, so every retry rebuilds context).

The failure modes are particular. Context windows fill silently and degrade quality before they error. Long-term stores accumulate stale facts that contradict each other; the model picks the wrong one. Episodic memory survives across retries but drifts across sessions if poorly engineered. Semantic memory hallucinates facts that aren't anchored to a real source. Each failure mode has a distinct corrective.

The most common engineering mistake is "let's add memory" as a vague feature — bolting a vector store on the side, writing every conversation turn to it, and hoping the model uses it well. The result is memory that's expensive to maintain, noisy in retrieval, and untrusted by the model. The right approach starts with which of the four memory types the use case actually requires, what data shape each requires, and what retention policy each follows.

This document is opinionated about four things:

1. **Memory is engineered by type, not as one bucket.** Each memory type has its own data shape, write path, read path, retention policy, and failure modes. Treating them uniformly produces a system that's wrong for all of them.
2. **Short-term context is the default; add other types only when needed.** Most "memory" features can be served by a well-engineered context-window strategy. Long-term, episodic, and semantic memory should be added explicitly when the use case proves the need.
3. **Retention and forgetting are first-class.** A memory store without a forgetting policy accumulates indefinitely until quality degrades. The forgetting policy is part of the memory design, not an afterthought.
4. **Memory must be observable.** What's in memory, what was retrieved, what was injected into the prompt — all visible in the trace. Memory bugs are the hardest agent bugs to debug without this visibility.

Structure: (2) the memory taxonomy; (3) short-term context engineering; (4) long-term memory store; (5) episodic memory (per-conversation); (6) semantic memory (cross-conversation); (7) retention and forgetting policy; (8) memory observability and debugging; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The memory taxonomy

Four types, four problems, four implementations.

### 2.1 Short-term context

**Problem.** Within a single agent invocation (one loop, possibly many turns), the model needs to remember what it has seen — the user's input, the tools it called, the results it received, the partial conclusions it drew.

**Implementation.** The conversation history sent in each LLM call. Tokens accumulate within the context window. Strategies for managing the accumulation: window trimming, summarisation, structured-extraction.

**Lifetime.** Single agent invocation. Discarded when the loop terminates.

**Failure modes.** Context-window saturation; quality degradation as window fills; loss of early context when trimming.

### 2.2 Long-term store

**Problem.** Facts about the world that the agent needs across invocations — domain knowledge, policies, procedures, reference data, user preferences.

**Implementation.** A retrieval system over a curated knowledge corpus. Often a vector store with an embedding pipeline; sometimes a structured database; sometimes a hybrid. Read into context on demand via retrieval tools.

**Lifetime.** Long. Updated when the underlying facts change. Quarterly or as-events-warrant.

**Failure modes.** Stale facts (the world changed; the store didn't); irrelevant retrieval (noisy chunks); hallucinated synthesis (the model invents a fact attributed to a source).

### 2.3 Episodic memory (per-conversation)

**Problem.** Within a long-running conversation that spans multiple agent invocations (each user turn invokes the agent), the model needs to remember what happened earlier in *this conversation*. The user mentioned their child's age earlier; later they ask about pediatric care — the agent should connect them.

**Implementation.** A per-conversation memory record. Append-only log of conversation events; possibly summarised; retrieved at each invocation. Distinct from short-term context (which is the in-flight loop's history) — episodic memory persists across invocations within the same conversation.

**Lifetime.** Conversation duration. Possibly persists after conversation ends (depending on policy).

**Failure modes.** Episodic drift (the summary loses detail across many turns); contradiction (the user said one thing then another; the memory has both); leakage across conversations (a poorly-scoped query returns another conversation's data).

### 2.4 Semantic memory (cross-conversation)

**Problem.** Facts about *the user* (or *the tenant*, or *the entity*) that should persist across conversations. The user told the agent last week they prefer email; this week the agent should remember.

**Implementation.** A per-entity memory store — user profile, tenant profile, patient profile. Structured (preferences as fields) or unstructured (notes the agent has accumulated). Retrieved at the start of each conversation; updated by explicit memory-write tools.

**Lifetime.** Indefinite (with forgetting policy). Lives at the entity scope (user, tenant), not the conversation scope.

**Failure modes.** Hallucinated facts (the agent thinks the user said something they didn't); cross-tenant leakage; staleness (the user's preference changed; memory didn't); attribution errors (the agent attributes a fact to the wrong source).

### 2.5 The decision: which types do you need?

| Use case | Short-term | Long-term | Episodic | Semantic |
| --- | --- | --- | --- | --- |
| Single-turn Q&A over docs | Yes | Yes (the docs) | No | No |
| Multi-turn Q&A in one session | Yes | Yes | Maybe | No |
| Multi-session conversational assistant | Yes | Yes | Yes | Maybe |
| Personalised assistant (cross-session) | Yes | Yes | Yes | Yes |
| Agentic task with tool use | Yes | Maybe | Maybe | No |

The decision starts with the use case. Most single-turn and multi-turn-single-session features need only short-term + long-term. Episodic and semantic add real value only when the use case is genuinely conversational across sessions.

### 2.6 The "memory" framework feature

Several frameworks offer "memory" as a feature — typically a wrapper around a vector store with auto-write of every conversation turn. This is *one implementation of one type* (often episodic, sometimes semantic). It is rarely the right starting point because it commits to a structure before the use case is analysed. Build the memory taxonomy for the use case first; choose the implementation that fits.

---

## 3. Short-term context engineering

The default. Get this right before any other memory type.

### 3.1 The context-window budget

The context window is a budget. Tokens spent on conversation history cannot be spent on retrieval results, system prompt, or generation. The budget breakdown for a typical agent turn:

| Component | Token range |
| --- | --- |
| System prompt | 1k–3k |
| Tool definitions | 1k–4k (varies with catalogue size) |
| Long-term retrieval results | 2k–8k |
| Conversation history (this loop) | 2k–20k (grows per turn) |
| Episodic memory (if present) | 1k–4k |
| Semantic memory (if present) | 0.5k–2k |
| Generation budget | 1k–4k |

Total typical: 8k–45k. Even with a 200k context window, the budget warrants engineering — quality often degrades on the bottom edge of the window even when tokens fit.

### 3.2 History accumulation strategies

As the loop progresses, conversation history grows: user input + each turn's model response (with tool calls) + each tool's result. Without management, a 30-turn loop's history can exceed 50k tokens.

Three strategies, in increasing complexity:

**Strategy 1: Full history.** Pass everything. Fine for short loops (< 10 turns) and short user inputs. Simple. Degrades as the loop lengthens.

**Strategy 2: Sliding window.** Keep the last N turns plus the initial user input. The model loses access to middle turns. Cheap. Loses information from older tool results that may still be relevant.

**Strategy 3: Running summary + recent turns.** After every N turns (or when token budget pressure rises), summarise the older turns into a compact summary; keep the recent turns verbatim. The model sees a "what happened so far" summary plus the recent exchange.

The summary strategy is the most common production pattern. Implementation: a dedicated summariser LLM call that takes the older turns and produces a compact summary; the summary replaces the older turns in subsequent calls.

### 3.3 The structured-extraction pattern

Beyond summary, structured extraction keeps key facts as discrete items:

```json
{
  "user_intent": "schedule a follow-up",
  "patient_context": {
    "patient_id": "uuid-xxx",
    "name": "Jane Doe",
    "confirmed_dob": "1980-01-15"
  },
  "tool_results_so_far": {
    "last_visit": "2026-02-10, cardiology",
    "open_referrals": ["pulmonology pending"],
    "current_meds": ["metoprolol 50mg"]
  },
  "decisions_made": [
    "Verified patient identity",
    "Confirmed cardiology follow-up was completed"
  ]
}
```

The extraction is updated per turn; included in the prompt as "current state." More structured than a prose summary; easier for the model to consume; legible to humans reading the trace. Engineering investment is higher (schema design, extraction prompt).

### 3.4 The "every turn re-derives state" anti-pattern

Some loops re-derive state from scratch each turn by re-passing the full history. Wasteful in tokens (the model parses the same history repeatedly), expensive in cost, slow. Compress the older history; the model only needs to re-derive at the moment it last left off.

### 3.5 Short-term context observability

The trace shows, per turn: the in-prompt context size, the history-management strategy applied, the summary contents (if produced), and the structured extraction state (if maintained). When the agent makes a wrong decision, the trace shows what it knew at that turn — debugging by trace reading.

---

## 4. Long-term memory store

The agent's knowledge of the world. Implemented as retrieval over a curated corpus.

### 4.1 What belongs in long-term memory

- Domain reference data (e.g., medication formulary, policy documents, code repositories).
- Procedural knowledge (how to handle a specific case).
- Reference examples (golden cases the agent may want to reference).
- Tenant-specific configuration (workflows, preferences at the tenant level).

What does NOT belong: per-conversation state (use episodic), per-user preferences (use semantic), in-flight tool results (use short-term).

### 4.2 Implementation: retrieval primitives

Long-term memory is implemented as a retrieval system; the agent accesses it via tools:

- `search_documents(query)` — returns relevant passages from a corpus.
- `fetch_document(doc_id)` — retrieves a known document.
- `lookup_reference(reference_name, key)` — structured lookup (e.g., formulary entry by drug name).

The agent calls these tools when it needs the information; the results enter the short-term context.

The retrieval primitives are themselves engineering objects: the retrieval pipeline, embedding model, chunking strategy, reranker — all governed by the disciplines in [rag-engineering/](../rag-engineering/). Memory engineering doesn't duplicate them; it composes on top of them.

### 4.3 Writes to long-term memory

Writes are infrequent and controlled. The agent does not write to long-term memory directly; an out-of-band process does:

- A content management system feeds the document ingestion pipeline (per [ingestion-pipeline-engineering.md](../rag-engineering/ingestion-pipeline-engineering.md)).
- Documents are versioned; updates re-embed and re-index.
- Stale documents are removed or marked deprecated.

The discipline keeps long-term memory authoritative — the model retrieves from a corpus that humans curate, not from agent-generated artefacts that might be wrong.

### 4.4 The "agent-written long-term" anti-pattern

A common mistake: the agent writes its conclusions to long-term memory ("I learned that X is true"). The next agent invocation retrieves the model's conclusion as a fact and treats it as ground truth. Hallucinations become persistent. Errors compound.

Long-term memory is for curated facts. Agent conclusions belong in episodic or semantic memory, with provenance attached (the model said this, on this date, with this confidence).

### 4.5 Long-term memory observability

The retrieval-instrumentation pattern (per [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md)) covers it: every retrieval is a span with query, hit count, top results, latency, cost. The trace shows what was retrieved when; debugging "the agent said X but the source doesn't support X" is a matter of opening the trace and checking what the retrieval returned.

---

## 5. Episodic memory (per-conversation)

The conversation's own log. Persists across agent invocations within a conversation.

### 5.1 What episodic memory contains

- The conversation turns (user message, agent response).
- The tool results from each invocation.
- Structured extractions from each turn (per section 3.3, if used).
- Conversation-level state (current topic, patient under discussion, pending tasks).

The store is conversational-scoped: one conversation, one episodic record.

### 5.2 Implementation patterns

**Pattern A: append-only conversation log.** Each turn appends an entry. On agent invocation, the recent N entries (or all if short) are loaded as context. Simple. Works for short-to-medium conversations.

**Pattern B: append + running summary.** The same as A, plus a per-conversation summary that's updated periodically (e.g., every 10 turns). The summary is loaded; recent turns are loaded; older turns are compressed.

**Pattern C: structured conversation state.** A schema captures the conversation's state (current patient, current topic, pending follow-ups). The state is updated by the agent (via memory-write tools) or by a deterministic state extractor. The state is loaded per invocation; raw turns may not be needed.

Pattern A is the default. Pattern B for long conversations. Pattern C for conversations where the state is meaningful and structured (e.g., a multi-step intake interview).

### 5.3 Read path

At each agent invocation:

1. Fetch the episodic record for this conversation (by conversation_id).
2. If the record exceeds a token budget, apply summary + recent strategy.
3. Insert into the system prompt as "context from this conversation so far."

### 5.4 Write path

At end of each agent invocation:

1. Append the new turn (user input + agent response + tool results).
2. If the record exceeds a turn count, trigger the summarisation step.
3. If structured state is maintained, update it.
4. Persist.

### 5.5 Episodic memory scope

The conversation is identified by a `conversation_id`. The episodic record is keyed by `(tenant_id, conversation_id)` — the tenant scope prevents cross-tenant leakage even if conversation_ids collide.

### 5.6 Episodic vs short-term boundary

Short-term is the agent's in-flight loop history (single invocation, many turns within the loop). Episodic is the conversation's history (across invocations). The agent's short-term context is freshly built each invocation; episodic feeds the rebuild.

For a single-invocation interaction (one user message → one response with no follow-up), episodic = short-term. For a multi-message conversation, they're distinct.

### 5.7 Failure modes

- **Episodic drift.** The summary loses fidelity over many turns; the agent forgets details the user mentioned earlier. Corrective: structured extraction (Pattern C) for the load-bearing facts; summary for the rest.
- **Contradiction.** The user said two things; the episodic record has both. Corrective: the agent is responsible for asking for clarification when it detects contradiction; the memory layer surfaces all contradictory entries.
- **Cross-conversation leakage.** A bug in scoping returns another conversation's data. Corrective: tenant + conversation scope is enforced at the storage layer, not just the application layer; integration tests verify isolation.
- **Episodic poisoning.** A malicious or buggy turn writes false data; subsequent invocations treat it as fact. Corrective: provenance attached to each entry; agent is instructed to treat episodic entries as "what was said" not "what's true."

---

## 6. Semantic memory (cross-conversation)

Facts about the user, tenant, or entity that persist across conversations.

### 6.1 What semantic memory contains

- User preferences ("prefers email," "prefers concise summaries," "primary language: Spanish").
- Stable facts the user provided in past conversations ("has a child named Sam, age 7").
- Past conversation summaries (when the agent should reference what happened before).
- Cross-session task state ("ongoing case #1234 is awaiting lab results").

The store is entity-scoped: one user, one semantic record. Multiple entity types possible (user-level, tenant-level, patient-level for healthcare).

### 6.2 Implementation patterns

**Pattern A: structured profile.** A schema captures known facts (preferences, demographics, stable attributes). Updated by explicit writes. Read on every invocation. Simple and reliable.

**Pattern B: notes log.** Append-only entries the agent writes ("user mentioned they're traveling next week"). Read by retrieval or by recent-N. Flexible but harder to keep coherent.

**Pattern C: hybrid.** Structured profile for the known fields; notes log for the flexible facts. Most production systems land here.

### 6.3 Read path

At conversation start:

1. Fetch the semantic record for the entity (e.g., the user).
2. Include the structured profile in the system prompt.
3. Include the most recent or most relevant notes (selection strategy: recency, relevance to current topic, manual flags).

### 6.4 Write path

Explicit memory-write tools:

- `remember_preference(field, value)` — updates a structured field.
- `add_note(text, tags)` — appends a note.
- `update_summary(text)` — replaces the rolling summary.

The agent uses these tools deliberately. The tool descriptions include guidance on when to use ("remember only stable facts; transient state goes in short-term").

### 6.5 The "auto-write every turn" anti-pattern

Some implementations write every user input as a note to semantic memory. The notes pile up; retrieval becomes noisy; the agent gets contradictory hints. Corrective: writes are deliberate, gated by a "should I remember this?" decision (the agent's prompt explicitly asks the question), or by a downstream extractor that picks stable facts from the conversation summary.

### 6.6 Semantic memory's failure modes

- **Hallucinated facts.** The agent writes "user said they prefer X" when the user didn't say so. Corrective: writes require provenance (which conversation, which turn). On read, the agent (via prompt) is instructed to verify facts before relying on them.
- **Cross-tenant leakage.** Semantic records are scoped at `(tenant_id, entity_id)`. Storage-level enforcement.
- **Staleness.** Preferences change; the record doesn't. Corrective: forgetting policy (section 7) plus periodic refresh prompts ("Earlier you mentioned X; is this still accurate?").
- **Attribution errors.** A fact written from conversation A is mis-attributed. Corrective: provenance per entry.
- **Privacy.** Semantic memory accumulates sensitive personal information. Subject to retention policies, deletion requests, audit. Treat as the most sensitive store the agent touches.

### 6.7 Semantic memory and privacy

Healthcare, finance, legal — domains with strong privacy requirements — make semantic memory a regulatory concern. The store contains PII or PHI. The retention policy must comply with data residency, right-to-be-forgotten, retention-window rules. Engineer the deletion path before the writes accumulate. See sibling [ai-security-reference-architecture]'s `privacy-and-compliance/` folder.

---

## 7. Retention and forgetting policy

A memory store without a forgetting policy accumulates indefinitely. Quality degrades; cost grows; privacy risk compounds.

### 7.1 Why retention matters

- **Quality.** Stale facts contradict current facts; retrieval brings in outdated information; the model has to disambiguate.
- **Cost.** Storage grows; indexing cost grows; retrieval latency may grow.
- **Privacy.** Personal data accumulates beyond the consent or retention requirement.
- **Audit.** Long-retained data is in scope for data requests, audits, breaches — expanding the impact of any incident.

### 7.2 Retention policies by memory type

| Memory type | Retention horizon | Forgetting mechanism |
| --- | --- | --- |
| Short-term | Loop duration | Discarded at loop end |
| Long-term | Source-controlled | Updated/deleted at the source; re-indexed |
| Episodic | Conversation + N days | Configurable; default e.g. 30 days post-conversation |
| Semantic | Indefinite with refresh + entity-scoped | TTL on entries (e.g., 12 months); deletion on entity removal |

### 7.3 Episodic retention

After a conversation ends, the episodic record may be retained for some period (to resume the conversation, to support follow-up, for audit). After that, it is either:

- **Deleted entirely** (privacy-preserving; most strict).
- **Summarised into semantic memory** (the relevant facts are extracted to the user's semantic profile; the raw conversation is deleted).
- **Archived for audit** (kept in cold storage with restricted access; not used by the agent).

The policy is per-tenant, per-product, possibly per-user (consent flag).

### 7.4 Semantic retention

Semantic records grow over time. Forgetting strategies:

- **TTL per entry.** Notes older than N months are dropped automatically.
- **Refresh on use.** When the agent retrieves and uses a fact, its TTL extends.
- **Periodic compaction.** A background process consolidates entries: superseded facts are removed; structured fields are updated to current state.
- **Explicit forget tool.** The user (or the agent on behalf of the user) can delete specific facts.
- **Right-to-be-forgotten.** The entity can be deleted entirely; all semantic memory for the entity is removed.

The compaction process is itself a careful engineering exercise — it can introduce contradictions if poorly designed.

### 7.5 The deletion path

Compliance with privacy regulation requires reliable deletion. The path:

1. Deletion request received.
2. Identify all memory stores holding data for the entity.
3. Delete (or mark for deletion) all entries.
4. Verify deletion completed; log the action.
5. Audit trail of what was deleted and when.

Across all four memory types (though short-term is moot since it's discarded already). The deletion path must be regularly tested.

### 7.6 The retention-policy review

Quarterly review of the retention policies. The world changes; regulation changes; product needs change. Policies that were right a year ago may be wrong now.

---

## 8. Memory observability and debugging

Memory bugs are invisible without observability. Engineer this from day one.

### 8.1 What to observe

**Per agent invocation:**

- Short-term context size in tokens (broken down by component).
- Episodic memory load: which conversation, how many entries, total tokens.
- Semantic memory load: which entity, structured profile size, notes loaded.
- Long-term memory retrievals: queries, hits, top results.
- Memory-write tool calls: which tool, what was written.

**Per memory store:**

- Total entries (per tenant, per entity).
- Growth rate.
- Retention-policy compliance (entries over TTL).
- Access patterns (which entries are read; which are dead).

### 8.2 The trace as memory debugger

When the agent makes a wrong decision, the trace should answer:

- What did the agent know? (Episodic + semantic + retrieved long-term that was in context.)
- What did it not know that would have mattered? (Inspect the conversation history; was the relevant fact written to episodic?)
- Did the retrieval bring back the right context?
- Did the memory-write tool capture what should have been captured?

Each answer is a span attribute or a memory-store query result. The debugging is procedural.

### 8.3 Memory-specific alerts

- **Episodic record size exceeded.** A conversation's episodic record exceeds the configured budget; investigate (conversation may be looping, summary may have failed).
- **Semantic memory write rate spike.** The agent is writing many notes for an entity; may indicate an over-eager auto-write or a misbehaving prompt.
- **Retention-policy violations.** Entries past TTL still present; the forgetting process is not running.
- **Cross-tenant leakage suspected.** Anomalous access pattern (queries return data from outside the expected scope).

### 8.4 Memory diff tools

For debugging "the agent's memory is wrong," tools to:

- Dump the current semantic record for an entity.
- Dump the episodic record for a conversation.
- Show the diff between two points in time.
- Show provenance for each entry (which conversation, which turn).

Without these, memory bugs are caught only by their downstream symptoms.

### 8.5 The eval surface for memory

- **Episodic accuracy.** Multi-turn eval set; verify the agent uses earlier-turn facts correctly.
- **Semantic accuracy.** Cross-conversation eval set; agent should remember stable facts from prior conversations and apply them.
- **Forgetting compliance.** Eval that runs after a TTL elapse; verify expired entries are gone.
- **Cross-tenant isolation.** Eval with multi-tenant data; verify no cross-leakage.

The eval surface lives in [agent-evals.md](./agent-evals.md).

---

## 9. Worked Meridian example

Meridian's care-coordinator agent uses three of the four memory types.

### 9.1 The four types at Meridian

| Type | Used? | Implementation |
| --- | --- | --- |
| Short-term | Yes (every invocation) | Running summary after 8 turns; structured state extraction |
| Long-term | Yes | RAG over policy docs, formulary, clinical guidelines, patient-records (read-only) |
| Episodic | Yes | Append-only conversation log per (tenant_id, conversation_id), 30-day retention |
| Semantic | Partial | Structured per-patient profile only; no agent-written notes |

### 9.2 Short-term: running summary + structured state

The loop runs up to 12 turns. After turn 8, a summariser LLM call collapses turns 1–6 into a 200-token summary; turns 7+ are preserved verbatim. The structured state extracts the patient under discussion, the topic, the tools called so far, and the open questions:

```json
{
  "patient_in_focus": {"patient_id": "uuid-xxx", "name": "Jane Doe"},
  "topic": "follow-up scheduling for cardiology",
  "tool_results": {
    "last_visit": "2026-02-10 cardiology",
    "next_appointment": null,
    "outstanding_referrals": []
  },
  "open_questions": [
    "Confirm patient preference for in-person vs telehealth"
  ]
}
```

The state is included in the system prompt of each turn; the model uses it to maintain coherence across the loop.

### 9.3 Long-term: RAG over clinical knowledge

Long-term memory is a vector store + structured lookups:

- **Documents:** clinical guidelines, care protocols, internal policy, payer policy documents.
- **Structured:** medication formulary lookup (by drug name → details), procedure code lookup.

The agent has tools `search_clinical_guidelines(query)`, `lookup_medication(name)`, `lookup_procedure_code(code)`. Retrieval is governed by the standard RAG pipeline (per the `rag-engineering/` folder).

Long-term memory is updated when source content changes; the model never writes to it.

### 9.4 Episodic: per-conversation log

Each clinician's conversation with the care coordinator is a long-running interaction (typically 5–30 turns across an afternoon). The episodic record persists across the day:

- **Storage.** Postgres table keyed by `(tenant_id, conversation_id, turn_id)`. Entries: user message, agent response, tool results, structured state at end of turn.
- **Load.** On agent invocation, the last N turns + the running summary load into the short-term context as "context from this conversation so far."
- **Retention.** 30 days after the conversation's last activity, the record is summarised (a compact "what this conversation was about") and the raw turns are deleted. The summary is retained for 1 year for audit.

The pattern handles the multi-turn-in-one-session and multi-session-in-one-week cases well.

### 9.5 Semantic: patient profile (structured only)

Per-patient semantic memory is a structured profile:

- Demographics (canonical source: EHR; cached in profile).
- Care preferences (telehealth vs in-person, preferred contact method).
- Stable clinical context (chronic conditions, active care plans).
- Last-conversation summary (one paragraph, from the episodic compaction).

Notably absent: agent-written free-text notes. The team explicitly rejected the auto-write pattern because of hallucination risk and PHI accumulation.

Updates are explicit:

- Demographics: written by the EHR sync.
- Care preferences: written by a deliberate `update_preference` tool the agent calls after explicit user confirmation.
- Stable clinical context: written by the EHR sync.
- Last-conversation summary: written by the episodic compaction job.

### 9.6 The deletion path

A patient's right-to-be-forgotten request triggers:

1. Delete all episodic records for the patient (across all conversations).
2. Delete the semantic profile.
3. Delete all in-flight short-term context (none survives, this is a no-op).
4. Long-term store: patient is in EHR-sourced retrieval; deletion is upstream in the EHR.
5. Audit log entry: deletion completed, what was deleted, by what request.

Tested quarterly with a synthetic patient identifier.

### 9.7 The memory engineering investment

Two engineering quarters' work:

- **Q1: short-term + long-term.** Running summary, structured state extraction, RAG pipeline. The agent works well for single-session multi-turn use.
- **Q2: episodic + semantic + retention.** Conversation log, patient profile, retention policy and deletion path, compaction job, observability.

The team explicitly resisted adding an "auto-write semantic notes" feature despite product pressure; the resistance proved correct when a similar product at a competitor had a PHI-leak incident traced to such a feature.

---

## 10. Anti-patterns

### 10.1 "One memory store for everything"

The team builds one vector store, writes every conversation turn to it, and calls it "memory." Retrieval is noisy, the store is expensive, the model is confused by mixed-purpose results.

**Corrective.** Memory by type. Short-term context engineered separately. Long-term as RAG. Episodic per-conversation. Semantic per-entity.

### 10.2 "Auto-write every conversation turn to semantic memory"

Every user message gets saved to the user's semantic record. Hallucinations propagate; retrieval is noisy; privacy is degraded.

**Corrective.** Writes are deliberate. Either a "should I remember this?" decision in the prompt or a downstream extractor that picks stable facts.

### 10.3 "Agent writes its conclusions to long-term memory"

The agent's outputs are written back to the long-term store; future invocations treat hallucinations as ground truth.

**Corrective.** Long-term memory is curated, sourced by humans. Agent conclusions go to episodic with provenance.

### 10.4 "No retention policy"

The memory stores grow forever. Quality degrades; cost grows; privacy risk compounds.

**Corrective.** Retention policies per type. Forgetting mechanism enforced. Quarterly review.

### 10.5 "No deletion path"

A privacy request arrives; the team scrambles to find every store holding the entity's data; deletion is partial and unaudited.

**Corrective.** Deletion path designed and tested before writes accumulate. Quarterly synthetic-deletion test.

### 10.6 "Episodic memory leaks across conversations"

A bug in scoping returns another conversation's data. The agent answers with another patient's information.

**Corrective.** Storage-level scoping (tenant + conversation). Integration tests verify isolation. Span attribute records the scope on every retrieval.

### 10.7 "Memory not observable"

The trace doesn't show what was in memory, what was retrieved, what was written. Memory bugs are caught only by downstream symptoms.

**Corrective.** Memory state in every span. Memory-write tool calls logged. Memory diff tools for debugging.

### 10.8 "Memory bolted on as a framework feature"

The framework's memory wrapper is enabled; no thought to which memory type the use case needs.

**Corrective.** Memory taxonomy analysis before implementation. Pick the type(s) the use case requires; build for them.

---

## 11. Findings (sprint-assignable)

### AGT-MEM-001 — Severity: Critical
**Finding.** Auto-write of every conversation turn to semantic memory; agent-generated content treated as fact.
**Recommendation.** Disable auto-write; replace with deliberate write pattern per section 6.5; audit existing semantic records and clean hallucinated entries.
**Owner.** ai-platform-eng + feature-team, sprint N+1.

### AGT-MEM-002 — Severity: Critical
**Finding.** No retention policy on episodic or semantic memory; data accumulates indefinitely.
**Recommendation.** Define policies per section 7.2; implement TTL and compaction; backfill cleanup for existing data.
**Owner.** ai-platform-eng + compliance, sprint N+1.

### AGT-MEM-003 — Severity: Critical
**Finding.** No deletion path; privacy requests are handled manually and incompletely.
**Recommendation.** Deletion path per section 7.5; quarterly test with synthetic entity.
**Owner.** ai-platform-eng + compliance, sprint N+1.

### AGT-MEM-004 — Severity: High
**Finding.** Episodic memory not scoped at storage layer; cross-conversation leakage possible.
**Recommendation.** Scoping enforced at storage (composite key `(tenant_id, conversation_id)`); integration tests verify.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-MEM-005 — Severity: High
**Finding.** Short-term context grows unboundedly; quality degrades on long loops.
**Recommendation.** Running summary or structured-extraction strategy per section 3.2/3.3.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-MEM-006 — Severity: High
**Finding.** Memory bugs cannot be debugged from traces; no memory-state observability.
**Recommendation.** Memory-state span attributes per section 8.1; memory diff tools per section 8.4.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-MEM-007 — Severity: High
**Finding.** Long-term memory is written by the agent; hallucinated conclusions are now persistent.
**Recommendation.** Audit and clean; switch to curated-source-only write per section 4.3; agent conclusions go to episodic with provenance.
**Owner.** ai-platform-eng + content team, sprint N+2.

### AGT-MEM-008 — Severity: High
**Finding.** Episodic memory load includes raw turns of long conversations; context-window saturation.
**Recommendation.** Summary + recent strategy per section 5.2 Pattern B; cap raw turn count.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MEM-009 — Severity: Medium
**Finding.** Semantic memory has no provenance; can't trace where a fact came from.
**Recommendation.** Provenance per entry (conversation_id, turn_id, source); included in span when read.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MEM-010 — Severity: Medium
**Finding.** Episodic compaction does not extract stable facts to semantic; useful information is lost on TTL expiration.
**Recommendation.** Compaction job per section 5/7; extracts stable facts to semantic profile; documents what was extracted.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MEM-011 — Severity: Medium
**Finding.** Memory eval is absent; multi-turn accuracy, cross-conversation memory, and forgetting compliance are not tested.
**Recommendation.** Memory eval surface per section 8.5; included in eval gate.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MEM-012 — Severity: Medium
**Finding.** Memory-write tool descriptions don't guide the agent on when to write; over-eager writes accumulate noise.
**Recommendation.** Tool description discipline per section 6.4; describe when to write and when not.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MEM-013 — Severity: Medium
**Finding.** Semantic memory schema is unstable; field additions break existing reads.
**Recommendation.** Versioned schema; backward-compatible additions; migration on read.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-MEM-014 — Severity: Medium
**Finding.** Long-term retrieval is not instrumented; can't tell which queries returned which results.
**Recommendation.** [retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) pattern.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-MEM-015 — Severity: Low
**Finding.** Memory architecture is undocumented; new engineers cannot understand the four-type decomposition.
**Recommendation.** Per-feature memory design document; updated on changes.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-MEM-016 — Severity: Low
**Finding.** Memory store growth is not monitored; cost surprises happen quarterly.
**Recommendation.** Growth-rate metric per store; dashboard panel; alert on > 2x prior-month rate.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-MEM-017 — Severity: Low
**Finding.** Memory store backups are not tested; recovery from corruption is unverified.
**Recommendation.** Quarterly backup-restore test on a non-production replica.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-MEM-018 — Severity: Low
**Finding.** Memory schemas are not shared across teams; multiple teams reinvent similar shapes.
**Recommendation.** Shared schema library for common shapes (conversation log, user profile, note entry).
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team designing memory for a new agent:

- [ ] **Sprint 0 — taxonomy analysis.** Apply section 2.5. Which memory types does the use case require?
- [ ] **Sprint 0 — memory design document.** Per-type implementation, retention policy, deletion path, observability plan.
- [ ] **Sprint 1 — short-term context strategy.** Running summary or structured extraction per section 3.
- [ ] **Sprint 1 — long-term retrieval.** Tools surface to the RAG pipeline; not built fresh.
- [ ] **Sprint 2 — episodic implementation.** If needed. Schema, load path, write path, scoping.
- [ ] **Sprint 2 — semantic implementation.** If needed. Structured profile; explicit write tools.
- [ ] **Sprint 3 — retention policies.** TTLs, compaction job, expiration handling.
- [ ] **Sprint 3 — deletion path.** End-to-end deletion across all stores; tested.
- [ ] **Sprint 3 — observability.** Memory state in spans, memory diff tools.
- [ ] **Sprint 4 — eval.** Multi-turn, cross-conversation, forgetting compliance, cross-tenant isolation.
- [ ] **Ongoing — quarterly review.** Retention policies, store growth, deletion-path test, eval results.

For a team with a problematic existing memory implementation:

- [ ] **Sprint 0 — audit.** What memory exists? Which type does each implementation serve? What's the overlap, the gaps?
- [ ] **Sprint 1 — fix the deletion path.** Privacy risk is the most urgent.
- [ ] **Sprint 1 — fix retention.** Stop unbounded growth.
- [ ] **Sprint 2 — disable auto-write of semantic.** Replace with deliberate writes.
- [ ] **Sprint 2 — clean hallucinated entries.** Audit semantic records; remove agent-generated unverified facts.
- [ ] **Sprint 3 — implement missing types.** What the taxonomy analysis revealed.
- [ ] **Sprint 4 — observability.** Make memory debuggable.

A team that completes the sequence has a memory architecture that's right-sized, debuggable, compliant, and cost-controlled. A team that defers it has a system that grows toward a privacy incident or a quality crisis.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 5 (memory engineering).
- [agent-loop-design.md](./agent-loop-design.md) — runner that holds short-term context and dispatches memory tools.
- [tool-architecture.md](./tool-architecture.md) — memory-write tools follow the standard tool architecture.
- [error-and-partial-failure.md](./error-and-partial-failure.md) — memory-store failures handled with the same disciplines.
- [agent-evals.md](./agent-evals.md) — memory eval patterns.
- [agent-observability.md](./agent-observability.md) — trajectory observability that includes memory state.
- [rag-engineering/ingestion-pipeline-engineering.md](../rag-engineering/ingestion-pipeline-engineering.md) — long-term memory ingestion pipeline.
- [rag-engineering/retrieval-engineering.md](../rag-engineering/retrieval-engineering.md) — long-term memory retrieval.
- [rag-engineering/chunking-engineering.md](../rag-engineering/chunking-engineering.md) — long-term memory chunking decisions.
- [observability-and-telemetry/retrieval-instrumentation.md](../observability-and-telemetry/retrieval-instrumentation.md) — retrieval span shape used by long-term and episodic retrieval.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — agent-step spans that carry memory state.
- Sibling repo: [ai-architecture-reference-architecture/data-architecture-for-ai/vector-store-architecture.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/data-architecture-for-ai/vector-store-architecture.md) — vector store architectural patterns that back semantic / episodic / long-term implementations.
- Sibling repo: [ai-architecture-reference-architecture/multi-tenancy-and-isolation/per-tenant-vector-namespacing.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/multi-tenancy-and-isolation/per-tenant-vector-namespacing.md) — tenant scoping for memory stores.
- Sibling repo: ai-security-reference-architecture `privacy-and-compliance/` — privacy and retention requirements that govern memory design.
- MemGPT (Packer et al., 2023) — research on hierarchical memory for LLM agents; informative on the layered-memory pattern.
