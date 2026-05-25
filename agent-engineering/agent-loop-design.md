# Agent Loop Design

> **Audience.** Engineers building or refactoring the runner that drives an agent loop. Tech leads who want the loop's behavior to be predictable, debuggable, and bounded rather than emergent. **Scope.** The *engineering* depth on agent-loop construction — loop shape, state structure, termination conditions, checkpointing, runner interface. Pair with [agent-engineering-playbook.md](./agent-engineering-playbook.md) for the broader practice. Not the architectural topology choice (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `reference-patterns/agent-topologies.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

[agent-engineering-playbook.md](./agent-engineering-playbook.md) section 3 outlined loop design — loop shapes, explicit budgets, termination conditions, state checkpointing. This document is the depth on those four areas. The playbook tells you what to do; this tells you how to engineer it.

The runner — the small piece of code that drives the loop — is the engineering object. Its design determines whether the loop is predictable, debuggable, recoverable, and observable. A poorly designed runner produces an agent that surprises the team in production; a well-designed runner produces an agent the team can reason about.

This document is opinionated about three things:

1. **The loop is engineered, not improvised.** Every loop turn is a deliberate decision point with a structured input and output. The model's response is data, not control flow.
2. **State is explicit and durable.** Loop state lives in a structured object that can be serialized, inspected, and resumed. The model's internal "memory" is not the loop's state.
3. **Termination is multi-conditional and explicit.** The loop terminates when one of N conditions fires — final answer, budget breach, escalation tool, error. Each condition produces a distinct outcome the runner emits.

Structure: (2) the runner architecture; (3) loop shape implementations; (4) loop state structure; (5) termination conditions; (6) state checkpointing; (7) the model-response parsing layer; (8) integration with observability and circuit breakers; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The runner architecture

The runner is a small piece of code that drives the loop. It is not a framework; it is platform code under the team's control.

### 2.1 The runner's responsibilities

- Initialize loop state from the input.
- For each turn: build the prompt for this turn; call the LLM wrapper; parse the response; decide next action.
- Dispatch tool calls (through the tool registry); receive results; update state.
- Check budgets before each call; terminate if exceeded.
- Emit per-turn span (per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md)).
- Checkpoint state at defined points.
- Terminate with a defined outcome.

What the runner does NOT do:

