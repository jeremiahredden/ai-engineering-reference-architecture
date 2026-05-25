# Agent Step Instrumentation

> **Audience.** Engineers building or refactoring observability for an agent-shaped AI feature. Tech leads who have been asked "why did the agent loop for 18 turns" and could not answer from the existing traces. **Scope.** The *engineering* pattern for instrumenting an agent loop — per-turn spans, per-tool-call sub-spans, budget visibility, decision visibility. Builds on [trace-and-span-design.md](./trace-and-span-design.md) (the framework) and [llm-call-instrumentation.md](./llm-call-instrumentation.md) (the per-LLM-call wrapper). Not the loop design itself (sibling [agent-engineering/agent-loop-design.md](../agent-engineering/) coming). Not the alerting that consumes these traces ([alerting-and-paging-design.md](./) coming). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Agent loops are the most opaque shape in production AI systems. The single LLM call has one span; the agent loop is N LLM calls + M tool calls + variable retrieval steps, where N and M depend on what the model decided at each turn. Without instrumentation tailored to the agent loop, debugging an agent failure is reading scattered log entries and guessing what happened between them.

The discipline this document codifies: every agent loop turn produces a span; every tool call produces a sub-span; budget state at each decision point is captured as a span attribute; the agent's decision at each turn is captured explicitly. The cumulative effect is that an investigator reading the trace can follow what the agent did, why it decided what it did, what budget remained, and where the loop terminated.

The discipline matters because agent failures are usually emergent: the model was confused for two turns, made a wrong tool call on turn three, the tool returned junk, the model retried, the budget breached on turn six. No single span captures the failure; the *sequence* of spans is the diagnosis. Without per-turn spans and budget attributes, the sequence is invisible.

This document is opinionated about three things:

1. **The agent loop is wrapped, like the LLM call is wrapped.** A single agent-loop runner is the only path; it owns the per-turn instrumentation; consumers do not implement their own.
2. **Budgets are first-class span attributes.** Turn budget remaining, cost budget remaining, time budget remaining, tool-call budget remaining — all visible on every turn span. This is the only way to debug "why did the loop terminate at turn N" without correlation.
3. **The agent's decision is captured explicitly.** Not inferred from the next span; recorded as `ai.agent.decision: tool_call` or `final_answer` or `escalate` or `terminate_budget`. The decision is the load-bearing question; surface it directly.

Structure: (2) the agent-loop runner interface; (3) per-turn span structure; (4) tool-call sub-spans; (5) budget attributes; (6) decision and termination instrumentation; (7) supervisor / worker hierarchies; (8) trace-as-debugging-surface playbook; (9) worked Meridian Care Coordinator example; (10) anti-patterns; (11) findings; (12) adoption checklist.

---

## 2. The agent-loop runner interface

Similar to the LLM-call wrapper: a single runner is the only path; consumers invoke it; the runner owns the loop, the instrumentation, the budget enforcement, the termination decision.

### 2.1 The signature

```python
class AgentLoopRunner:
    def run(
        self,
        topology: str,                  # "single" | "supervisor_worker" | "plan_execute" | etc.
        initial_input: AgentInput,
        tools: list[Tool],
        budgets: AgentBudgets,          # turn / cost / time / tool_call limits
        context: CallContext,           # tenant, user, feature, session, trace
        prompts: AgentPromptBundle,     # versioned prompts per role
        model_routing: ModelRouting,    # which model for which role
    ) -> AgentResult:
        ...
```

The runner is responsible for:
- Opening the top-level agent span.
- Loop control (turn iteration, budget checking, termination).
- Per-turn span creation with all required attributes.
- Dispatching LLM calls (via the LLM-call wrapper).
- Dispatching tool calls (via the tool registry).
- Recording the agent's decision each turn.
- Handling the loop termination (final answer, budget breach, escalation, error).
- Closing the top-level span with the outcome.

What the runner does NOT do:
- Build the prompts — that is the prompt-engineering layer's job.
- Implement tool authorization — that is the tool registry's job.
- Implement individual LLM-call instrumentation — that is the LLM-call wrapper's job.
- Make routing decisions across models — that is the routing layer's job.

### 2.2 The "only path" discipline

