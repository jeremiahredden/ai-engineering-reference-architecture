# Agent Engineering Playbook

> **Audience.** Engineers and tech leads shipping a production agent — first one, or one that is misbehaving. **Scope.** The engineering practice of shipping agents: the agent-vs-workflow decision, loop design, tool architecture, memory, error handling, multi-agent coordination, agent-specific evals, observability, cost control, and on-call disciplines. Not architectural pattern selection (the architecture sibling repo's `reference-patterns/` owns that); not adversarial agent security (the [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture) sibling's `agent-security/` folder owns that). **Worked client.** Meridian Health, the fictional regulated healthcare SaaS used across the sibling reference architecture repos.

---

## 1. Why this document exists

Agents are the most operationally demanding shape of AI system in 2026. The single LLM call has bounded cost, bounded latency, and one failure mode. The agent has unbounded cost (loops), unbounded latency (slow tools, retries, replanning), dozens of failure modes (wrong tool, wrong arguments, infinite loop, partial completion, lost context, hallucinated intermediate results, side-effects-on-retry), and a debugging surface that is read-the-trace rather than run-the-step. They also produce production outages that look new to teams accustomed to deterministic systems: a single misbehaving agent can burn six figures in a weekend, take real-world actions that are expensive to reverse, and degrade quietly in ways that no liveness probe catches.

So this playbook is structured around the disciplines that prevent those outcomes. The first section forces the agent-vs-workflow decision before any engineering investment. The middle sections walk through the engineering primitives — loop design, tool architecture, memory, error and partial-failure handling, observability, cost control. The later sections cover multi-agent coordination (which is rare and over-used), agent-specific evals (which are harder than single-call evals and more important), the operational depth required for on-call, the eight anti-patterns I see most often, and the worked Meridian Care Coordinator example.

A team that adopts this playbook will produce agents that are bounded in cost, debuggable from traces, recoverable from partial failures, gated by HITL on real-world side effects, and operable on the same on-call rotation as the rest of the platform. None of that comes for free; all of it comes from engineering discipline that scales the agent's blast radius proportionally to the team's ability to operate it.

---

## 2. The agent-vs-workflow decision

The single most consequential agent engineering decision is whether to build an agent at all. Many features that get scoped as "agent" projects are workflow projects with LLM steps embedded. Workflows are bounded, debuggable, and dramatically cheaper to operate. Agents are open-ended, expensive, and harder to debug. The default should be workflow; the burden of justification should be on the agent.

### 2.1 The decision criteria

Build a **workflow with LLM steps** when:

- The set of sub-tasks is known in advance and stable across runs.
- The order of sub-tasks is fixed or follows a small, enumerable set of branches.
- Each sub-task has a well-defined input and output.
- The total cost and latency need to be predictable.
- The system's behavior must be deterministically reproducible from a release artifact for audit purposes.

Build an **agent** when:

- The set of sub-tasks depends on intermediate results that cannot be planned upfront.
- The number of steps required varies meaningfully across runs.
- The system must adapt its approach based on what it discovers as it works.
- The latency and cost budgets allow for variable, multi-step execution.

### 2.2 The hybrid pattern (most often correct)

A surprising number of real systems are best built as a workflow *whose skeleton is fixed* but which has one or two steps where an agent loop is contained inside a single workflow stage. The outer workflow guarantees the overall structure (orchestration, audit, fan-out, rollback); the inner agent loop handles the genuinely open-ended sub-step (a multi-hop retrieval question, a tool-use sequence whose path depends on what the tools return).

This hybrid pattern is the architecture for both the Meridian Care Coordinator (workflow-shaped overall with agent loops for the multi-step coordination tasks) and the Meridian analytics-warehouse copilot (workflow-shaped SQL generation with an agent loop for schema retrieval and query revision). It usually outperforms "make the whole thing an agent" along every dimension that matters.

### 2.3 The trap

The trap is "everything is an agent." The agent feels powerful and general; the workflow feels old-school. Resist. The agent costs 5–15x what the workflow costs, is 5–10x slower, and is dramatically harder to operate. A team that picks the agent shape when a workflow would do has chosen to pay that tax forever.

**Cross-link.** Architecture: `reference-patterns/agent-topologies.md` (coming).

---

## 3. Loop design

Once an agent is the right shape, the loop is the central engineering object. Get it right and the rest of the agent is tractable; get it wrong and everything downstream — observability, cost control, error handling — is patching around it.

### 3.1 Loop shapes

The canonical agent-loop shapes:

- **ReAct.** Model thinks, decides on an action (tool call), observes the result, repeats. The default loop shape for most modern agents. Simple, well-understood, well-supported.
- **Plan-then-execute.** Model produces a plan upfront, then executes the steps. Useful when the planning needs frontier-model capability but the execution can be cheaper.
- **Reflection / self-critique.** After producing an output, the model critiques itself and revises. Adds cost and latency; earns it on quality-sensitive workloads.
- **Supervisor / worker.** A supervisor agent decides which worker to invoke; each worker is a specialized sub-agent. Adds complexity, earns it when worker specialization is real.
- **Tree-of-thought / search.** Explores multiple solution paths in parallel; selects best. Expensive; useful in research-shaped problems.

Pick one. Single shape per agent in a single bounded context. A loop that tries to mix shapes ("ReAct for tool calls, reflection on every output, plan-then-execute for new conversations") becomes ungovernable.

### 3.2 Explicit budgets