- Build the prompt (the prompt assembler does).
- Make the LLM call directly (the LLM wrapper does).
- Authorize tool calls (the tool registry does).
- Compute cost (the wrapper does).
- Apply guardrails (the gateway does).
- Persist conversation history outside the loop (the application's memory layer does).

Each responsibility belongs to a single component. The runner orchestrates them.

### 2.2 The runner interface

```python
class AgentRunner:
    def __init__(
        self,
        topology: AgentTopology,        # see section 3
        budgets: AgentBudgets,          # turn / cost / time / tool-call limits
        llm_client: LLMClient,          # the LLM wrapper
        tool_registry: ToolRegistry,    # the tool dispatch surface
        checkpoint_store: CheckpointStore,
        observability: ObservabilityClient,
    ):
        ...

    def run(
        self,
        initial_input: AgentInput,
        prompts: AgentPromptBundle,     # versioned prompts per role
        model_routing: ModelRouting,    # which model for which role
        context: CallContext,           # tenant, user, feature, session, trace
        resume_from: CheckpointId | None = None,  # for recovery
    ) -> AgentResult:
        ...
```

The runner is injectable. Tests can substitute mock LLM clients and mock tool registries; production wires the real components. The dependency injection makes the runner testable in isolation.

### 2.3 The run method

```python
def run(self, initial_input, prompts, model_routing, context, resume_from=None):
    state = self._initialize_state(initial_input, resume_from)
    self._open_top_level_span(context, state)

    try:
        while not self._should_terminate(state):
            self._check_budgets(state)               # may raise BudgetBreach
            turn_span = self._open_turn_span(state)
            try:
                turn_input = self._build_turn_input(state, prompts)
                turn_output = self._execute_turn(turn_input, model_routing, context)
                state = self._update_state(state, turn_output)
                self._record_turn(turn_span, state, turn_output)
                self._checkpoint(state)
            finally:
                self._close_turn_span(turn_span)

        outcome = self._produce_outcome(state)
        self._close_top_level_span(outcome)
        return outcome

    except BudgetBreach as e:
        outcome = self._produce_terminated_outcome(state, e)
        self._close_top_level_span(outcome)
        return outcome
```

The structure is deliberate:

- The loop is a `while` with an explicit termination check at the top.
- Budgets are checked before each turn; breaches raise an exception that the loop catches and converts to a terminated outcome.
- Each turn is wrapped in its own span; the span is always closed (try/finally).
- State updates after every turn; checkpointing is automatic.
- The outcome is constructed deliberately; not whatever the last turn returned.

### 2.4 The platform discipline

The runner is the only path to agent loops. Application code does not implement its own loop; it instantiates the runner with the appropriate topology and budgets.

Lint rules can detect application code that loops over LLM calls (a `while` containing a call to the LLM client) and flag for review.

---

## 3. Loop shape implementations

Different topologies share the runner skeleton but differ in turn structure.

### 3.1 ReAct (single-agent)

```
Turn structure:
  1. Build prompt with conversation history + tool descriptions.
  2. Call LLM; receive response.
  3. Parse response:
     - If response has tool calls: dispatch them, collect results, add to state.
     - If response has final answer: terminate with `final_answer` outcome.
     - If response has no tool calls and no answer: terminate with `error` (malformed).
  4. Loop back to step 1.
```

Implementation:

```python
def _execute_turn_react(self, state, prompts, model_routing, context):
    messages = self._build_messages(state, prompts.single)
    response = self.llm_client.call(
        provider=model_routing.single.provider,
        model=model_routing.single.model,
        messages=messages,
        prompt_version=prompts.single.version,
        tools=self.tool_registry.get_available_tools(state.allowed_tool_set),
        context=context,
    )
    return TurnOutput(
        decision=self._parse_decision(response),
        llm_response=response,
        tool_results=self._dispatch_tools_if_any(response, state, context),
    )
```

### 3.2 Plan-then-execute

```
Initial turn: planning.
  Call planner LLM (typically more capable); receive structured plan.
Subsequent turns: execution.
  Each plan step is one turn.
  Call executor LLM (often cheaper) with the plan step + context.
  Track plan progress in state.
  Optionally: re-plan if execution surfaces a need.
```

Implementation: two distinct turn types. The first turn is the planning turn (calls the planner model with the planner prompt, parses a structured plan). Subsequent turns are execution turns (call the executor model with the current plan step). The runner's loop alternates based on state.

The state includes the plan; the runner can be paused after planning (for HITL approval of the plan) before execution begins.

### 3.3 Reflection / self-critique

```
Turn structure:
  1. Call LLM; receive proposed response.
  2. Call critic (often same LLM with critique prompt); receive critique.
  3. If critique flags issues:
     - Call LLM again with critique as context; receive revised response.
     - Optionally re-critique (bounded recursion).
  4. Return final response.
```

Implementation: the turn is logically one user-visible turn, but internally three LLM calls (initial + critique + revision). Each is a sub-span; the turn span wraps all three.

### 3.4 Supervisor / worker

```
Turn structure:
  1. Supervisor LLM analyzes input; decides which workers to invoke.
  2. Dispatch workers (potentially in parallel).
  3. Collect worker results.
  4. Supervisor LLM consolidates; decides:
     - Need more workers? Loop back to step 1.
     - Final answer? Terminate.
```

Implementation: the supervisor is the loop driver; workers are sub-spans (LLM calls or recursive sub-loops). State tracks: supervisor's current plan, worker dispatch status, worker results.

The Meridian Care Coordinator's supervisor / worker implementation: supervisor turn dispatches to classifier + clinical-knowledge in parallel (step 2), then dispatches to drafting if the supervisor decides drafting is needed (next turn), then consolidates and finalizes.

### 3.5 The shape's runtime indicator

The runner records the topology on every span (`ai.agent.topology` per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md)). Investigators can see at a glance which loop shape is running.

---

## 4. Loop state structure

State is the explicit object the runner maintains across turns. The structure:

### 4.1 The state fields