Just like the LLM-call wrapper, the agent-loop runner is the only path that agent code uses to run a loop. Application code that builds its own loop (calling the LLM wrapper directly, managing its own state) is rejected at code review. Lint rules can detect common patterns (an iteration over LLM calls without going through the runner).

The discipline pays off when a new observability requirement lands: there is one place to add it.

### 2.3 The agent budgets

```python
class AgentBudgets:
    turn_budget: int = 10
    cost_budget_usd: float = 0.50
    time_budget_ms: int = 30_000
    tool_call_budget: int = 12
```

Each budget is enforced per-loop. When any budget is breached, the loop terminates; the termination is recorded; the agent's prompt gets the "we are out of budget" signal so the next-and-final turn can produce a graceful response.

Budgets are passed in (not hard-coded) so different topologies and features can have different limits. The Meridian Care Coordinator's chat loop uses `(8, 0.50, 30_000, 10)`; the async per-patient sub-task uses `(6, 0.75, 60_000, 8)`; the analytics copilot's contained agent uses `(6, 1.00, 60_000, 12)`.

---

## 3. Per-turn span structure

Each loop turn is a span. The span captures everything that happened that turn.

### 3.1 The span hierarchy

```
span: agent:reAct  (or supervisor_worker, plan_execute, etc.)
├── span: turn:1
│   ├── span: llm_call                  (the model's reasoning + tool selection)
│   └── span: tool_call:retrieve         (tool the model selected)
│       ├── span: retrieval:bm25
│       └── span: retrieval:vector
├── span: turn:2
│   ├── span: llm_call
│   └── span: tool_call:check_drug_interactions
├── span: turn:3
│   └── span: llm_call                  (final answer, no tool call)
```

Three things to notice:

- The turn span is the wrapper around what happens in that turn.
- The LLM call within the turn is a separate sub-span (instrumented by the LLM-call wrapper — see [llm-call-instrumentation.md](./llm-call-instrumentation.md)).
- Tool calls within the turn are sub-spans of the turn (not of the LLM call). The LLM call decided to make the tool call, but the tool call's execution is separate from the LLM call.

### 3.2 Turn span attributes

Every turn span has:

| Attribute | Source / Meaning |
|---|---|
| `ai.agent.topology` | The topology being run |
| `ai.agent.turn.number` | The turn number (1, 2, 3, ...) |
| `ai.agent.turn.role` | The role of the call this turn (supervisor, worker:clinical_knowledge, etc.) — only meaningful for multi-role topologies |
| `ai.agent.budget.turns_remaining` | Turns left after this one |
| `ai.agent.budget.cost_remaining_usd` | Cost budget left |
| `ai.agent.budget.time_remaining_ms` | Time budget left |
| `ai.agent.budget.tool_calls_remaining` | Tool-call budget left |
| `ai.agent.decision` | What the agent decided this turn — see §6 |
| `ai.agent.decision.reason` | If the decision was `terminate_*`, the reason |
| `ai.agent.parent_trace_id` | If this loop is nested in another loop (e.g., async per-patient inside a coordination task), the outer trace |

The budget attributes are critical. Without them, a trace shows that the loop terminated at turn 6 but does not show *why* — was it because the model returned a final answer, or because the cost budget breached, or because turn budget breached? With them, the answer is on the turn span itself.

### 3.3 Pre- and post-turn timing

The turn span's start time is when the runner begins the turn's preparation (prompt assembly, context update); the end time is when the turn completes (the LLM call finished, the tool call finished if any). The latency attribute captures both.

For multi-call turns (e.g., a supervisor that dispatches in parallel), the turn span encloses all the parallel work; sub-spans show the parallelism.

---

## 4. Tool-call sub-spans

Each tool call within a turn produces a sub-span. The LLM-call wrapper produces its own spans for the LLM call itself; the tool-call sub-span captures the tool execution.

### 4.1 The sub-span attributes

