# Multi-Agent Coordination

> **Audience.** Tech leads and architects evaluating multi-agent topologies for a feature. Engineers maintaining an existing multi-agent system. Anyone tempted by a multi-agent framework demo to assume their feature warrants it. **Scope.** When multi-agent is genuinely warranted (rare), the supervisor/worker pattern, hand-off vs delegation, shared-state vs message-passing, the operational realities and the over-engineering trap. Not the single-agent loop runner (see [agent-loop-design.md](./agent-loop-design.md)). Not multi-agent security threats (sibling [ai-security-reference-architecture]). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Multi-agent is the second-most-overdone pattern in 2026 (the first being agent-when-workflow-would-work). Framework demos make it look attractive — six specialised agents pass messages, debate, refine, and produce a polished output. Production reveals: six agents are six times the cost, six times the latency, sixty times the failure modes (compounding across agents), and a debugging surface no engineer can read end-to-end.

The vast majority of features for which a team reaches for multi-agent are better served by:

- A single agent with a richer tool catalogue.
- A workflow whose LLM steps are specialised (different prompts, different models per step).
- A hybrid (workflow skeleton with one agent step) where the agent step has the flexibility it needs.

There are genuinely multi-agent-shaped problems — they exist — but they are uncommon, and the operational tax is severe enough that the decision should require explicit justification. A multi-agent system that is justified is engineered carefully; one that isn't justified shouldn't ship.

This document is opinionated about four things:

1. **The default for "multi-agent" is "no."** Single agent first; multi-agent only when a single agent provably cannot fit. The bar for "provably" is high: the team has built and operated the single-agent version and identified specific limitations that only multi-agent solves.
2. **Topology drives operational properties.** The two viable production topologies are supervisor/worker and pipeline. Peer-to-peer / debate / mesh topologies are research-interesting and operationally hostile.
3. **State-sharing mechanism is a design decision.** Shared state vs message-passing have different consistency properties, different failure modes, different observability requirements. Pick deliberately.
4. **Multi-agent multiplies the operational burden.** Every discipline from single-agent — loop budgets, tool architecture, memory, error handling, observability, eval, cost control — applies to every agent in the system. Then there's a coordination layer on top. Plan for the multiplied investment.

Structure: (2) the two questions — is multi-agent warranted, what topology; (3) the over-engineering trap; (4) supervisor/worker pattern; (5) pipeline pattern; (6) hand-off vs delegation; (7) shared state vs message-passing; (8) multi-agent observability and debugging; (9) cost and budgets; (10) worked Meridian example (the team chose not to build multi-agent; the rationale is the example); (11) anti-patterns; (12) findings; (13) adoption checklist; (14) references.

---

## 2. The two questions

Multi-agent decision-making collapses to two questions, asked in order.

### 2.1 Question 1 — Is multi-agent warranted?

The criteria for a genuine multi-agent fit:

- **The problem decomposes into distinctly different roles.** Not "specialised prompts" — distinctly different roles that benefit from separate context, separate memory, separate tool catalogues, and separate eval surfaces. Examples: a "planner" agent that builds a high-level plan and a "executor" agent that fills in steps; a "researcher" agent that gathers information and a "writer" agent that synthesises.
- **Single-agent with sub-prompts is insufficient.** A single agent with a "for this step, behave as a planner; for the next, as an executor" prompt could not produce the quality outcome — the team has tried.
- **The roles' interactions are bounded.** Not "the agents debate freely." Specific hand-off points; specific message shapes.
- **The cost multiplier is acceptable.** N agents = roughly N× the cost of a single agent on the same task (often more because of message passing overhead). The business value warrants it.
- **The latency multiplier is acceptable.** N agents = roughly N× the latency (serial) or max(latencies) + overhead (parallel where parallelism exists). If sub-5s latency matters, multi-agent is rarely viable.
- **The operational investment is feasible.** The team has the budget to build the coordination layer, the observability, the eval, the runbook — and to maintain them.

A "yes" on all six is rare. A "no" on any one is a sufficient reason to use a different shape.

### 2.2 Question 2 — What topology?

If multi-agent is warranted, two topologies have production track records:

- **Supervisor / worker (hub-and-spoke).** One supervisor agent decides which worker to invoke; workers execute; results return to the supervisor. The supervisor maintains the plan and coordinates.
- **Pipeline.** A fixed sequence of agents; each consumes the prior's output and produces its own. Simple, predictable.

Topologies to avoid in production: peer-to-peer (any agent can call any other; coordination is emergent), debate/critique (agents argue back and forth until convergence; unbounded), mesh (multiple supervisors; cross-supervisor coordination).

The first two are debuggable and bounded. The latter three are research-grade.

### 2.3 The decision matrix

| Criterion | Single agent | Pipeline | Supervisor / worker |
| --- | --- | --- | --- |
| Roles distinct enough | N/A | One per pipeline stage | One per worker |
| Coordination needed | None | Sequential | Dynamic dispatch |
| State sharing | Implicit | Pipeline payload | Supervisor's plan |
| Failure handling | Per-tool, per-loop | Per-stage + plan-level | Per-worker + supervisor-level |
| Cost envelope | 1× | N× (one per stage) | 1× (supervisor) + per-call N (workers) |
| Debugging | Single trajectory | Sequential trajectory | Supervisor trajectory + per-worker trajectory |
| Operational burden | Baseline | Baseline + pipeline | Baseline + supervisor + per-worker |