```python
class AgentState:
    # Initialization
    initial_input: AgentInput
    started_at: datetime
    trace_id: str

    # Conversation
    history: list[Message]              # all turns so far (LLM and tool exchanges)
    tool_call_log: list[ToolCallRecord] # what was called and what came back

    # Topology-specific
    plan: Plan | None                   # for plan-then-execute
    worker_results: dict[str, Any]      # for supervisor / worker

    # Budgets
    turn_number: int
    turns_remaining: int
    cost_remaining_usd: float
    time_budget_ms: int
    time_remaining_ms: int
    tool_calls_remaining: int

    # Control
    allowed_tool_set: list[str]         # which tools are available this turn
    pending_terminations: list[TerminationSignal]
```

The state is serializable (Pydantic model, plain dict, or similar). The runner mutates it deliberately; nothing else mutates it.

### 4.2 State transitions

Every turn produces a state update. The runner has a single state-update method that consumes the previous state and the turn's output:

```python
def _update_state(self, state: AgentState, turn_output: TurnOutput) -> AgentState:
    new_state = state.copy()
    new_state.turn_number += 1
    new_state.turns_remaining -= 1
    new_state.tool_calls_remaining -= len(turn_output.tool_results or [])
    new_state.cost_remaining_usd -= turn_output.llm_response.cost
    new_state.time_remaining_ms = self._compute_time_remaining(state)
    new_state.history.append(turn_output.llm_response.as_message())
    if turn_output.tool_results:
        new_state.history.extend([r.as_message() for r in turn_output.tool_results])
        new_state.tool_call_log.extend([r.as_record() for r in turn_output.tool_results])
    return new_state
```

The state update is the only place state changes. The pattern makes the loop reasoning simple: state at turn N + turn output = state at turn N+1.

### 4.3 State and memory

The state is the loop's working memory (per [agent-engineering-playbook.md](./agent-engineering-playbook.md) section 5). It is short-term — it lives for the loop's duration; when the loop terminates, the state is part of the result.

Long-term memory (episodic, semantic) is separate; the application's memory layer manages it. The state may include retrieved long-term memory items as input but does not own them.

### 4.4 State immutability

Treating state as immutable (copy-on-update) prevents subtle bugs where two parts of the runner see different state at the same time. The performance cost is negligible (states are small; copies are cheap).

---

## 5. Termination conditions

The loop terminates when one of several conditions is met. Each produces a defined outcome.

### 5.1 The conditions

| Condition | When | Outcome |
|---|---|---|
| Final answer | LLM response is a final answer (no tool calls) | `success` with the answer |
| Escalate-to-human | LLM called the `escalate_to_human` tool | `escalated` with the escalation reason |
| Turn budget exhausted | turns_remaining == 0 | `terminated_turn_budget` |
| Cost budget exhausted | cost_remaining_usd <= 0 | `terminated_cost_budget` |
| Time budget exhausted | time_remaining_ms <= 0 | `terminated_time_budget` |
| Tool-call budget exhausted | tool_calls_remaining <= 0 | `terminated_tool_budget` |
| Unrecoverable error | provider permanent error, tool registry refusal not handleable | `terminated_error` |

### 5.2 The termination check

The runner checks termination at the top of the loop, before any turn work:

```python
def _should_terminate(self, state: AgentState) -> bool:
    return any([
        state.pending_terminations,
        state.turns_remaining <= 0,
        state.cost_remaining_usd <= 0,
        state.time_remaining_ms <= 0,
        state.tool_calls_remaining <= 0,
    ])
```

The check is fast; it runs before each turn. Budget breaches happen before any work is done in the breaching turn.

### 5.3 The final turn

When a budget breach is detected, the runner may give the LLM one "final turn" with the breach signal in the context — asking the LLM to produce a graceful failure response rather than just terminating silently. The final turn has its own bounded budget; it must not breach further.

This pattern produces a user-facing response on budget termination rather than a silent failure. The cost is one extra LLM call.

### 5.4 The outcome production

```python
def _produce_outcome(self, state: AgentState) -> AgentResult:
    if state.pending_terminations:
        return self._produce_termination_outcome(state)
    if state.history and state.history[-1].is_final_answer():
        return AgentResult(
            outcome="success",
            content=state.history[-1].content,
            turn_count=state.turn_number,
            cost_total=state.cost_total,
            ...,
        )
    # The loop terminated without a clear reason — shouldn't happen, but defensive.
    return AgentResult(outcome="terminated_error", reason="loop_terminated_without_clear_cause", ...)
```