| Attribute | Source / Meaning |
|---|---|
| `ai.tool.name` | The tool name |
| `ai.tool.version` | The tool registration version |
| `ai.tool.class` | `read-only` / `draft-only` / `side-effect-with-HITL` / `side-effect-immediate` |
| `ai.tool.arguments_hash` | Hash of arguments |
| `ai.tool.authorization.decision` | `allowed` / `denied` |
| `ai.tool.authorization.reason` | If denied, the reason code |
| `ai.tool.success` | Boolean |
| `ai.tool.error_type` | When failed |
| `ai.tool.latency_ms` | Tool execution latency |
| `ai.tool.proposal_id` | For side-effect-with-HITL tools |
| `ai.tool.draft_id` | For draft-only tools |

The authorization attributes matter. When a tool call fails authorization, the trace shows the decision and the reason on the sub-span; the investigator does not have to correlate against a separate authorization log.

### 4.2 The audit-log linkage

For side-effect tools (draft-only and side-effect-with-HITL), the tool-call sub-span includes the linkage attributes (`ai.tool.proposal_id`, `ai.tool.draft_id`). These link the trace to the audit log entries described in [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md).

The lineage from agent-action → proposal → human-review → realized-side-effect is queryable across the trace + audit log via these linkage IDs.

### 4.3 Concurrent tool calls

When the model returns multiple tool calls in one response (parallel-tool-call patterns), the sub-spans run concurrently under the same turn span. The trace shows the parallelism explicitly; the turn's total latency is the max of the parallel tool calls' latencies.

---

## 5. Budget attributes

The four budgets are visible on every turn span. This is the load-bearing instrumentation.

### 5.1 Why every turn carries all four budgets

When the loop terminates at turn 8, the investigator wants to know:
- Did the model produce a final answer? (Look at `ai.agent.decision` on the last turn.)
- Did the model exhaust the turn budget? (Was `turns_remaining = 0` going into the last turn?)
- Did the cost budget breach? (`cost_remaining_usd` approaching zero.)
- Did the time budget breach? (`time_remaining_ms` approaching zero.)
- Did the tool-call budget breach? (`tool_calls_remaining = 0`.)

Without all four on every turn, the diagnostic requires correlation with budget-tracking logs. With them, the trace is self-contained.

### 5.2 Budget decrement timing

Budgets are decremented at the start of the turn (turn budget) and at the start of each LLM call / tool call (cost budget, tool-call budget). Time budget is computed at each decision point as `time_budget - elapsed`.

The turn span's budget attributes are *the budget state entering this turn*. The next turn's span shows the state after this turn (or no next turn if the loop terminated).

### 5.3 Pre-decision budget check

The runner checks budgets before each action:

- Before calling the LLM (the turn's reasoning call): check turn / cost / time budgets.
- Before calling each tool: check tool-call / cost / time budgets.
- Between actions: check time budget.

A budget breach at any check point terminates the loop. The termination is recorded; the agent gets a final turn with the `terminate_budget` signal so it can produce a graceful response.

---

## 6. Decision and termination instrumentation

Every turn ends with a decision. Capture it explicitly.

### 6.1 The `ai.agent.decision` attribute

Values:

| Value | Meaning |
|---|---|
| `tool_call` | The model returned one or more tool calls; the loop continues |
| `final_answer` | The model returned a final answer with no tool calls; the loop terminates normally |
| `escalate_human` | The model called the `escalate_to_human` tool; the loop terminates with the escalation |
| `terminate_turn_budget` | Turn budget exhausted; loop terminated by runner |
| `terminate_cost_budget` | Cost budget exhausted; loop terminated by runner |
| `terminate_time_budget` | Time budget exhausted; loop terminated by runner |
| `terminate_tool_budget` | Tool-call budget exhausted; loop terminated by runner |
| `terminate_error` | Unrecoverable error; loop terminated |

The decision is on every turn span (for non-final turns, almost always `tool_call`; for the final turn, the actual reason).

### 6.2 The decision.reason attribute

For `terminate_*` decisions, an additional reason attribute:

- `ai.agent.decision.reason = "turn_budget_exhausted"` — and the budget value at termination.
- `ai.agent.decision.reason = "cost_budget_exhausted"` — and the cost-so-far.
- `ai.agent.decision.reason = "time_budget_exhausted"` — and the elapsed time.

For `escalate_human`:
- `ai.agent.decision.reason = "low_confidence"` / `"out_of_scope"` / `"explicit_user_request"` — the agent's reason for escalating.

### 6.3 The loop's final outcome

The top-level agent span (the parent of all turn spans) has its own outcome attributes:

| Attribute | Meaning |
|---|---|
| `ai.agent.outcome` | `success` / `escalated` / `terminated_budget` / `terminated_error` |
| `ai.agent.outcome.turn_count` | Total turns executed |
| `ai.agent.outcome.tool_call_count` | Total tool calls executed |
| `ai.agent.outcome.total_cost_usd` | Aggregate cost |
| `ai.agent.outcome.total_latency_ms` | Aggregate latency |

These attributes support aggregate analysis: "what percentage of Care Coordinator interactions terminate by turn budget vs final answer?" "What is the p95 turn count by question class?"

---

## 7. Supervisor / worker hierarchies

The supervisor / worker topology adds a layer of hierarchy. The instrumentation pattern:

### 7.1 The supervisor span

The top-level agent span for a supervisor / worker topology represents the supervisor's role. Within it, the supervisor's per-turn spans are children. The supervisor's tool calls (which include dispatch-to-worker calls) are sub-spans of the supervisor's turn spans.

### 7.2 Workers as nested loops or single calls

If the worker is a single LLM call, it shows up as a sub-span (the `llm_call` span produced by the LLM-call wrapper).

If the worker is itself a multi-turn loop (rare; supervisor / worker is usually single-call workers), the worker is its own agent-loop span hierarchy nested under the supervisor's tool-call sub-span.

For the Meridian Care Coordinator: workers are single-call, so they appear as `llm_call` sub-spans of the supervisor's turn spans (representing the supervisor's tool calls that dispatched to them). The Care Coordinator's trace shape is in section 9.