The decision matrix is for after question 1 returned "yes." If question 1 returned "no," all three multi-agent columns are out of scope.

---

## 3. The over-engineering trap

The pattern that produces wasted engineering quarters.

### 3.1 The trap, narrated

The team has a feature. They sketch an architecture: "a planner agent and an executor agent." They build it. Three weeks in, the planner is producing detailed plans the executor refuses to execute (the executor's prompt was tuned independently and they're not aligned). They add a "reviewer" agent that arbitrates. The reviewer's outputs are sometimes inconsistent, so they add a "memory" agent that holds the canonical state. Now there are four agents. Each agent's eval is on its own; the overall outcome's eval is poor because the agents' individual evals don't predict whole-system quality.

Six weeks in, an observability tool shows: the whole-system cost is 7× the single-agent baseline they prototyped first. The latency is 4×. The quality is comparable. They rip it out and ship the single agent.

### 3.2 Why the trap is common

- **Framework marketing.** Multi-agent frameworks present the topology as the headline feature. The marketing implies multi-agent is "more advanced" than single-agent.
- **Demo bias.** Demos show six agents collaborating elegantly on a curated task. Production traffic is not curated.
- **Architectural elegance bias.** A diagram with multiple roles looks impressive in a design review. A diagram with one box doesn't.
- **Specialisation intuition.** "Different prompts for different roles" feels right. The intuition is right; the conclusion (separate agents) doesn't follow.
- **Recovery bias.** When a single agent has quality issues, the team assumes adding more agents will fix it. Usually it doesn't; it just spreads the problem.

### 3.3 The corrective practice

Before any multi-agent design:

1. **Ship a single-agent baseline first.** Get production data on quality, cost, latency, failure modes. The baseline is the comparison point.
2. **List the specific limitations.** Where does the single agent fail? Be specific: which input class, which outcome quality dimension, which cost or latency bound.
3. **Try single-agent fixes.** Better prompts, better tools, better memory, different model. Document what was tried.
4. **Only if single-agent fixes provably insufficient, design multi-agent.** Be explicit about which limitations multi-agent solves; specify the topology; commit to the operational investment.

The discipline takes time. It is much cheaper than building the multi-agent system that gets ripped out.

### 3.4 The "specialised prompts" alternative

Often the desire for multi-agent reduces to: "the same agent calls a tool, but for this sub-task I want it to behave differently."

This is what tool-specific prompts (or workflow steps with their own prompts) solve. A single agent can call a `summarize_clinical_notes` tool whose implementation includes an LLM call with a specialised summarisation prompt. The tool is the "specialist." The agent doesn't multiply.

### 3.5 The "agents as tools" alternative

A pattern that's misleadingly called multi-agent but isn't: a single agent has a tool that, internally, runs a small LLM-loop. The tool is presented to the model as a tool; the internal loop is implementation detail.

This is single-agent. The "internal agent" inside the tool is not interacting with the outer model directly; it's a sub-routine the outer model invokes. Clean, simple, no coordination layer needed. If you find yourself reaching for multi-agent, check whether you actually want this pattern.

---

## 4. Supervisor / worker pattern

The most operationally tractable multi-agent topology.

### 4.1 The shape

- **One supervisor agent.** Maintains the plan, decides which worker to invoke next, integrates worker results. Sees the user's intent at the top level.
- **N worker agents.** Each is a specialist with its own tool catalogue and prompt. Invoked by the supervisor; returns a result.
- **Coordination layer.** Code that dispatches worker invocations on the supervisor's command; aggregates results; enforces budgets.

```
supervisor → coordinator → worker_A → coordinator → supervisor
                       → worker_B → coordinator → supervisor
                       → worker_C → coordinator → supervisor → final answer
```

### 4.2 The supervisor's responsibilities

- Read the user's input.
- Form a high-level plan (which workers to call, in what order, with what arguments).
- Dispatch worker calls via the coordinator.
- Integrate worker results into the plan's state.
- Decide when to terminate (the plan is complete, the budget is exhausted, escalation is appropriate).
- Produce the final answer.

The supervisor's loop is a normal agent loop (per [agent-loop-design.md](./agent-loop-design.md)). Its tools include `dispatch_worker(worker_name, input)` for each worker plus standard agent tools (escalate, complete).

### 4.3 The worker's responsibilities

- Receive a specific sub-task.
- Execute the task using its specialised tools.
- Return a structured result to the coordinator.
- Has its own budgets (turn, cost, time).

Each worker is, internally, a normal agent loop. From the outside, it looks like a function call: input in, output out.

### 4.4 The coordinator's responsibilities

The coordinator is code, not an agent:

- Validates the dispatch request (worker_name exists; input matches expected schema).
- Enforces per-worker budgets and per-overall-plan budgets.
- Logs the dispatch (span per worker invocation).
- Handles worker failure (returns structured error to supervisor).
- Aggregates worker cost into the overall plan's cost.
- Enforces concurrency limits (max N workers running in parallel).
- Enforces hand-off and delegation rules (per section 6).