The outcome production is the runner's deliberate finishing step. The outcome is structured, not "whatever the last turn returned."

---

## 6. State checkpointing

For long-running agents, the state should be checkpointable. The pattern:

### 6.1 When to checkpoint

- After every turn (lightweight; state is small).
- Before any LLM call (so a crash during the call can resume with the pre-call state).
- Before any side-effect-with-HITL tool call (so the proposal can be re-issued safely if the runner crashes).

### 6.2 The checkpoint store

The checkpoint store is a key-value store keyed by trace_id. The value is the serialized state. The store has TTL (Meridian: 24 hours; checkpoints older than this are unlikely to be useful).

For the durable-workflow pattern (Temporal, Step Functions), the workflow engine is the checkpoint store. For simpler patterns, Redis or a relational table works.

### 6.3 Resume from checkpoint

The runner can be invoked with a `resume_from` checkpoint ID. The state is loaded; the loop continues. The pattern:

- Transient failures (network, provider 5xx) → automatic resume on retry.
- Manual recovery (debugging, post-incident replay) → operator-invoked resume.
- Step Functions / Temporal workflow restart → orchestration-driven resume.

### 6.4 Checkpoint and side effects

Side-effect-with-HITL tool calls (per [tool-call-authorization.md](../../ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md)) require the checkpoint to be aware of the proposal lifecycle. If a checkpoint resumes after a proposal was created but before the human action, the resume should not re-create the proposal — it should pick up where the proposal currently is in its lifecycle.

The pattern: checkpoint state includes pending proposal IDs; resume queries the proposal registry for current status before continuing.

---

## 7. The model-response parsing layer

The LLM's response is unstructured (or semi-structured). The parser translates it into the runner's structured representation.

### 7.1 The response shapes

LLM responses come in a few shapes:
- **Final answer.** Plain text or structured (JSON object).
- **Tool call(s).** One or more tool-call requests with arguments.
- **Mixed.** Some text plus tool calls (e.g., "I'll look that up: [tool_call]").
- **Refusal.** The model declined to respond.
- **Malformed.** The response cannot be parsed.

### 7.2 The parser

```python
def _parse_decision(self, response: LLMResponse) -> Decision:
    if response.is_refusal():
        return Decision(kind="refusal", details=response.refusal_classification)
    if response.has_tool_calls():
        return Decision(kind="tool_call", tool_calls=response.tool_calls)
    if response.is_final_answer():
        return Decision(kind="final_answer", content=response.content)
    return Decision(kind="malformed", raw=response.raw)
```

The parser is one place. The runner consumes the structured Decision; downstream code does not parse responses again.

### 7.3 Repair loops

For malformed responses (the LLM's structured output is invalid), the runner can attempt a bounded repair:

```python
if decision.kind == "malformed" and state.repair_attempts < MAX_REPAIR_ATTEMPTS:
    state.repair_attempts += 1
    # Re-prompt with the schema error
    response = self.llm_client.call(
        ...,
        messages=state.history + [self._build_repair_message(decision.raw)],
    )
    decision = self._parse_decision(response)
```

Repair has a budget (typically 2 attempts). Exhausted repair → loop termination with `terminated_error`.

### 7.4 Refusal handling

Refusal is not an error; it is a deliberate decision by the model. The runner's response to refusal:
- Add the refusal to history (so subsequent turns see it).
- Decide whether to terminate (escalate to human) or continue (try a different approach).

The decision logic is topology-specific. For ReAct: the next turn's prompt includes the refusal; the model decides what to do. For supervisor / worker: the supervisor decides whether the refusal is terminal or whether to dispatch to a different worker.

---

## 8. Integration with observability and circuit breakers

The runner is wired into the observability and circuit-breaker layers.

### 8.1 Observability

Per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md):
- Top-level span: `agent:<topology>` with outcome attributes.
- Per-turn span: `turn:N` with decision and budget attributes.
- Sub-spans: `llm_call` (from the wrapper), `tool_call:<name>` (from the registry).