### 7.3 Worker attribution

Each worker call carries `ai.agent.role` (e.g., `worker:clinical_knowledge`, `worker:drafting`, `worker:classifier`) as a span attribute. This lets per-worker analysis aggregate cleanly across the platform.

### 7.4 The supervisor's consolidation

After workers have returned, the supervisor's consolidation pass is a separate `llm_call` sub-span (often named `worker:supervisor_consolidation` or similar). This is the final LLM call that produces the user-facing response.

---

## 8. Trace-as-debugging-surface playbook

The instrumentation pattern serves the debugging discipline. When an agent failure is investigated, the playbook is:

### 8.1 Pull the trace

The failed interaction's trace ID is the entry point. The interaction ID is available from the user-facing system (the chat log, the task result, the error report).

### 8.2 Read top-down

Start at the top-level agent span. Read its outcome attributes:
- Did the agent terminate by budget? Skip to the budget analysis.
- Did the agent escalate? Read the escalation reason.
- Did the agent complete with `success` but the user reported it as wrong? Continue to per-turn analysis.

### 8.3 Per-turn analysis

For each turn span:
- What was the model's decision? (`ai.agent.decision`)
- What was the budget state entering this turn?
- What sub-spans does it contain (LLM call, tool calls)?
- For the LLM call sub-span: what was the prompt version, the model version, the cost, the latency, the finish reason?
- For tool-call sub-spans: which tools were called, with what authorization decision, with what success?

The diagnostic question is usually answerable from one or two turn spans:
- "Turn 4 made a wrong tool call" → inspect turn 4's tool-call sub-span; correlate against the tool's expected use.
- "Turn 5 hit the cost budget" → inspect turn 5's budget attributes; correlate against the budgeted limit.
- "Turn 2's worker call had a wrong answer" → inspect the worker LLM call sub-span; check the retrieved doc IDs; identify whether the retrieval was correct.

### 8.4 Common diagnosis paths

| Symptom | Trace evidence to check |
|---|---|
| Agent looped 12 turns when expected 3 | Per-turn decisions; was the model failing to terminate? Was a tool returning garbage? |
| Agent's final answer was wrong | Final LLM call's prompt version (regression?); retrieval sub-spans' doc IDs (correct content retrieved?) |
| Cost overran | Per-turn cost-remaining attribute; identify which turn exceeded the per-turn baseline |
| Latency was slow | Per-turn latency; LLM-call latency attributes (TTFT, total); tool-call latency |
| Tool authorization rejected | Tool-call sub-span's authorization attributes; correlate against caller role / tenant |
| Side effect was committed when it should not have been | Tool-call sub-span's class (was it side-effect-immediate when it should have been side-effect-with-HITL?) |