Every agent loop has four budgets that must be set explicitly. Setting them too late — after a runaway-cost incident — is the standard learning sequence; setting them at design time avoids it.

| Budget | Default starting value | Behavior on breach |
|---|---|---|
| Turn budget | 10 turns | Loop terminates; final answer is the supervisor's best-available summary; escalation flag set |
| Cost budget | $0.50 per single interaction | Loop terminates immediately; graceful failure response; page on-call |
| Time budget | 60 seconds per interaction | Loop terminates; partial result returned with timeout flag; page on-call |
| Tool-call budget | 12 tool calls | Loop terminates; same handling as turn budget |

The numbers above are illustrative; calibrate to the workload. The discipline is having all four — many agents have a turn budget and nothing else, which means a 6-turn loop that makes 10 tool calls per turn at $0.30 per tool call can still consume $18 before any limit triggers.

### 3.3 Termination conditions

The loop terminates when one of:

- The model returns a final answer (no more tool calls).
- A budget is breached.
- An explicit termination tool is called (some patterns have a `done` or `escalate` tool).
- A non-recoverable error occurs (provider permanent failure, authentication failure).

A loop without explicit termination conditions is the classic runaway-cost pattern. The model gets confused, keeps calling tools, and the only thing stopping it is when somebody notices the dashboards.

### 3.4 State checkpointing

For agents that run more than a few seconds, the loop state should be checkpointable. The pattern:

```
After every turn:
  - serialize the loop state (turn history, retrieved context, tool-call log,
    intermediate decisions, budget remaining)
  - persist to durable storage keyed by trace ID
  - emit a checkpoint event
On any failure:
  - the last checkpoint is the recovery point
  - resume from the checkpoint or terminate with the checkpoint as the partial result
```

Without checkpointing, a single transient failure costs the entire loop's progress. With it, even a 10-turn agent that fails on turn 9 retains 8 turns of useful work that can be inspected, resumed, or returned as a partial result.

This is also the basis for the durable-workflow pattern (Temporal, AWS Step Functions, durable function frameworks) which is the cleanest implementation for non-trivial agents.

---

## 4. Tool architecture

The tool surface is the most leveraged piece of agent engineering. A poorly designed tool surface produces a confused agent no matter how good the model is. A well-designed tool surface lets a smaller, cheaper model perform well above its weight. The two highest-leverage tool-engineering investments are (a) naming and (b) error-return design.

### 4.1 Tool naming

The model selects tools by reading their names and descriptions. The selection happens *before* any code runs. So the tool's name and description are the agent's interface to the tool.

- **Names should describe the action, not the implementation.** `look_up_drug_interactions` is good; `query_fda_spl_graph` is bad. The model can reason about the action; it cannot reason about the implementation.
- **Descriptions should specify both what the tool does and when it should be used.** A two-sentence description naming the use case and the input/output contract beats a long description that lists every possible application.
- **Descriptions should disambiguate from sibling tools.** If three tools could plausibly retrieve the same thing, the descriptions must explain when to use each. Otherwise the model picks randomly.

A common diagnostic when an agent is making wrong tool calls is to read the tool descriptions and ask "would I, with no knowledge of this system, know which tool to call here?" If not, the fix is the descriptions.

### 4.2 Argument schemas

Tool arguments are described to the model via JSON Schema (or equivalent). The schema is part of the model's prompt — bloat hurts, ambiguity hurts more.

- **Use enumerated values where possible.** A `status` field with `enum: ["draft", "approved", "sent"]` is much easier for the model to call correctly than a free-text `status: string`.
- **Use structured types over freeform strings.** A `patient_context` object with named fields beats a `patient_info: string` blob.
- **Describe every field with one sentence.** Fields without descriptions get hallucinated values.
- **Validate strictly at the boundary.** A tool that accepts arguments that do not match its schema can produce real-world side effects with wrong inputs. Validate, reject, return an error the agent can read.

### 4.3 Error-return design

When a tool fails, what it returns to the agent determines whether the agent can recover. Two patterns:

- **The agent-friendly error.** A structured response with `success: false`, an `error_type` enum, a `human_readable_message` for the model to reason over, and a `suggested_recovery` field if applicable. The model sees a failure it can think about.
- **The thrown exception.** The model sees a stack trace or an empty response and has no signal to act on. Agents handling thrown exceptions either retry blindly or give up.

Use the agent-friendly error pattern. Translate exceptions at the tool boundary into structured responses the agent can read.

Example:

```json
{
  "success": false,
  "error_type": "missing_authorization",
  "human_readable_message": "This patient is at a hospital you do not have access to. Verify the patient ID, or escalate to a clinician at that hospital.",
  "suggested_recovery": "verify_patient_id_or_escalate"
}
```

### 4.4 Side-effects and idempotency

Tools that take real-world side effects (write to a database, send a message, charge a card, create an order) need engineering discipline that read-only tools do not:

- **Default to HITL.** Side-effect tools should require human approval before execution unless there is a specific reason not to (and that reason should be documented and reviewed).
- **Idempotency keys.** Side-effect tools accept an idempotency key from the agent; identical calls with the same key are deduplicated. Prevents the "agent retried and double-charged" failure mode.
- **Compensating actions.** For tools whose side effects cannot be undone (an email was sent), the agent's design must accept that retries are not safe. For tools where compensation is possible (an order can be cancelled), the compensating action is itself a tool, and the recovery path is engineered.
- **Audit logging at the tool boundary.** Every side-effect tool call is logged with: agent state at call time, arguments, authorization decision, result, downstream effect. This is the trail that supports both debugging and compliance.