The runner is the source of the agent / turn spans; the wrapper and registry produce their own spans within.

### 8.2 Circuit breakers

The cost circuit breaker (per [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)) is inside the LLM wrapper; the runner does not implement cost checks itself. The runner's per-loop cost budget is a *softer* limit than the gateway's circuit breaker — the runner terminates politely; the gateway hard-stops calls that exceed the broader budget.

The two interact: the runner's loop cost budget should be ≤ the per-interaction cost circuit. If the loop runs to the loop budget, it terminates with `terminated_cost_budget`; if the gateway's circuit fires first (because aggregate costs across multiple components exceed the per-interaction limit), the wrapper raises CircuitBreakerTripped and the runner catches it as an unrecoverable error.

### 8.3 The runner's failure path

When the runner encounters an unrecoverable condition (gateway circuit, persistent provider failure, tool registry refusal that the topology cannot handle), it terminates the loop with a structured outcome. The outcome is the runner's response to the failure; it is not an exception thrown to the application.

The application then decides what to do with the terminated outcome (return a graceful failure to the user, escalate to human, retry from checkpoint).

---

## 9. Worked Meridian Care Coordinator example

### 9.1 The supervisor / worker runner

The Care Coordinator's runner is a `SupervisorWorkerRunner` extending the base `AgentRunner`. The topology is supervisor / worker. The runner is invoked per chat interaction with:

- `initial_input`: the user's question + conversation context.
- `prompts`: pinned versions of the supervisor / classifier / clinical_knowledge / drafting / query_rewriter prompts.
- `model_routing`: Opus for supervisor and clinical_knowledge; Sonnet for drafting; Haiku for classifier and query rewriter.
- `budgets`: turn 8, cost $0.50, time 30s, tool calls 10.

### 9.2 A turn-by-turn walkthrough

For the chat interaction described in [meridian-care-coordinator.md](../../ai-architecture-reference-architecture/reference-systems/meridian-care-coordinator.md) section 5.1:

**Turn 1.**
- Supervisor LLM call. Decision: `tool_call` (dispatch to classifier + clinical_knowledge).
- Classifier worker LLM call (sub-span). Returns: question_class=clinical-protocol, retrieval_surfaces=[clinical-guidelines, tenant-protocols], conversational_context=first_turn.
- Clinical_knowledge worker LLM call (sub-span). Internally: retrieval (via tool registry) + reasoning. Returns: answer with citations.
- State updated: history includes supervisor decision, worker results. Budgets decremented. Turn 2.

**Turn 2.**
- Supervisor LLM call. Decision: `tool_call` (dispatch to drafting if a follow-up action is offered).
- Drafting worker LLM call (sub-span). Returns: offered follow-up action.
- State updated. Turn 3.

**Turn 3.**
- Supervisor LLM call. Decision: `final_answer`.
- Loop terminates with `success`.

The runner returns the AgentResult with the final answer; the application formats it for the chat panel; the SSE stream completes.

### 9.3 The runner's instrumentation

Every turn emits a span. The trace shape matches the architecture described in [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) section 9.

### 9.4 Resume from checkpoint

For the async coordination path (per-patient sub-tasks), each per-patient Lambda invokes a fresh runner. If a Lambda crashes mid-task, the orchestrator (Step Functions) retries; the retry resumes from the last checkpoint.

For the chat path, checkpoint resume is rare (chats are short-lived). The checkpoint pattern is in place but rarely used for chat.

### 9.5 The runner's testability

The runner is unit-tested with mock LLM clients (returning canned responses) and mock tool registries. The tests verify:

- Budget enforcement (a mock that returns `tool_call` forever should terminate at turn budget).
- Termination conditions (each kind produces the right outcome).
- State transitions (state update is correct after each turn type).
- Checkpoint serialization (round-trip preserves state).
- Topology-specific behavior (supervisor dispatch, worker result aggregation).

The integration tests use the real wrapper against a sandbox provider account; these verify end-to-end behavior on a small corpus.

### 9.6 The platform discipline