The pattern: the trace contains the evidence; the diagnosis is *read the trace*, not *rerun the agent*.

---

## 9. Worked Meridian Care Coordinator example

### 9.1 Trace shape

The Care Coordinator uses supervisor / worker. A typical chat interaction's trace:

```
trace: interaction-2026-05-25-a7b8
└── span: agent:supervisor_worker
    │   attributes: ai.agent.topology=supervisor_worker
    │              ai.agent.outcome=success
    │              ai.agent.outcome.turn_count=3
    │              ai.agent.outcome.tool_call_count=4
    │              ai.agent.outcome.total_cost_usd=0.18
    │              ai.agent.outcome.total_latency_ms=4280
    ├── span: turn:1
    │   │   attributes: ai.agent.turn.number=1
    │   │              ai.agent.turn.role=supervisor
    │   │              ai.agent.budget.turns_remaining=8
    │   │              ai.agent.budget.cost_remaining_usd=0.50
    │   │              ai.agent.budget.tool_calls_remaining=10
    │   │              ai.agent.decision=tool_call
    │   ├── span: llm_call  (supervisor reasoning, see llm-call-instrumentation example)
    │   ├── span: tool_call:dispatch_classifier
    │   │   └── span: llm_call  (the classifier worker)
    │   └── span: tool_call:dispatch_clinical_knowledge
    │       ├── span: llm_call  (the clinical-knowledge worker — reasoning over retrieved chunks)
    │       └── span: tool_call:retrieve
    │           ├── span: retrieval:bm25
    │           ├── span: retrieval:vector
    │           └── span: rerank
    ├── span: turn:2
    │   │   attributes: ai.agent.turn.number=2
    │   │              ai.agent.budget.turns_remaining=7
    │   │              ai.agent.budget.cost_remaining_usd=0.31
    │   │              ai.agent.budget.tool_calls_remaining=6
    │   │              ai.agent.decision=tool_call
    │   ├── span: llm_call  (supervisor consolidation reasoning)
    │   └── span: tool_call:dispatch_drafting
    │       └── span: llm_call  (the drafting worker)
    └── span: turn:3
        │   attributes: ai.agent.turn.number=3
        │              ai.agent.budget.turns_remaining=6
        │              ai.agent.budget.cost_remaining_usd=0.24
        │              ai.agent.budget.tool_calls_remaining=4
        │              ai.agent.decision=final_answer
        └── span: llm_call  (supervisor's final consolidation)
```

The trace tells the full story:
- The supervisor took 3 turns total.
- Turn 1: dispatched to classifier and clinical-knowledge worker in parallel.
- Turn 2: dispatched to drafting worker.
- Turn 3: consolidated and produced the final answer.
- Outcome: success, $0.18 cost, 4.3s latency, well within budgets.

### 9.2 A debugging scenario

A clinician reports that an interaction "took forever and the answer was wrong." Pull the trace:

```
span: agent:supervisor_worker
    attributes: ai.agent.outcome=terminate_cost_budget
                ai.agent.outcome.turn_count=7
                ai.agent.outcome.tool_call_count=15
                ai.agent.outcome.total_cost_usd=0.51

(turns 1-5 normal-looking)

span: turn:6
    attributes: ai.agent.turn.number=6
                ai.agent.budget.cost_remaining_usd=0.08
                ai.agent.budget.tool_calls_remaining=2
                ai.agent.decision=tool_call
    span: tool_call:dispatch_clinical_knowledge   (3rd time this turn)
        span: llm_call
            attributes: ai.llm.prompt.version=clinical_knowledge@1.8.0
                       ai.llm.response.finish_reasons=[stop]
                       ai.llm.response.had_tool_calls=False

span: turn:7
    attributes: ai.agent.turn.number=7
                ai.agent.budget.cost_remaining_usd=0.02
                ai.agent.decision=terminate_cost_budget
                ai.agent.decision.reason=cost_budget_exhausted
    span: llm_call  (degraded final response)
```