### 4.5 Tool registry as platform component

In a multi-feature platform, tools are best managed as a centralized registry with declarative authorization, audit logging, and HITL contracts. The pattern:

- Each tool registers with the registry on platform startup.
- Each tool declares its authorization requirements (roles, scopes, tenant scope).
- Each tool declares its side-effect class (read-only, draft-only, side-effect-with-HITL, side-effect-immediate).
- The registry enforces authorization on every call, logs every call, and refuses tool calls that violate the declared class (a side-effect tool registered without an HITL path is rejected at registration time).

This pattern keeps tool authorization out of calling code — the agent loop does not need to know which tools require approval, because the registry knows. It is also the surface where audit and platform-level policy enforcement attach.

**Cross-link.** Architecture: `guardrails-and-policy-architecture/tool-call-authorization.md` (coming).

### 4.6 MCP vs local function decision

Model Context Protocol (MCP) servers are a useful pattern when:

- Tools need to be shared across multiple agents or services.
- Tools have an independent deployment lifecycle.
- Tools need to be developed in a different language than the agent.

Local function calls (in-process tools) are the right pattern when:

- Tools are tightly coupled to the agent.
- Latency matters and the MCP-over-network overhead is meaningful.
- The operational complexity of a separate service is not justified.

Most production agents in 2026 use a mix — high-traffic, latency-sensitive tools as local functions; shared, deployable tools as MCP servers.

---

## 5. Memory engineering

Agent memory is overloaded — it can mean four different things, with four different implementations. Conflating them is the most common cause of memory failures.

### 5.1 The memory taxonomy

- **Short-term / context memory.** Verbatim conversation history within a single session. Implementation: just include in the prompt up to the context budget; summarize older turns when the budget is approached.
- **Long-term episodic memory.** Information about past interactions for the same user, across sessions. Implementation: structured event log (decisions, escalations, unresolved items), retrieved on demand rather than included by default.
- **Long-term semantic memory.** Knowledge the agent has accumulated about the world or the domain. Implementation: a knowledge store (often a vector store) that the agent retrieves from like any other retrieval surface.
- **Working memory.** The intermediate state of the current task — what has been retrieved, what tools have been called, what the plan is. Implementation: the loop state checkpoint (section 3.4).

Each has different volatility, different lookup pattern, different cost, different staleness tolerance. Treating them as one bucket — "the agent has memory" — produces a system where the agent occasionally remembers things it should not, occasionally forgets things it should remember, and is hard to debug because the memory surface is not separable.

### 5.2 Short-term memory implementation

The default pattern:

1. The full verbatim history is included in the prompt while it fits in the context window's history budget.
2. When the history budget is approached (typically 70% of allocated tokens), a summarization step replaces the oldest turns with a running summary.
3. The most recent N turns (typically 3–5) are always verbatim — verbatim recent history is much higher signal than summarized recent history.

Implementation pitfall: summarization that loses the information needed for the next turn. Mitigated by keeping verbatim recent turns and by tuning the summarization prompt against an eval set of multi-turn cases.

### 5.3 Long-term episodic memory implementation

The structured event log pattern:

```
After every significant event in a session (escalation, decision, unresolved
item), append a structured record:
{
  event_id, user_id, session_id, timestamp, event_type, summary, references
}

When a new session begins for the same user, retrieve the last N events
plus any unresolved items; include in the system context.
```

Critical engineering details:

- **Scope episodic memory by identity, not just by tenant.** Two clinical staff working on the same patient should not share each other's episodic memory.
- **Episodic memory is not patient-data by default.** Track the staff member's pattern of work, not the patient's details (which live in the EHR and are retrieved as needed).
- **Define a retention policy.** Episodic memory is data; data has lifecycle.

### 5.4 Long-term semantic memory implementation

Semantic memory is essentially the retrieval surface (RAG). Treat it as such — vector store, embedding pipeline, retrieval client wrapper, the whole stack described in the [rag-engineering/](../rag-engineering/) sibling folder. The only agent-specific overlay: the agent has its own discretion about *whether to query semantic memory* (via a tool call) versus the deterministic retrieval used in non-agent RAG.

### 5.5 Memory failure modes

- **Hallucinated memory.** The agent claims to remember something it never knew. Cause: usually a summarization step that hallucinated content. Mitigation: never summarize without a source-document grounding.
- **Stale memory.** The agent remembers something that has since changed. Cause: episodic memory not invalidated when underlying state changed. Mitigation: time-stamp every memory record; the consuming prompt notes the time and treats older records as advisory rather than authoritative.
- **Contradicting memory.** Multiple memory records contradict each other. Cause: events were appended without conflict-resolution. Mitigation: when retrieving multiple records on the same topic, the prompt is given all of them and asked to resolve; for high-stakes decisions, conflict triggers HITL escalation.

---

## 6. Error and partial-failure handling

Agents fail more often and more interestingly than single-call systems. The engineering practice that distinguishes a usable agent from a brittle one is whether each failure class has a designed response.

### 6.1 The failure taxonomy