- `meridian.agent.runner` is the only loop driver. Application code does not implement loops.
- New topologies extend the base runner with topology-specific turn-execution logic.
- The state schema is versioned (Pydantic with `Config.use_enum_values`); state-schema changes are migrations.
- Quarterly review of runner code; refactor when accumulated logic suggests topology drift.

---

## 10. Anti-patterns

### 10.1 "Loop in application code"

Application code implements `while not done: response = llm.call(...); ...`. The instrumentation, budgeting, checkpointing, and termination logic are scattered across the application.

**Corrective.** Use the runner. Application code instantiates and invokes; loops live in the runner.

### 10.2 "State is implicit"

The loop's "state" is the LLM's conversation history plus some variables in the application function. There is no structured state object.

**Corrective.** Structured AgentState. Serializable. Single state-update method.

### 10.3 "Termination is whatever the LLM said"

The loop terminates when the LLM returns a response without tool calls. There is no budget check, no explicit outcome production, no error handling.

**Corrective.** Multi-conditional termination per section 5. Explicit outcome per condition.

### 10.4 "No checkpoint"

Loop state lives only in memory. A crash mid-loop loses all progress. Long-running loops cannot be safely resumed.

**Corrective.** Checkpoint per section 6. State persisted at defined points.

### 10.5 "Response parsing scattered"

LLM response parsing happens in multiple places — the runner parses tool calls, the application parses the final answer, eval code parses with its own logic. Inconsistencies accumulate.

**Corrective.** One parser. Returns structured Decision. Consumers consume structured.

### 10.6 "Repair loops without budget"

Malformed-response repair attempts retry until the LLM produces valid output, which could be many tries. Cost balloons; latency unbounded.

**Corrective.** Bounded repair (typically 2 attempts); exhaustion terminates with error.

### 10.7 "Final-turn graceful failure not implemented"

On budget breach, the loop terminates silently. The user-facing layer returns "service unavailable." No degraded response.

**Corrective.** Final turn after breach (section 5.3); LLM produces a graceful failure response within a tight bounded budget.

### 10.8 "Topology-specific logic in shared paths"

Topology-specific turn execution leaks into the shared runner code. Adding a new topology requires changes everywhere.

**Corrective.** Topology-specific code in topology subclasses. The base runner provides shared skeleton; subclasses override turn execution.

---

## 11. Findings (sprint-assignable)

### AGT-LOOP-001 — Severity: Critical
**Finding.** Agent loop is implemented in application code; instrumentation and budgets are partial.
**Recommendation.** Build the runner per section 2; migrate application code to invoke the runner.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-LOOP-002 — Severity: Critical
**Finding.** Loop state is implicit; there is no structured state object.
**Recommendation.** AgentState schema per section 4; single state-update method.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-LOOP-003 — Severity: Critical
**Finding.** Termination is whatever the LLM produced; no budget enforcement at loop level.
**Recommendation.** Multi-conditional termination per section 5; outcomes structured per kind.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-LOOP-004 — Severity: High
**Finding.** No checkpointing; transient failures lose all loop progress.
**Recommendation.** Checkpoint per section 6; checkpoint store with TTL.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-LOOP-005 — Severity: High
**Finding.** Response parsing scattered across multiple code paths; inconsistency observed.
**Recommendation.** One parser returns structured Decision; consumers consume structured.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-LOOP-006 — Severity: High
**Finding.** Repair loops are unbounded; cost and latency unbounded.
**Recommendation.** Bounded repair (2 attempts); exhaustion terminates with error.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-LOOP-007 — Severity: High
**Finding.** Final-turn graceful-failure is not produced on budget breach; loop terminates silently.
**Recommendation.** Final turn per section 5.3; produces graceful user-facing response.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-LOOP-008 — Severity: Medium
**Finding.** Topology-specific logic leaks into shared runner code; new topologies are hard to add.
**Recommendation.** Topology subclasses per section 2.2; shared skeleton in base.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-LOOP-009 — Severity: Medium
**Finding.** Runner is not unit-testable; tests require real LLM calls.
**Recommendation.** Dependency injection per section 2.2; mock LLM client and tool registry for tests.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-LOOP-010 — Severity: Medium
**Finding.** Loop state is mutable; subtle bugs from shared state across runner paths.
**Recommendation.** Immutable state (copy-on-update); single state-update method.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-LOOP-011 — Severity: Medium
**Finding.** Checkpoint does not handle side-effect-with-HITL proposals; resume can double-create proposals.
**Recommendation.** Checkpoint state includes pending proposal IDs; resume queries proposal registry before continuing.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-LOOP-012 — Severity: Medium
**Finding.** Refusal handling is ad-hoc; refusals are sometimes retried, sometimes terminate.
**Recommendation.** Topology-specific refusal logic per section 7.4; consistent within a topology.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-LOOP-013 — Severity: Medium
**Finding.** State schema is unversioned; schema changes break checkpoint compatibility.
**Recommendation.** Versioned state schema; migration for schema changes; check on checkpoint load.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-LOOP-014 — Severity: Low
**Finding.** Tool dispatch in runner does not handle concurrent tool calls; parallel tool calls run sequentially.
**Recommendation.** Concurrent tool dispatch when the model returns multiple tool calls in one response.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-LOOP-015 — Severity: Low
**Finding.** Runner observability is rich but does not surface checkpoint events.
**Recommendation.** Checkpoint events as span events per [agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md).
**Owner.** ai-platform-eng, sprint N+5.