Diagnosis from the trace:
- Cost budget was breached at turn 7 (after 6 productive turns + degraded final).
- Turns 5 and 6 both dispatched to `clinical_knowledge` worker again — the supervisor was looping on the same sub-task.
- The clinical-knowledge worker's LLM-call spans show no tool calls — the worker was producing answers each time but the supervisor was not satisfied and dispatching again.
- The retrieval spans in those turns show the same doc IDs being retrieved repeatedly.
- The user's question (visible in the top-level span's input attribute) was a complex multi-hop clinical question.

Conclusion: the supervisor's consolidation logic was failing to recognize that the clinical-knowledge worker had returned a sufficient answer; it kept re-dispatching. The fix: prompt-engineer the supervisor's consolidation logic to recognize sufficient-answer signals.

The fix lands; the regression case is added to the eval suite. Future occurrences are caught in CI before production.

### 9.3 What this trace lets the team avoid

Without per-turn spans and budget attributes:
- The team would see "the agent's interaction cost $0.51" and not be able to attribute which turns drove the cost.
- The team would see the final response and not know it was a degraded response (cost-budget terminated, not final-answer).
- The repetitive dispatch pattern would not be visible without correlating LLM calls and tool calls by hand.

With per-turn spans and budget attributes: the diagnosis takes about 10 minutes. Without: it takes hours and may not arrive at the right answer.

### 9.4 Aggregate analysis

The per-turn span attributes also support aggregate queries. The Care Coordinator team's dashboards include:

- **Outcome distribution by feature.** What percentage of chat interactions are `success` vs `escalated` vs `terminated_budget` vs `terminated_error`? Trend over time.
- **Turn-count distribution by feature.** P50, P95, P99 turn count per feature. A turn-count drift is often the leading indicator of a quality or prompt regression.
- **Per-worker performance.** Aggregate latency, cost, and quality (when judged) per worker role. Identifies which worker is the cost / latency bottleneck.
- **Budget headroom.** Distribution of `cost_remaining_usd` at loop termination. A trend toward lower headroom is the leading indicator of cost circuit breaker trips.

---

## 10. Anti-patterns

### 10.1 "Agent loop is application code, not a wrapped runner"

The team built the agent loop inline in application code. Instrumentation is partial; new agent features re-implement loops; per-turn observability is inconsistent.

**Corrective.** Single agent-loop runner; consumers invoke it; the runner owns per-turn instrumentation.

### 10.2 "Per-turn spans are absent"

The agent's trace shows the LLM calls and tool calls but no turn boundary. "Which turn was this LLM call part of" requires manual reconstruction.

**Corrective.** Per-turn span structure (section 3). The turn boundary is the load-bearing analytic unit; capture it explicitly.

### 10.3 "Budgets not on spans"

The runner enforces budgets but does not capture budget state on spans. When the loop terminates, the trace does not show which budget breached.

**Corrective.** Four budget attributes on every turn span (section 5).

### 10.4 "Agent decision is inferred from sub-spans"

The trace shows a turn span with sub-spans; whether the agent decided to make a tool call vs return a final answer is inferred from whether tool-call sub-spans exist. The decision is implicit.

**Corrective.** `ai.agent.decision` attribute on every turn span. Explicit beats implicit.

### 10.5 "Tool calls share a span with the LLM call that selected them"

The trace shows one span per turn; tool execution attributes are mixed with LLM-call attributes. Aggregating "tool latency" across the platform requires complex parsing.

**Corrective.** Tool-call sub-spans separate from LLM-call sub-spans within the turn. Each is independently queryable.

### 10.6 "Authorization decisions not on tool-call spans"

A tool-call sub-span shows the call happened but not whether authorization was checked. Investigating an unauthorized-action incident requires correlating against a separate authorization log.

**Corrective.** Authorization attributes on every tool-call sub-span (section 4.1).

### 10.7 "Supervisor / worker calls are flattened"

Worker calls are emitted as separate top-level spans rather than as sub-spans of the dispatching turn. The supervisor / worker hierarchy is invisible in the trace; debugging requires reconstructing the parent-child relationship.

**Corrective.** Worker spans are children of the supervisor's tool-call sub-span (which dispatched to the worker). The hierarchy mirrors the architecture.

### 10.8 "Outcome attributes only on the top-level span, not on per-turn"

The trace's top-level span shows the outcome; per-turn spans do not show the per-turn decision. Aggregate analysis is forced to read the top-level span and infer per-turn behavior.