- **Transient provider error.** Model API returned 5xx or timed out. The call was not processed.
- **Permanent provider error.** Model API returned 4xx (rate limit exceeded with no retries possible, authentication, content-policy refusal). The call was rejected.
- **Tool transient error.** The tool's backing service returned 5xx or timed out.
- **Tool permanent error.** The tool's backing service returned a structured failure (not authorized, resource not found, bad arguments).
- **Tool returned garbage.** The tool succeeded but returned content that the agent cannot use (empty result, malformed output, off-topic content).
- **Agent reasoning failure.** The model returned a tool call with wrong arguments, or a final answer that does not address the question, or an empty / malformed response.
- **Partial-success failure.** A multi-step operation completed some steps successfully and failed on a later step. The failed step may have side effects to roll back.
- **Authorization failure.** The agent tried to do something it is not permitted to do.

### 6.2 The response pattern per class

| Failure class | Engineered response |
|---|---|
| Transient provider error | Retry with exponential backoff, bounded retry count (typically 3); after exhaustion, escalate as service degradation |
| Permanent provider error | Do not retry; surface to the agent as a structured tool-call failure if applicable; otherwise terminate the loop with a graceful failure |
| Tool transient error | Same as provider transient — bounded retry with backoff |
| Tool permanent error | Pass the structured failure to the agent; the agent decides whether to try a different approach or escalate |
| Tool returned garbage | Validate at the tool boundary; if validation fails, return a structured error to the agent; the agent reasons over the error |
| Agent reasoning failure | Repair-loop: re-prompt with the schema validation error; bounded retries (typically 2) before escalating |
| Partial-success failure | Compensating actions where available; HITL escalation where not; never silent failure |
| Authorization failure | Pass to the agent as a polite-refusal tool response; the agent informs the user; never silently skipped |

### 6.3 The retry decision tree

Retry is wrong more often than teams think. The decision tree:

```
Should I retry this failed step?
├── Is the failure transient? (network, 5xx, timeout)
│   └── If yes: retry with backoff (bounded)
│
├── Is the step idempotent? (same call produces same result, no side effects)
│   └── If no: do NOT retry — retry may double-execute the side effect
│
├── Has the tool returned a structured permanent error?
│   └── If yes: do NOT retry; pass to the agent
│
└── Default: do not retry; escalate
```

Most retry-related agent incidents trace to violating one of these — either retrying a non-idempotent step that took a side effect, or retrying a permanent error that will keep failing.

### 6.4 Compensating actions

For agents that take side effects across multiple steps, the engineering pattern is the saga: each step that takes a side effect has a compensating action that undoes it. If a later step fails, the compensations are executed in reverse order to leave the world in a consistent state.

In practice, full saga discipline is expensive. The pragmatic version: identify the side-effect steps that *must* be reversible (the order entry, the appointment booking), engineer compensation for those, and accept that side effects on less-critical paths (a draft message that gets discarded) do not need compensation.

For Meridian Care Coordinator: order drafts are not side effects (they go to a queue for HITL approval), so no compensation is needed. Approved orders are side effects, but they are taken by the clinician through the platform UI rather than by the agent directly, so the compensation responsibility lies with the clinician's workflow rather than the agent. The agent's design intentionally keeps side effects out of its critical path.

---

## 7. Observability for agents

Agents are observable through traces or they are not observable at all. Single-line logs do not capture the structure of an agent run; aggregated metrics tell you that something is wrong without telling you what. The trace of a single failing interaction is the debugging surface that makes the bug actionable.

### 7.1 Trace structure

One trace per top-level user request. Each turn is a span. Each tool call is a sub-span. Each LLM call (supervisor, worker, classifier, query rewriter) is a span. Retrievals are spans. The hierarchy lets you read the trace top-down and follow exactly what the agent did.

### 7.2 Attributes to capture

For every LLM call span:

- `prompt_version` — the versioned prompt artifact
- `model` — provider + model name
- `model_version` — the specific pinned version
- `model_params` — temperature, max_tokens, etc.
- `tokens_input` — split between cached and uncached
- `tokens_output`
- `cost_usd` — computed at request time
- `latency_to_first_token_ms`
- `latency_total_ms`
- `finish_reason`
- `tool_calls_present`

For every tool call span:

- `tool_name`
- `arguments_hash` — hash, not full args, to avoid log bloat
- `authorization_decision` — allowed, denied, requires-HITL
- `authorization_reason`
- `success`
- `error_type` — when failed
- `latency_ms`
- `idempotency_key` — when applicable

For every retrieval span:

- `query`
- `query_rewrite` — if applicable
- `retrieved_doc_ids` — full list
- `retrieval_scores` — per-doc, per-retriever
- `reranker_scores` — when applicable
- `embedding_model_version`
- `corpus_version`
- `latency_ms`

For each agent-loop span (the loop turn):

- `turn_number`
- `budget_remaining` — all four budgets
- `decision` — tool call vs final answer
- `loop_state_checkpoint_id` — if checkpointing

### 7.3 The trace as primary debugging surface

When a production failure is filed against the agent, the diagnostic discipline is:

1. Pull the trace by interaction ID.
2. Read the trace top-down. The structure tells you the path the agent took.
3. Find the span where things went wrong (a tool call with the wrong arguments, a retrieval that returned empty, a worker that returned a wrong answer).
4. The span's attributes tell you why (the prompt version that was used, the model that was used, the corpus version, the budget state).
5. Often the answer is "the prompt version was bumped yesterday and this is the regression" or "the corpus was refreshed and a chunk that used to be retrieved is no longer there." The trace makes the diagnosis concrete.

Without this practice, "the agent did something weird" is investigated by running new experiments and guessing. With this practice, the existing trace is the evidence.

### 7.4 Alerting

The agent-specific alerts (in addition to the standard latency / error-rate alerts):