### 4.5 Worker design

Each worker is a focused agent:

- Narrow tool catalogue (5–10 tools per worker).
- Specialised system prompt.
- Specific success criteria (return value schema).
- Own budgets, tighter than the supervisor's overall budget.

Workers should be the size of "agent" features that would be small standalone agents. If a worker has 30 tools and unbounded scope, it's not a worker — it's a second supervisor, and the design is mesh, not supervisor/worker.

### 4.6 Supervisor / worker observability

- **One trace per top-level invocation.**
- **Supervisor span tree.** The supervisor's loop produces a trajectory; each `dispatch_worker` call is a sub-span.
- **Worker trajectory.** Each worker's loop produces its own trajectory; rooted under the dispatch span.
- **Coordinator events.** Span attributes record dispatch decisions, budget enforcement, failure handling.

The trace is hierarchical: supervisor → coordinator → worker. An on-call engineer can navigate the hierarchy to find the failure point.

### 4.7 Supervisor / worker cost engineering

Cost decomposes:

- Supervisor's per-loop cost (typically lower; the supervisor does coordination, not heavy work).
- Per-worker cost (each worker's own cost envelope; tracked separately).
- Aggregate cost (sum of supervisor + all workers).

Budgets apply at each level. The plan-level budget caps the total; per-worker budgets cap each worker's contribution; the supervisor's budget caps the coordination overhead.

### 4.8 When supervisor / worker is the right shape

- Clear hub of coordination (the supervisor's role is genuinely "coordinate," not "execute").
- Workers have distinct specialisations that justify separate eval surfaces.
- Worker invocations are bounded (the supervisor calls each worker N times for small N, not 100s).
- Quality benefits from worker-specific prompts that wouldn't fit in a single agent.

If these aren't true, single-agent or hybrid is likely a better fit.

---

## 5. Pipeline pattern

A fixed sequence of agents; each consumes the prior's output.

### 5.1 The shape

```
input → agent_A → output_A → agent_B → output_B → agent_C → final_output
```

Each agent is a stage. The sequence is fixed at design time (it does not branch dynamically; if it does, it's becoming supervisor/worker).

### 5.2 When pipeline is the right shape

- The task is a sequence of transformations.
- Each transformation benefits from a specialised prompt and possibly a different model.
- The transformations are ordered (B requires A's output).
- The shape doesn't branch — every input goes through the same stages.

Example: a research-and-write pipeline. Agent A researches (search tools); agent B drafts (writing prompt); agent C critiques and revises (review prompt). Each stage is specialised.

If branching is needed (some inputs skip a stage; some inputs need additional stages), the shape is workflow with LLM steps, not pipeline-of-agents.

### 5.3 Pipeline vs workflow

A workflow with LLM steps is a pipeline. The distinction this section makes is: when each stage is a *full agent with its own loop* (not just a single LLM call), the pipeline-of-agents pattern applies.

Pipeline-of-agents adds N agent loops to the cost and latency. A workflow with N LLM steps is much cheaper. The pipeline-of-agents shape warrants its overhead only when each stage genuinely benefits from a loop (multiple tool calls, multiple turns of refinement).

### 5.4 Pipeline observability and failure handling

- One trace per pipeline invocation.
- One sub-trace per stage.
- Per-stage failure handling: a stage may produce an error envelope; the pipeline either short-circuits or invokes a per-stage fallback.
- Cost decomposes per stage.

### 5.5 The "stages drift" failure mode

In production, each pipeline stage's prompt evolves independently. After six months, stage B is consuming output stage A no longer produces (a schema field was renamed). The pipeline silently degrades.

Corrective: schema-pin the inter-stage interface. Each stage publishes a schema for its output; the next stage consumes per the schema; schema changes are coordinated.

---

## 6. Hand-off vs delegation

Two ways for one agent to involve another. The semantic difference affects state ownership and failure handling.

### 6.1 Hand-off

Agent A is in control. Agent A finishes its part and hands the entire interaction to agent B. After hand-off:

- Agent A is done; it does not see further activity.
- Agent B is now in control; it sees the conversation context and continues.
- The user-facing experience may shift in tone (different agents have different voices); often a hand-off message announces the switch.

Example: a triage agent classifies an incoming request and hands off to a specialist agent. The specialist completes the interaction.

**Semantics.**
- One agent is active at a time.
- State ownership transfers.
- Failure of the receiving agent does not return to the handing-off agent.

### 6.2 Delegation

Agent A invokes agent B to do a sub-task; agent B returns a result; agent A continues. After delegation:

- Agent A is still in control.
- Agent B was a sub-routine.
- Agent A integrates B's result into its own state and proceeds.

This is the supervisor/worker pattern (section 4).

**Semantics.**
- One agent is the principal; others are sub-routines.
- State ownership stays with the principal.
- Failure of the delegate returns to the principal; the principal handles.

### 6.3 The choice

| Property | Hand-off | Delegation |
| --- | --- | --- |
| Who is in control | Receiver after hand-off | Caller stays in control |
| State ownership | Transfers | Stays with caller |
| Failure handling | New agent's responsibility | Caller's responsibility |
| User-facing experience | May change | Consistent (caller is voice) |
| Complexity | Lower (no integration of results) | Higher (caller integrates) |

Hand-off is right when the role genuinely changes (triage agent → specialist; intake agent → fulfilment agent) and the user accepts the new agent's voice. Delegation is right when one agent owns the interaction and uses others as specialists.

### 6.4 The "round-trip" pattern

A hybrid: agent A hands off to agent B; agent B completes; agent B hands back to agent A for closure. This is two hand-offs in sequence. Use sparingly; the user-facing voice changes twice, and the second hand-off is often unnecessary (delegation would have been simpler).

### 6.5 Failure handling at hand-off

When the receiver fails, the system needs a recovery path. Options:

- **Hand-off-back to sender.** The receiver hands back with the failure context; the sender resumes. Requires the sender's state to be reconstructible.
- **Escalate to human.** The receiver fails; the failure escalates rather than handing back. Often simpler.
- **Retry with a different receiver.** A backup specialist is invoked.

The pattern is part of the design; not left to the model's discretion.

### 6.6 Hand-off observability

The trace must clearly show hand-off points:

- Span event: `handoff.initiated` with `from_agent`, `to_agent`, `handoff_reason`.
- New trace context for the receiver, but the receiver's trace is linked to the original.
- Aggregate metric: hand-off rate per agent; common destinations.

---

## 7. Shared state vs message passing

How agents share information. Two paradigms; pick deliberately.

### 7.1 Shared state

A central state store is read and written by all agents:

```
                    ┌─────────────┐
                    │ shared state │
                    └─────────────┘
                       ↑   ↑   ↑
                       │   │   │
              ┌────────┘   │   └────────┐
              │            │            │
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ agent A  │ │ agent B  │ │ agent C  │
        └──────────┘ └──────────┘ └──────────┘
```

Each agent reads the current state, decides what to do, writes updates back. Consistency is managed at the store level (versioning, transactions, locks).

**Pros.**
- All agents see the latest state.
- Coordination is implicit (the state is the contract).
- Easy to reason about "what does each agent see."

**Cons.**
- Concurrent writes need consistency mechanisms (locks, versioning, conflict resolution).
- The state store becomes a single point of failure.
- Schema changes affect every agent.
- The state can grow unboundedly (per the [memory-engineering.md](./memory-engineering.md) disciplines).

**When to use.** Supervisor/worker where the supervisor's plan is the state and workers update specific fields. Pipeline where the pipeline payload is the state.

### 7.2 Message passing

Agents communicate via messages:

```
agent A ──message──> agent B
agent B ──message──> agent C
agent C ──message──> agent A
```

Each agent has its own state. Messages carry the necessary context.

**Pros.**
- Decoupling: each agent's state is local; no central dependency.
- Natural for async or distributed deployment.
- Failure isolation: a failing state store doesn't take down the system.

**Cons.**
- Each message carries context, increasing payload size.
- Coordination is explicit (every interaction is a message).
- Failure handling per message (retries, dead-letter queues).
- Debugging requires reading the message log, not just a state snapshot.

**When to use.** Pipeline patterns naturally. Distributed multi-agent (where agents are in different processes). Patterns with asynchronous interactions.

### 7.3 Hybrid: shared state for coordination, messages for sub-tasks

The most common production shape: a shared state for the supervisor's plan + dispatch messages to workers carrying the worker's input. Workers return results as messages; the coordinator integrates into the shared state.

The hybrid captures the consistency benefits of shared state for the plan (one canonical truth) and the decoupling benefits of message-passing for worker invocations.

### 7.4 State schema discipline

Whichever mechanism, the schema is part of the design:

- Versioned schema; backward-compatible additions; major versions for breaking changes.
- Field semantics documented (what does this field mean; who writes it; who reads it).
- Validation at write (the store rejects invalid writes; the message receiver rejects malformed messages).
- Test coverage for schema evolution.

### 7.5 The "agents share LLM context" anti-pattern

A pattern that looks like state-sharing but isn't: agent A's full conversation history is dumped into agent B's prompt as context. Agent B is overwhelmed by context that's mostly irrelevant; cost balloons; quality drops.

Corrective: extract structured state from agent A; pass structured state to agent B; do not pass raw conversation history. Structured handoff scales; raw context dumping doesn't.

---

## 8. Multi-agent observability and debugging

The hardest observability problem in the practice.

### 8.1 The traceability principle

A single top-level user request should have a single trace_id that all participating agents (supervisor, workers, pipeline stages) share. The trace is a tree:

```
trace_id: abc
├── supervisor.loop
│   ├── supervisor.turn(1)
│   │   ├── llm.call
│   │   └── tool.dispatch_worker (worker_A)
│   │       ├── worker_A.loop
│   │       │   ├── worker_A.turn(1)
│   │       │   │   ├── llm.call
│   │       │   │   └── tool.fetch_patient
│   │       │   └── worker_A.turn(2)
│   │       │       └── llm.call (final)
│   │       └── coordinator.result
│   └── supervisor.turn(2)
│       └── llm.call (final)
```

The hierarchy is the debugging surface. An engineer paged on a multi-agent issue should be able to open the top span and navigate down.

### 8.2 Per-agent metrics

Per agent in the system:

- Invocation count.
- Per-invocation cost (mean, p50, p95, p99).
- Per-invocation latency.
- Per-invocation turn count.
- Failure rate by category (per [error-and-partial-failure.md](./error-and-partial-failure.md)).
- Escalation rate.

The metrics are sliced per agent; coordination-layer metrics (cross-agent dispatches, hand-offs, messages) are separate.

### 8.3 Aggregate metrics

Per multi-agent feature:

- End-to-end cost.
- End-to-end latency.
- End-to-end success rate.
- Per-stage failure breakdown (which agent failed; what category).
- Hand-off rate; delegation rate.

### 8.4 The hand-off span

Per hand-off:

- `handoff.from_agent`, `handoff.to_agent`, `handoff.reason`, `handoff.context_passed_tokens`.
- Trace linkage: the receiver's trace is rooted under the hand-off span (or shares the trace_id depending on implementation).
- Aggregate: hand-off matrix (from-agent × to-agent) shows the system's coordination patterns.

### 8.5 The dispatch span

Per worker dispatch (in supervisor/worker):

- `dispatch.worker_name`, `dispatch.input_tokens`, `dispatch.budget`.
- Linkage: the worker's trajectory is rooted under the dispatch span.
- Aggregate: per-worker invocation count; per-worker cost contribution.

### 8.6 Alerts specific to multi-agent

- **Coordination-layer failure spike.** Dispatches failing; hand-offs failing.
- **Per-worker cost spike.** A single worker contributing disproportionately to cost.
- **Hand-off loop.** Agent A hands to B which hands to A — circular. Should be impossible in a well-designed system; an alert catches the bug.
- **Aggregate latency spike.** End-to-end latency exceeds threshold even though individual agents are within limits — coordination overhead pathology.

### 8.7 Debugging without the trace

Sometimes the trace is incomplete (sampling, an agent ran in a different process, the observability backend dropped data). Then debugging from logs and the agents' final states is the fallback. Pattern: every agent logs its incoming request and outgoing response with a correlation_id; aggregate by correlation_id to reconstruct.

### 8.8 The on-call burden

Multi-agent doubles or triples the per-incident time. The runbook is per-agent and per-coordination-layer. Triage starts at the aggregate metrics; drills down into the agent and span tree; identifies the failing component; applies the per-component runbook.

The on-call team should be sized for multi-agent's complexity. If you can't staff it, you can't run multi-agent.

---

## 9. Cost and budgets for multi-agent

Cost discipline that's harder than single-agent.

### 9.1 The cost decomposition

```
total_cost = coordinator_overhead + sum(agent_costs)
agent_cost = sum(turn_costs) per agent
turn_cost = sum(LLM call cost + tool call cost) per turn
```

Per-feature dashboards must show this decomposition. Otherwise, cost incidents are diagnosed slowly.

### 9.2 The budget hierarchy

| Budget level | Enforces |
| --- | --- |
| Per-LLM-call | The model wrapper's call cap |
| Per-turn (within an agent loop) | The runner's per-turn budget |
| Per-agent (whole loop) | The runner's agent-loop budget |
| Per-worker dispatch | The coordinator's per-dispatch cap |
| Per-feature (whole multi-agent invocation) | The feature-level cap |
| Per-tenant (across invocations) | The tenant-level cap |

A breach at any level triggers the appropriate response. The hierarchy mirrors single-agent but adds the worker-dispatch and feature levels.

### 9.3 Cost-aware termination

When the feature-level budget is near exhaustion:

- Supervisor receives a signal "budget near exhaustion."
- Supervisor produces a graceful final answer (with whatever it has) or escalates.
- Coordinator refuses further worker dispatches.

The graceful-termination behaviour is engineered; it's not the model's discretion.

### 9.4 The cost-incident response

When a cost spike alert fires:

- Open the per-agent cost breakdown; identify which agent is the contributor.
- Open the per-feature trace samples; identify the pattern (one agent looping, one worker called repeatedly, coordination overhead).
- Apply per-agent or per-coordination-layer mitigation.
- Roll back if the cause is a recent change.
- Post-incident: tighten the budget; add observability; update eval.

The pattern is the same as single-agent but with more places to look.

### 9.5 Multi-agent cost vs single-agent baseline

When evaluating whether to keep a multi-agent system, compare:

- Multi-agent cost per outcome.
- Single-agent cost per outcome (if the single-agent baseline is available).
- Quality difference.
- Latency difference.
- Operational burden difference.

If the multi-agent system isn't producing materially better outcomes for the additional cost, retire it. The discipline applies quarterly, not once.

---

## 10. Worked Meridian example

Meridian's care-coordinator is a *single-agent* system. The decision not to build multi-agent is the example — the rationale, the alternative considered, the periodic re-evaluation.

### 10.1 The multi-agent design that was considered

Early product discussions sketched a multi-agent topology:

- **Triage agent.** Receives the clinician's question; classifies; routes.
- **Clinical-data agent.** Specialised on patient-record retrieval and clinical-context synthesis.
- **Scheduling agent.** Specialised on appointment and care-plan operations.
- **Escalation agent.** Specialised on packaging context for human care managers.

The pitch: each agent has a focused tool catalogue, a specialised prompt, and an own eval surface. The team would build distinct quality bars per role.

### 10.2 The single-agent baseline that was built first

Per the discipline in section 3.3, the team built a single-agent baseline before committing to multi-agent. The single agent had:

- The combined tool catalogue (12 tools spanning clinical data, scheduling, escalation).
- A system prompt with sub-sections covering each role.
- Standard loop budgets (per [agent-loop-design.md](./agent-loop-design.md)).
- The Meridian observability and eval surfaces.

The baseline ran in production for two quarters.

### 10.3 The decision data

After two quarters, the team compared the baseline's quality, cost, latency, and operational burden against the projected multi-agent design.

| Metric | Single-agent (actual) | Multi-agent (projected) |
| --- | --- | --- |
| End-to-end cost per request | $0.18 | $0.45–0.85 (3–5× from coordination + multiple loops) |
| End-to-end latency p50 | 4.2s | 11–18s (sequential coordination) |
| End-to-end latency p99 | 18s | 45–80s |
| Quality on golden eval | 92.4% | Projected 93–94% (uncertain) |
| Operational burden | 1 runbook, 1 alert family, 1 eval surface | 4 runbooks, 4 alert families, 4 + 1 eval surfaces |
| On-call team size requirement | Current team | +1 engineer (estimated) |

The projected quality gain (1–2 percentage points) did not justify the 3–5× cost, 3–4× latency, and roughly 4× operational burden.

### 10.4 The fixes that closed the gap

The single agent's quality issues that motivated the multi-agent consideration were addressed with single-agent fixes:

- **Per-role-section prompts.** The system prompt had clear sub-sections; the agent followed them.
- **Specialised inner LLM tools.** For specific sub-tasks (summarising a clinical note, packaging context for escalation), the team built tools whose implementation included a focused inner LLM call with a specialised prompt. The agent calls these as tools (per section 3.5's "agents as tools" pattern). No multi-agent needed.
- **Memory + structured state.** The structured-state extraction (per [memory-engineering.md](./memory-engineering.md) section 3.3) helped the agent maintain coherence across the multi-role conversation.
- **Per-tool eval.** Each tool's behaviour was independently eval'd (per [agent-evals.md](./agent-evals.md)); the eval surfaces approximated the per-role eval the multi-agent design would have had.

The quality on the golden eval rose to 94.1% over the two quarters — comparable to the multi-agent projection at a fraction of the cost.

### 10.5 The quarterly re-evaluation

The team reviews the shape quarterly. After 14 months, the conclusions:

- Single-agent quality is stable at 93–95%; no specific failure mode has emerged that multi-agent would address.
- The combined tool catalogue (now 14 tools) is well-curated; the agent's selection accuracy is > 96%.
- The structured-state extraction handles the multi-role coherence.
- The operational burden of the single agent is sustainable for the current team.

No quarter has produced a "switch to multi-agent" recommendation. The decision document is updated each quarter with the data.

### 10.6 The one case where multi-agent was warranted (and not built)

A separate Meridian product — the analytics-warehouse copilot for payer-customer analysts — has a different shape consideration. The analyst's question may require:

- SQL generation (one specialisation).
- Schema exploration (a different specialisation, with deep knowledge of the warehouse's metadata).
- Result interpretation and narrative generation (a third specialisation, focused on producing analyst-grade explanations).

The team considered multi-agent. They built the single-agent baseline. The single agent's quality on result interpretation was below target (the model was technically correct but the explanations were too dry for the analyst audience).

The team's intervention: a hybrid shape (per [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md)) — outer workflow handles SQL generation and execution; the result-interpretation step has an inner LLM call with a specialised "narrative analyst" prompt. Not multi-agent; the inner call is a workflow step, not a separate agent loop.

This is the agents-as-tools / specialised-workflow-step alternative resolving what looked like a multi-agent problem. The team explicitly recorded the decision: "we considered multi-agent; the hybrid achieved the quality with a fraction of the operational burden."

### 10.7 What Meridian's experience generalises

The pattern across Meridian:

- Single-agent first; multi-agent considered but rarely built.
- Specialised behaviour achieved through specialised tools, specialised inner LLM calls, or hybrid workflow steps — not separate agents.
- Per-tool eval and structured-state extraction substitute for the per-agent eval surfaces a multi-agent design would offer.
- Quarterly re-evaluation is documented; the conversation stays alive even when the answer hasn't changed.

The generalisation: most "we need multi-agent" intuitions reduce to "we need specialisation" — which has lighter-weight implementations.

---

## 11. Anti-patterns

### 10.1 "Multi-agent because the framework supports it"

The team picks a multi-agent framework, builds multi-agent, ships. Production reveals over-engineered system.

**Corrective.** Decision tree per section 2; single-agent baseline first; multi-agent only when justified.

### 10.2 "Mesh / peer-to-peer / debate topology in production"

Agents communicate freely; coordination is emergent. Unbounded; undebuggable.

**Corrective.** Supervisor/worker or pipeline only. Research-grade topologies stay in research.

### 10.3 "Workers are oversized"

Each "worker" has 30 tools and unbounded scope. The system is two supervisors, not supervisor/worker.

**Corrective.** Per section 4.5: narrow tool catalogues, focused responsibilities, own budgets.

### 10.4 "No coordinator; agents call each other directly"

Agents have tools that invoke other agents directly. Coordination logic is scattered across agent prompts.

**Corrective.** Coordinator is code per section 4.4. Agents don't dispatch each other; the coordinator dispatches.

### 10.5 "Raw context passed at hand-off"

Agent A's full conversation history goes to agent B. Agent B is overwhelmed; cost balloons.

**Corrective.** Structured hand-off context per section 7.5. Extract; don't dump.

### 10.6 "Hand-off loops"

Agent A hands to B; B hands to A; A hands to B again. Infinite recursion.

**Corrective.** Hand-off depth limit; alert; the design should not permit cycles.

### 10.7 "Per-agent observability without aggregate observability"

Each agent has its own trace and metrics; no end-to-end view. Cross-agent incidents take hours to diagnose.

**Corrective.** Shared trace_id; hierarchical span tree per section 8.1. End-to-end metrics.

### 10.8 "Multi-agent cost not decomposed"

Cost is a single number. Cost incidents reveal nothing about which agent contributed.

**Corrective.** Per-agent cost decomposition per section 9.1. Dashboard panel.

---

## 12. Findings (sprint-assignable)

### AGT-MULTI-001 — Severity: Critical
**Finding.** Multi-agent system was built without a single-agent baseline; cannot demonstrate the multi-agent investment is justified.
**Recommendation.** Build the single-agent baseline; compare quality, cost, latency, operational burden; retire multi-agent if not justified.
**Owner.** ai-platform-eng + feature-team, sprint N+1.

### AGT-MULTI-002 — Severity: Critical
**Finding.** Mesh / peer-to-peer / debate topology in production; coordination emergent and unbounded.
**Recommendation.** Restructure to supervisor/worker or pipeline per section 2.2; if neither fits, retire and use single-agent or workflow.
**Owner.** ai-platform-eng + feature-team, sprint N+1.

### AGT-MULTI-003 — Severity: Critical
**Finding.** Hand-off loops possible in design (A→B→A); cost incidents traceable to loops.
**Recommendation.** Hand-off depth limit per section 6; alert on hand-off events; design audit to remove cycles.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-MULTI-004 — Severity: High
**Finding.** No coordinator layer; agents dispatch each other directly via tool calls.
**Recommendation.** Introduce coordinator (code, not agent) per section 4.4; agents call coordinator tools, not each other.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-MULTI-005 — Severity: High
**Finding.** Workers are oversized (each has 25+ tools and unbounded scope).
**Recommendation.** Narrow worker catalogues per section 4.5; if a worker is supervisor-sized, the design is mesh and warrants restructure.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-MULTI-006 — Severity: High
**Finding.** Hand-offs pass raw conversation history; receiver context inflated.
**Recommendation.** Structured hand-off context per section 7.5; structured-extraction at hand-off boundary.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-MULTI-007 — Severity: High
**Finding.** No aggregate observability; per-agent traces exist but no end-to-end view.
**Recommendation.** Shared trace_id; hierarchical span tree per section 8.1; aggregate metrics.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-MULTI-008 — Severity: High
**Finding.** Cost is not decomposed by agent; incident triage cannot identify the contributor.
**Recommendation.** Per-agent cost decomposition per section 9.1; dashboard panel.
**Owner.** ai-platform-eng + ops, sprint N+3.

### AGT-MULTI-009 — Severity: Medium
**Finding.** Per-feature budget is the only enforcement; per-agent and per-worker-dispatch budgets are absent.
**Recommendation.** Budget hierarchy per section 9.2; cost-aware termination per section 9.3.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MULTI-010 — Severity: Medium
**Finding.** Coordinator-layer failures are not specifically observable; failures appear as "dispatch failed" with no detail.
**Recommendation.** Coordinator events per section 4.4 / 8.5; alert on coordination failures per section 8.6.
**Owner.** ai-platform-eng + ops, sprint N+3.

### AGT-MULTI-011 — Severity: Medium
**Finding.** Multi-agent system has no per-worker eval; quality issues identified only at the end-to-end level.
**Recommendation.** Per-worker eval surfaces + end-to-end eval; the two together diagnose where quality issues originate.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-MULTI-012 — Severity: Medium
**Finding.** State-sharing schema is not versioned; schema changes silently break consumers.
**Recommendation.** Versioned schema per section 7.4; backward-compatible additions; major versions for breaking.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-MULTI-013 — Severity: Medium
**Finding.** Multi-agent runbook is not per-component; on-call response is improvised.
**Recommendation.** Runbook with sections per agent, plus a coordination section.
**Owner.** ops + ai-platform-eng, sprint N+4.

### AGT-MULTI-014 — Severity: Medium
**Finding.** Hand-off events are not recorded as span events; hand-off matrix cannot be computed.
**Recommendation.** `handoff.*` span events per section 6.6; aggregate matrix dashboard.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-MULTI-015 — Severity: Low
**Finding.** Pipeline stages drift; inter-stage schema is implicit.
**Recommendation.** Inter-stage schema pinning per section 5.5; coordinated schema changes.
**Owner.** ai-platform-eng + feature-team, sprint N+4.

### AGT-MULTI-016 — Severity: Low
**Finding.** Quarterly comparison of multi-agent vs single-agent baseline is not performed.
**Recommendation.** Quarterly review per section 9.5; retire multi-agent if no longer justified.
**Owner.** ai-platform-eng + tech-lead-of-feature, sprint N+5.

### AGT-MULTI-017 — Severity: Low
**Finding.** Hand-off vs delegation choice was not deliberate; the team mixes both in ad-hoc ways.
**Recommendation.** Design audit per section 6; document each cross-agent interaction's kind.
**Owner.** ai-platform-eng + feature-team, sprint N+5.

### AGT-MULTI-018 — Severity: Low
**Finding.** Agents-as-tools alternative (per section 3.5) was not considered; the team built multi-agent for what was a sub-routine pattern.
**Recommendation.** Re-examine the design with the agents-as-tools alternative.
**Owner.** ai-platform-eng + feature-team, sprint N+5.

---

## 13. Adoption sequencing checklist

For a team considering multi-agent for a new feature:

- [ ] **Sprint 0 — single-agent baseline.** Build and ship the single-agent version.
- [ ] **Sprint 0 — limitations analysis.** Specific, measured limitations of the single-agent baseline.
- [ ] **Sprint 0 — single-agent fixes attempted.** Prompts, tools, memory, model.
- [ ] **Sprint 0 — decision document.** Multi-agent warranted? Topology choice? Operational investment scoped?
- [ ] **Sprint 1 — topology implementation.** Supervisor/worker or pipeline; coordinator layer.
- [ ] **Sprint 1 — worker design.** Narrow scope, focused tools, own budgets.
- [ ] **Sprint 1 — shared state or message schema.** Versioned.
- [ ] **Sprint 2 — observability.** Shared trace_id, hierarchical spans, per-agent + aggregate metrics.
- [ ] **Sprint 2 — budget hierarchy.** Per-level enforcement; cost-aware termination.
- [ ] **Sprint 2 — failure handling.** Per-agent + coordination-layer; runbook sections.
- [ ] **Sprint 3 — eval.** Per-worker + end-to-end.
- [ ] **Sprint 3 — production canary.** Behind feature flag; comparison to single-agent baseline.
- [ ] **Sprint 4 — promotion or retirement.** Promote if multi-agent demonstrably better; retire if not.
- [ ] **Ongoing — quarterly review.** Compare to single-agent baseline; consider retirement.

For a team with an existing problematic multi-agent system:

- [ ] **Sprint 0 — single-agent baseline.** Build the comparison.
- [ ] **Sprint 0 — audit.** Topology, coordinator presence, worker scope, hand-off pattern, observability, cost.
- [ ] **Sprint 1 — fix the worst gap.** Often the missing coordinator or the mesh topology.
- [ ] **Sprint 2 — observability.** End-to-end visibility.
- [ ] **Sprint 3 — eval per worker.** Find quality issues.
- [ ] **Sprint 4 — decision.** Keep, restructure, or retire.

A team that completes the sequence has either a working multi-agent system whose investment is justified, or has retired the multi-agent system in favour of a simpler shape. Both outcomes are valid; the third outcome — keeping a multi-agent system that's not justified — is not.

---

## 14. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — broader practice; section 7 (multi-agent considerations).
- [agent-vs-workflow-decision.md](./agent-vs-workflow-decision.md) — single-agent vs workflow vs hybrid decision; precedes the multi-agent question.
- [agent-loop-design.md](./agent-loop-design.md) — single-agent runner; each agent in a multi-agent system is built on this.
- [tool-architecture.md](./tool-architecture.md) — each agent's tools follow the standard discipline.
- [memory-engineering.md](./memory-engineering.md) — each agent's memory; shared state is itself a memory store.
- [error-and-partial-failure.md](./error-and-partial-failure.md) — failure-handling disciplines apply per agent.
- [agent-observability.md](./agent-observability.md) — trajectory observability with multi-agent hierarchy.
- [agent-cost-control.md](./agent-cost-control.md) — cost-control with multi-agent decomposition.
- [agent-evals.md](./agent-evals.md) — per-agent and end-to-end eval.
- [agent-anti-patterns.md](./agent-anti-patterns.md) — including the "multi-agent for everything" anti-pattern.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — span shape; the hierarchical multi-agent trace builds on it.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/agent-topologies.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/agent-topologies.md) — architectural topology catalogue (sibling repo's perspective on multi-agent shapes).
- LangGraph, CrewAI, AutoGen — multi-agent frameworks; engineering principles here apply across them.
- "Multi-agent reinforcement learning" (Stone & Veloso, 2000) — early academic frame for multi-agent coordination problems.
- "The Sciences of the Artificial" (Simon, 1969) — chapter on bounded rationality, informative on coordination overhead.