**Corrective.** Per-turn decision on per-turn spans; outcome on the top-level. Both layers populated.

---

## 11. Findings (sprint-assignable)

### OBS-AGENT-001 — Severity: Critical
**Finding.** Agent loops are implemented as application code; per-turn instrumentation is absent.
**Recommendation.** Build the agent-loop runner per section 2; migrate existing agent code through it; instrument per-turn.
**Owner.** ai-platform-eng, sprint N+1.

### OBS-AGENT-002 — Severity: Critical
**Finding.** Budgets are enforced but not visible on spans; loop terminations cannot be attributed from the trace.
**Recommendation.** Budget attributes on every turn span per section 5; surfaces in trace-as-debugging-surface workflow.
**Owner.** ai-platform-eng, sprint N+1.

### OBS-AGENT-003 — Severity: High
**Finding.** Agent decision is inferred from sub-span presence; the trace does not capture decisions explicitly.
**Recommendation.** `ai.agent.decision` and `ai.agent.decision.reason` attributes per section 6.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-AGENT-004 — Severity: High
**Finding.** Tool-call sub-spans do not include authorization-decision attributes; unauthorized-call investigations require cross-system correlation.
**Recommendation.** Authorization attributes on every tool-call sub-span per section 4.1.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-AGENT-005 — Severity: High
**Finding.** Supervisor / worker trace hierarchy is flat — worker calls are top-level spans rather than children of dispatching turn.
**Recommendation.** Restructure span hierarchy per section 7; workers are children of the supervisor's dispatch.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-AGENT-006 — Severity: High
**Finding.** Per-turn cost attribution is absent; aggregate cost is captured but per-turn breakdown requires parsing sub-spans.
**Recommendation.** Per-turn cost rollup attribute; sum from per-LLM-call costs within the turn.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-AGENT-007 — Severity: High
**Finding.** Loop outcome (success vs escalated vs terminated-budget) is not surfaced on the top-level agent span; aggregate analysis cannot distinguish outcomes.
**Recommendation.** `ai.agent.outcome` attribute per section 6.3.
**Owner.** ai-platform-eng, sprint N+2.

### OBS-AGENT-008 — Severity: High
**Finding.** Multiple parallel tool calls within a turn are recorded sequentially in the span hierarchy; concurrency is invisible.
**Recommendation.** Concurrent tool-call sub-spans per section 4.3; trace backend visualizes parallelism.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-AGENT-009 — Severity: Medium
**Finding.** Worker role attribution is inconsistent; aggregate per-worker analysis requires manual filtering.
**Recommendation.** Standardized `ai.agent.role` values per section 7.3; lint rule on worker dispatch.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-AGENT-010 — Severity: Medium
**Finding.** Side-effect tool linkage (proposal_id, draft_id) is in audit log but not in the trace; lineage queries require cross-system join.
**Recommendation.** Linkage attributes on tool-call sub-spans per section 4.2.
**Owner.** ai-platform-eng + security-eng, sprint N+3.

### OBS-AGENT-011 — Severity: Medium
**Finding.** Aggregate turn-count distribution is not dashboarded; turn-count drift cannot be early-detected.
**Recommendation.** Per-feature turn-count distribution dashboard; alert on > 20% drift week-over-week.
**Owner.** ai-platform-eng + observability-eng, sprint N+4.

### OBS-AGENT-012 — Severity: Medium
**Finding.** Per-worker quality (when sampled by online judge) is not aggregated to the worker role.
**Recommendation.** Per-worker judge-pass-rate dashboards; surface which worker is the quality bottleneck.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-AGENT-013 — Severity: Medium
**Finding.** Budget-headroom-at-termination trends are not surfaced; cost-circuit-breaker trips are reactive rather than predictive.
**Recommendation.** Distribution of `cost_remaining_usd` at loop termination per feature; alert on declining trend.
**Owner.** ai-platform-eng, sprint N+4.

### OBS-AGENT-014 — Severity: Medium
**Finding.** Loop nesting (per-patient inside async coordination) is not linked across traces.
**Recommendation.** Parent-trace event linkage per [trace-and-span-design.md](./trace-and-span-design.md) section 2.3.
**Owner.** ai-platform-eng, sprint N+3.