- **Agent-loop runaway.** Loop reached turn budget. Page if frequent.
- **Cost runaway.** Single interaction or session exceeded cost circuit breaker. Page immediately.
- **Tool-authorization rejection rate elevated.** Spike in agent attempting unauthorized tool calls — could be prompt-injection, could be agent confusion, could be a tool definition change.
- **Quality SLI breach.** Judge-pass-rate dropped below threshold on the production stream. Page if breach persists.
- **Retrieval-empty rate elevated.** Agent is asking for retrievals that return nothing — likely a corpus problem or a query-rewriting regression.

**Cross-link.** Engineering: `observability-and-telemetry/agent-step-instrumentation.md`, `alerting-and-paging-design.md` (coming).

---

## 8. Cost control

Cost is the agent failure mode that surprises teams hardest. A 5-turn loop with hybrid retrieval and a reranker at $0.20 per loop scales to $200 with a 1,000-fan-out async task; a misbehaving agent that loops 100 times per request can burn through a daily budget in a single bad hour.

### 8.1 The cost-control stack

Four layers, all needed:

1. **Per-interaction cost budget.** Hard ceiling for a single interaction. Termination + graceful failure when breached.
2. **Per-session cost budget.** A user with 20 turns in a session should not consume 20x the per-interaction budget unbounded.
3. **Per-tenant daily cost budget.** Tenant-level circuit breaker for the case where many users are individually within budget but the aggregate exceeds plan.
4. **Per-feature daily cost budget.** Across all tenants, a single agent feature should not consume an unbounded share of the platform's budget.

Each layer fails open or fails closed independently; the operational decision depends on the feature.

### 8.2 The cost-as-circuit-breaker pattern

```
On every LLM call:
  - check the four budgets
  - if any is exceeded:
      - record breach
      - emit alert / page on-call (depending on severity)
      - return graceful failure to the agent (the next call is not made)
  - else: proceed, record cost incurred
```

The breach handling is what matters. "Continue but warn" is the same as no circuit breaker. The discipline is *stop spending*. Recovery is operational (raise the budget, investigate the cause, restart) rather than automatic.

### 8.3 Tier routing inside the loop

A common cost-control move: the supervisor uses an expensive model, the workers use cheaper models for their specialized subtasks, and only the genuinely-hard reasoning escalates back to the supervisor. The Meridian Care Coordinator's three-tier routing (Opus / Sonnet / Haiku) is the example.

The eval discipline: tier routing must be validated against the eval suite. The case where the cheap-tier worker silently produces lower-quality results is the regression that tier routing introduces.

### 8.4 Cost-attribution telemetry

Every span carries `cost_usd`. Aggregations support diagnostics:

- Cost per feature
- Cost per tenant
- Cost per user (for B2C)
- Cost per question-class
- Cost per agent-loop length

When a cost spike happens, the dashboard tells you whether it is one tenant, one feature, one question class, or a generalized increase. The diagnosis follows.

**Cross-link.** Engineering: `cost-and-finops/cost-budget-circuit-breaker.md`, `cost-and-finops/tier-routing-for-cost.md` (coming).

---

## 9. Multi-agent coordination (used sparingly)

Multi-agent systems are over-used. Most multi-agent designs are simpler as a single-agent loop with multiple tools, or as a supervisor / worker pattern with deterministic dispatch rather than agent-to-agent message passing.

### 9.1 When multi-agent is warranted

- The sub-domains are genuinely specialized and the specialization warrants separate prompts and possibly separate model tiers.
- The coordination pattern is well-bounded (supervisor / worker, not free-for-all).
- The orchestration overhead is justified by the parallelism or specialization gain.

The Meridian Care Coordinator is multi-agent in the supervisor / worker sense — but it is *one* supervisor with deterministic dispatch to specialized workers, not an open-ended swarm. The supervisor decides which worker to invoke; workers do not invoke each other.

### 9.2 When multi-agent is over-engineered

- Two agents are talking to each other because the team thought "agents collaborating" sounded interesting. The same work could be done by one agent with two tools.
- The handoff between agents is itself the failure mode — agent A produces output, agent B misinterprets, the chain breaks. Single-agent designs do not have this failure mode.
- The latency is the sum of two agent loops in series, the cost is the sum of two agent loops in series, and the eval surface is two agents instead of one.

### 9.3 Coordination patterns when multi-agent is the right shape

- **Supervisor / worker.** One supervisor dispatches deterministically to workers. Workers do not communicate with each other. Result aggregation happens at the supervisor. *This is the Care Coordinator's pattern.*
- **Pipeline.** A → B → C in sequence; each agent's output is the next agent's input. Useful for staged transformations. Failure modes are well-bounded; backwards communication is not allowed.
- **Map-reduce.** Many parallel agents process partitions; a single agent aggregates. Useful for fan-out workloads.

Patterns to be skeptical of:

- **Conversational multi-agent.** Two agents "negotiating." Almost always cheaper, faster, and better as a single agent reasoning with two tool sources.
- **Hierarchical with deep nesting.** Supervisor → sub-supervisor → workers. Coordination overhead exceeds benefit past two levels.

---

## 10. Agent-specific evals

Agent evals are harder than single-call evals along every dimension. The agent's trajectory is non-deterministic — the same input can lead to different paths. Outcome eval is necessary but not sufficient; trajectory eval is necessary but expensive.

The full discipline lives in the sibling [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md); the agent-specific overlays:

### 10.1 The three eval shapes for agents