### AGT-LOOP-016 — Severity: Low
**Finding.** Runner test coverage is partial; new topologies often ship without comprehensive tests.
**Recommendation.** Test template per topology; standard test suites cover budget exhaustion, termination conditions, state transitions, checkpoint round-trip.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-LOOP-017 — Severity: Low
**Finding.** Multi-tenancy is implicit in runner — tenant context is propagated through CallContext but not enforced.
**Recommendation.** Tenant context as required state field; runner refuses to start without it.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-LOOP-018 — Severity: Low
**Finding.** Runner documentation is thin; new engineers reading the code do not understand the design choices.
**Recommendation.** Architecture documentation for the runner; commit alongside code.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team with agent loops in application code:

- [ ] **Sprint 0 — design.** Define the runner interface (section 2.2). Decide on the topology subclass pattern.
- [ ] **Sprint 1 — base runner.** Build the base runner with the loop skeleton, state structure, termination conditions, observability integration.
- [ ] **Sprint 1 — first topology.** Implement the topology subclass for the team's primary agent (likely ReAct or supervisor/worker).
- [ ] **Sprint 2 — migrate primary path.** Replace the application's agent loop with the runner. Verify trace coverage matches the agent-step-instrumentation pattern.
- [ ] **Sprint 2 — budgets.** Wire the four budgets. Verify termination conditions fire as expected.
- [ ] **Sprint 3 — response parser.** Single parser; consumers consume structured Decision.
- [ ] **Sprint 3 — repair loop.** Bounded repair for malformed responses.
- [ ] **Sprint 4 — checkpointing.** State checkpoint store; resume from checkpoint.
- [ ] **Sprint 4 — additional topologies.** Add topology subclasses as needed (plan-then-execute, reflection).
- [ ] **Sprint 5 — testability.** Mock LLM client and tool registry; comprehensive runner test suite.
- [ ] **Ongoing — discipline.** Lint rule against application-code loops. Quarterly review of runner code.

A team that completes this sequence has a runner that is predictable, debuggable, recoverable, and observable. A team that defers the runner pattern accumulates loop logic across application code with the cost of inconsistency.

---

## 13. References

- ReAct paper (Yao et al., 2022) — foundational single-agent loop reference.
- Reflexion paper (Shinn et al., 2023) — reflection / self-critique pattern.
- LangGraph, CrewAI, AutoGen — frameworks that implement runner-like patterns; engineering practice here is framework-agnostic.
- Temporal, AWS Step Functions, durable function frameworks — the durable-workflow substrates that are the natural homes for checkpoint persistence.
- This repo: [agent-engineering/agent-engineering-playbook.md](./agent-engineering-playbook.md) — the broader practice that this document deepens (sections 3 and 5).
- This repo: [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — the observability pattern the runner emits to.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — the wrapper the runner calls.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — the gateway-side circuit-breaker that interacts with the runner's budgets.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/agent-topologies.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/agent-topologies.md) — the topology decision that determines which subclass to use.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md) — the tool registry the runner dispatches to.