### OBS-AGENT-015 — Severity: Medium
**Finding.** Per-turn role (which worker / which dispatch level) is sometimes ambiguous when the same role is invoked multiple times in one turn.
**Recommendation.** Add `ai.agent.role.invocation_index` for disambiguation when needed.
**Owner.** ai-platform-eng, sprint N+5.

### OBS-AGENT-016 — Severity: Low
**Finding.** Turn-level latency is computed but not surfaced as a per-feature SLI.
**Recommendation.** Per-feature P50 / P95 / P99 turn latency dashboards.
**Owner.** ai-platform-eng, sprint N+5.

### OBS-AGENT-017 — Severity: Low
**Finding.** Loop termination reason histograms are not generated; engineering team does not know the rate of budget-induced terminations.
**Recommendation.** Histogram by `ai.agent.outcome` per feature per day; surface trends.
**Owner.** ai-platform-eng, sprint N+5.

### OBS-AGENT-018 — Severity: Low
**Finding.** Tool retry / repair-loop instrumentation does not distinguish retries within a turn from new tool calls.
**Recommendation.** Add `ai.tool.retry_attempt` attribute when applicable.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team running agent-shaped AI features without per-turn instrumentation:

- [ ] **Sprint 0 — runner.** Build the agent-loop runner per section 2. Define the standard topology entry points.
- [ ] **Sprint 1 — migrate primary.** Migrate the main agent feature to use the runner. Verify per-turn spans appear.
- [ ] **Sprint 1 — budgets.** Wire the four budgets into the runner; emit budget attributes on every turn span.
- [ ] **Sprint 2 — decisions.** Emit `ai.agent.decision` and `ai.agent.outcome` on every turn / top-level span.
- [ ] **Sprint 2 — tool-call sub-spans.** Tool calls are sub-spans of the turn (not flattened); authorization attributes captured.
- [ ] **Sprint 3 — supervisor / worker hierarchy.** For supervisor / worker topologies, restructure to match section 7.
- [ ] **Sprint 3 — concurrent tool calls.** Parallel tool calls are sub-spans showing the parallelism.
- [ ] **Sprint 4 — aggregate dashboards.** Outcome distribution, turn-count distribution, per-worker performance, budget-headroom trends.
- [ ] **Sprint 4 — async / HITL linkage.** Cross-trace parent-trace linkage for async coordination and HITL flows.
- [ ] **Ongoing — debugging discipline.** When agent failures are investigated, the trace is the first artifact pulled; the team does not re-run before reading the existing trace.

A team that completes this sequence has the trace-as-debugging-surface for agent failures that turns "rerun and see what happens" into "read the trace and diagnose." This is the discipline that makes agent operations sustainable on a normal on-call rotation.

---

## 13. References

- OpenTelemetry semantic conventions, including `gen_ai.*` and the (developing) `ai.agent.*` extension.
- LangSmith run-tree, Braintrust span model, Phoenix trace conventions — vendor patterns for agent observability.
- This repo: [observability-and-telemetry/trace-and-span-design.md](./trace-and-span-design.md) — the framework that this builds on.
- This repo: [observability-and-telemetry/llm-call-instrumentation.md](./llm-call-instrumentation.md) — the per-LLM-call wrapper that the agent loop calls internally.
- This repo: [observability-and-telemetry/retrieval-instrumentation.md](./) (coming) — the retrieval-specific instrumentation that often appears within tool-call sub-spans.
- This repo: [agent-engineering/agent-engineering-playbook.md](../agent-engineering/agent-engineering-playbook.md) — section 7 covers the broader agent observability discipline; this document is the depth.
- This repo: [agent-engineering/agent-loop-design.md](../agent-engineering/) (coming) — the design choices for the agent loop that this instrumentation makes visible.
- This repo: [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — the circuit-breaker pattern that consumes the budget attributes this document emits.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/agent-topologies.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/agent-topologies.md) — the topology decision that determines the per-turn shape.
- Sibling repo: [ai-architecture-reference-architecture/guardrails-and-policy-architecture/tool-call-authorization.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/guardrails-and-policy-architecture/tool-call-authorization.md) — the audit log that the tool-call sub-spans link to.