- **Outcome eval.** Was the final answer correct? Necessary; insufficient — passes a case where the agent reached the right answer by luck.
- **Trajectory eval.** Were the intermediate steps reasonable? LLM judge scores the trajectory: was the tool selection right, were the retrievals likely to retrieve relevant content, was the planning coherent?
- **Step-budget eval.** Did the agent complete within turn / cost / time budget? Catches the agent that gets the right answer but burns 20 turns.

Run all three. Agent quality is multidimensional and a single number does not capture it.

### 10.2 Cost-aware sampling

A 300-case agent eval can cost $50–$300 to run. Run the full suite less often than you would for a single-call system. The fast suite for per-PR runs should be a stratified subset (~30 cases) chosen to cover the case classes that matter most.

### 10.3 Regression cases from production trajectories

Every observed agent failure in production becomes a permanent eval case. The pattern: an agent loop did something wrong → the trace is captured → the trace becomes a trajectory-eval case with "the agent should not have done this" as the criterion.

This is the highest-value source of agent eval cases, because it is composed of failure modes that have already happened in production rather than failure modes the team imagined upfront.

---

## 11. The Meridian Care Coordinator worked example

Applied to the Care Coordinator (see the architecture sibling's [reference-systems/meridian-care-coordinator.md](../../ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md) for system context):

### 11.1 Loop design

Supervisor / worker topology. Loop shape: ReAct at the supervisor level; the workers are stateless single-call invocations. Per-interaction turn budget: 8 turns (most interactions complete in 1–3). Cost budget: $0.50 per interaction, $1.50 per session. Time budget: 60 seconds. Tool-call budget: 10.

The supervisor's tools include: invoke the classifier, invoke the clinical-knowledge worker, invoke the drafting worker, query the drug-interaction graph (when classifier indicates), draft a patient message, draft an order, draft an appointment, escalate to a human, finalize the answer. Tool selection is deterministic for the dispatch-to-worker calls (classifier output determines which worker is called); only the side-effect tools and the final-answer decision involve real model choice.

### 11.2 Tool architecture

All tools registered in the platform tool registry. Tool naming follows the verb-noun pattern (`draft_patient_message`, `look_up_drug_interactions`). Argument schemas use enums where possible (`message_type: "follow_up" | "appointment_reminder" | "education"`). Errors are agent-friendly structured responses.

The side-effect tools (`draft_patient_message`, `draft_order`, `draft_appointment`) are registered as `side_effect_with_HITL` class — the registry enforces that they produce drafts rather than committing, and the drafts flow to the clinician's approval queue.

Tool boundary defense-in-depth: every tool call is authorized through the registry (role-based + tenant-scope-based), arguments are validated against the schema, results are logged at the doc-ID level for retrieval tools and at the draft-ID level for side-effect tools.

### 11.3 Memory

- **Short-term.** Verbatim history up to ~6K tokens of the history budget. Older turns are replaced by a Sonnet-tier running summary; the most recent 4 turns are always verbatim.
- **Long-term episodic.** Structured event log scoped per staff identity (not just per tenant). Tracks decisions, escalations, unresolved follow-ups. Retrieved at session start if the staff member is a returning user. No patient data in the episodic store by default.
- **Long-term semantic.** This is the clinical guidelines + protocols + drug-interaction-graph retrieval surface — treated as RAG.
- **Working.** Loop state checkpointed per turn; durable storage keyed by trace ID. Resumable on transient failures.

### 11.4 Error handling

- Provider transient: 3 retries with exponential backoff.
- Provider permanent (rate limit exceeded after retries): graceful failure with on-call page.
- Tool transient: 2 retries with backoff at the tool boundary.
- Tool permanent (authorization denied): structured response to the agent ("you do not have access to this patient's data"); agent informs the user.
- Tool garbage (retrieval returned empty for a question it should have answered): retry once with query rewriting; if still empty, escalate to human.
- Agent reasoning failure (wrong tool arguments): repair-loop with schema validation error returned; max 2 repair attempts before escalation.
- Side-effect failure: HITL is the boundary — drafts that fail to register in the approval queue are surfaced to the staff member who initiated them.

### 11.5 Observability

100% trace sampling. Every span carries the attributes listed in section 7.2. Traces export to Datadog with PHI redaction in the pipeline. Per-feature, per-tenant, per-question-class cost dashboards. Quality SLI (judge-pass-rate on a 10% production sample) updated hourly with alerting on drift.

The trace is the primary debugging surface for every Care Coordinator production incident. The discipline is: never investigate by re-running; always investigate from the captured trace first.

### 11.6 Cost control

The four-layer cost stack as designed: per-interaction $0.50, per-session $1.50, per-tenant-daily $50, per-feature $1,500/day across all tenants. Circuit-breaker breach is hard-stop with graceful failure response and on-call page.

Tier routing inside the loop: Opus for supervisor and clinical-knowledge worker, Sonnet for drafting, Haiku for classification and query rewriting. Eval validates that tier routing does not regress quality on the eval suite.

### 11.7 Multi-agent posture

Supervisor / worker, single supervisor, deterministic dispatch. Workers do not invoke other workers. The pattern was chosen specifically because it provides specialization benefits (clinical reasoning gets the strongest model; drafting gets a model tuned for instruction-following; classification gets the fastest model) without the operational cost of free-form agent-to-agent coordination.

---

## 12. Anti-patterns

The eight agent engineering anti-patterns I see most often, with the corrective pattern for each.

### 12.1 "Agent for everything"

Every feature becomes an agent because the agent feels powerful. The simple FAQ workflow that costs $0.005 with a single LLM call costs $0.15 with a 5-turn agent loop, takes 8 seconds instead of 2, and has worse quality on the simple cases because the agent over-thinks them.

**Corrective.** Apply the agent-vs-workflow decision (section 2). Most "agent" features are workflows. Most are best as workflows with one or two contained agent loops.

### 12.2 No turn budget

The loop has no explicit turn limit. The model gets confused, keeps calling tools, the loop runs for 40 turns before something else stops it. Cost and latency both blow out; trace is unreadable.

**Corrective.** Turn budget in section 3.2. 8–12 turns is the typical starting value. Termination on breach is mandatory.

### 12.3 No cost budget

The loop has a turn limit but no cost ceiling. Each turn makes 6 tool calls; each tool call costs $0.30; the 10-turn budget allows the agent to spend $18 per interaction. The first cost incident is the first time anyone realizes this.

**Corrective.** Cost-as-circuit-breaker (section 8). The four-layer stack: per-interaction, per-session, per-tenant-daily, per-feature-daily.

### 12.4 Side-effect tools without idempotency

The agent retries a transient failure on a tool that sent an email; the email goes twice. The agent retries a transient failure on a tool that created an order; the order is duplicated. The agent retries a transient failure on a tool that charged a credit card; the customer is double-billed.

**Corrective.** Idempotency keys at the tool boundary (section 4.4). Non-idempotent tools never retry; the failure escalates to HITL.

### 12.5 Memory that is actually just context stuffing

The team builds a "long-term memory" that is the full conversation history of every prior session, appended to the context. Cost explodes; quality degrades; the model gets confused by old context that is no longer relevant.

**Corrective.** Memory taxonomy in section 5.1. Long-term memory is structured (event log), not verbatim, and retrieved selectively rather than stuffed by default.

### 12.6 Retry-everything-on-error

The default error-handling code wraps every operation in a retry loop. Tools that should not retry retry; permanent errors are retried until budget is exhausted; the agent burns cost and time chasing failures that will not resolve.

**Corrective.** Retry decision tree in section 6.3. Retry only on transient errors and only for idempotent operations.

### 12.7 No trajectory observability

The agent's interactions are logged as request + response. When something goes wrong, the team cannot reconstruct what the agent did between the request and the response. Debugging is guessing.

**Corrective.** Trace structure in section 7. Every turn is a span; every tool call is a sub-span; attributes capture the state needed for diagnosis.

### 12.8 Human-in-the-loop as rubber stamp

The system has HITL for side effects on paper. In practice, the human clicks "approve" on everything without reading the drafts. The HITL is process theater; the agent's outputs ship un-reviewed.

**Corrective.** This is a process and UI design problem, not a pure engineering one. Mitigations: the HITL interface forces the human to engage with the content (display the draft, require a confirmation pattern that cannot be one-click defeated); the system surfaces side-effect drafts in a worklist queue that has its own SLA and review process; periodic random audits verify that human review is substantive. Where the HITL is rubber-stamped routinely, that is a signal that either the agent's drafts are reliable enough that HITL is not adding value (consider removing it) or that the workflow design is failing (re-engineer the HITL surface). Either way, surface the data and decide; do not leave the unaddressed rubber-stamp pattern in place as theater.

---

## 13. Findings (sprint-assignable)

The canonical agent-engineering findings I write into review documents. Each has an ID (`AGT-NNN`), severity, finding, recommendation, sprint owner template.

### AGT-001 — Severity: Critical
**Finding.** Agent loop has no cost circuit breaker; a runaway loop can consume unbounded budget.
**Recommendation.** Wire the four-layer cost stack (section 8.1); hard-stop on breach.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-002 — Severity: Critical
**Finding.** Agent loop has no explicit turn budget; loops can run indefinitely.
**Recommendation.** Set per-loop turn budget per section 3.2; terminate gracefully on breach.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-003 — Severity: Critical
**Finding.** Side-effect tools are exposed without HITL contract; agent can take real-world actions without human approval.
**Recommendation.** Re-register side-effect tools as `side_effect_with_HITL` class per section 4.5; refuse registration without HITL path.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-004 — Severity: Critical
**Finding.** Side-effect tools lack idempotency keys; retry can produce duplicate side effects.
**Recommendation.** Add idempotency-key contract per section 4.4; non-idempotent tools refuse retry.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-005 — Severity: High
**Finding.** Tool descriptions are vague; agent selects wrong tools at non-trivial rate.
**Recommendation.** Rewrite tool descriptions per section 4.1; eval the lift in tool-selection accuracy.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-006 — Severity: High
**Finding.** Tool errors are returned as thrown exceptions or empty responses; agent cannot reason about failures.
**Recommendation.** Adopt agent-friendly structured error pattern at tool boundary per section 4.3.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-007 — Severity: High
**Finding.** Agent uses a single "memory" bucket conflating short-term, long-term, episodic, and semantic memory.
**Recommendation.** Separate by taxonomy (section 5.1); implement each with the appropriate pattern.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-008 — Severity: High
**Finding.** Retry logic retries non-idempotent operations and permanent errors.
**Recommendation.** Implement the retry decision tree (section 6.3); validate against the failure-taxonomy table.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-009 — Severity: High
**Finding.** Agent traces are logs (request / response) rather than structured spans; debugging requires re-running rather than reading the trace.
**Recommendation.** Implement per-turn, per-tool-call, per-LLM-call spans with attributes per section 7.2.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-010 — Severity: High
**Finding.** Cost is not attributed per feature / tenant / question-class; cost incidents are not diagnosable.
**Recommendation.** Add cost attribution per section 8.4; surface in dashboards.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-011 — Severity: High
**Finding.** Loop state is not checkpointed; transient failures lose all loop progress.
**Recommendation.** Implement loop-state checkpointing per section 3.4; resumable on transient failures.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-012 — Severity: Medium
**Finding.** Agent eval uses outcome-only scoring; trajectory failures (over-budget, wrong-tool, unnecessary turns) are unmeasured.
**Recommendation.** Add trajectory eval and step-budget eval per section 10.1.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-013 — Severity: Medium
**Finding.** Multi-agent coordination uses agent-to-agent message passing where a single agent with multiple tools would do.
**Recommendation.** Refactor to single-agent with tool surface; preserve the parallelism via fan-out tool calls.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-014 — Severity: Medium
**Finding.** No alerts on agent-loop runaway (loop reached turn budget) or cost runaway (circuit breaker tripped).
**Recommendation.** Wire alerts per section 7.4; integrate with on-call paging.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-015 — Severity: Medium
**Finding.** Tool registry does not log authorization decisions to the audit trail.
**Recommendation.** Add authorization-decision events to the audit log; surface in traces.
**Owner.** ai-platform-eng + security-eng, sprint N+4.

### AGT-016 — Severity: Medium
**Finding.** Episodic memory is scoped per tenant rather than per identity; cross-staff memory leakage is possible.
**Recommendation.** Scope episodic memory by staff identity per section 5.3.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-017 — Severity: Medium
**Finding.** Production agent-loop failures are not promoted to permanent trajectory-eval cases.
**Recommendation.** Add the bug → regression-case process gate per section 10.3.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-018 — Severity: Low
**Finding.** Tier routing in the loop is not validated against the eval suite; cheaper-tier workers may silently regress quality.
**Recommendation.** Add per-tier eval comparison; gate routing changes on eval.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-019 — Severity: Low
**Finding.** Tool registry MCP-vs-local-function decisions are inconsistent across the platform.
**Recommendation.** Document the decision criteria (section 4.6); audit current tools and align.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-020 — Severity: Low
**Finding.** HITL approval surface is a one-click pattern that risks rubber-stamping.
**Recommendation.** Surface drafts in a structured worklist UI; require a content-engagement pattern (read confirmation, draft-summary attestation); audit HITL response time distribution.
**Owner.** ai-platform-eng + clinical-platform-design, sprint N+5.

---

## 14. Adoption sequencing checklist

For a team about to ship their first agent:

- [ ] **Sprint 0 — decide.** Apply the agent-vs-workflow decision (section 2). Document the result. If workflow, build the workflow; do not return.
- [ ] **Sprint 1 — bound.** Loop with explicit turn / cost / time / tool-call budgets. Cost-as-circuit-breaker wired before anything else. Termination on breach is mandatory.
- [ ] **Sprint 1 — observe.** Trace structure (section 7) implemented from day one. Re-instrumenting later is much harder.
- [ ] **Sprint 2 — tools.** Tool registry pattern. Side-effect tools as `side_effect_with_HITL` class. Agent-friendly error responses. Idempotency keys.
- [ ] **Sprint 2 — eval.** Outcome eval suite (per the eval-engineering playbook). Add trajectory eval as soon as enough trajectories exist to evaluate.
- [ ] **Sprint 3 — handle.** Failure-class response table (section 6.2). Retry decision tree (section 6.3). Compensating actions for reversible side effects.
- [ ] **Sprint 3 — memory.** Memory taxonomy (section 5.1) applied. Short-term + working memory at minimum. Long-term layers as the workload requires.
- [ ] **Sprint 4 — alert.** Alerts on agent-loop runaway, cost runaway, quality SLI breach, tool-authorization elevation.
- [ ] **Sprint 5 — operate.** Production trajectories become regression cases. Quarterly review of anti-patterns (section 12).
- [ ] **Ongoing — discipline.** Every production failure becomes a permanent eval case. Every fix lands with a trajectory-eval case to prevent recurrence.

A team that follows this sequencing has the foundation for an agent that can be operated on a normal on-call rotation. A team that skips early steps (the budget wiring, the trace structure, the HITL contract) carries the cost of retrofitting them forever.

---

## 15. References

- Anthropic, *Building agents with the Claude SDK*, Tool-use documentation, MCP specification.
- OpenAI, function-calling and agents documentation.
- LangGraph, CrewAI, AutoGen, Vercel AI SDK — framework-specific implementation references; engineering patterns here apply across them.
- LangSmith, Braintrust, Phoenix — agent-observability vendors.
- Temporal, AWS Step Functions, durable function frameworks — durable-workflow substrates for non-trivial agents.
- Sibling repo: [ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md](../../ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md) — system context for the worked example.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/](../../ai-architecture-reference-architecture/reference-patterns/) — pattern-selection rationale.
- Sibling repo: [ai-security-reference-architecture/agent-security/](https://github.com/jeremiahredden/ai-security-reference-architecture) — adversarial agent behavior, agent permission models, MCP server hardening, agent-specific threat modeling.
- This repo: [eval-engineering/eval-engineering-playbook.md](../eval-engineering/eval-engineering-playbook.md) for the eval discipline that agent-engineering depends on.
