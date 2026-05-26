# Agent vs Workflow Decision

> **Audience.** Tech leads and architects making the call on whether a new AI feature should be an agent (LLM decides the next step), a workflow (deterministic plan, LLM steps embedded), or a hybrid (workflow skeleton with a contained agent loop in one step). Engineers refactoring a feature that started as one shape and grew into the other. **Scope.** The decision framework — criteria, the decision tree, the three shapes' operational properties, and how to engineer each one. Not the runner implementation depth (see [agent-loop-design.md](./agent-loop-design.md)). Not the architectural topology catalogue (sibling [ai-architecture-reference-architecture](https://github.com/jeremiahredden/ai-architecture-reference-architecture)'s `reference-patterns/agent-topologies.md`). **Worked client.** Meridian Health.

---

## 1. Why this document exists

[agent-engineering-playbook.md](./agent-engineering-playbook.md) opens with a single claim that most "agent" projects should not be agents — they should be workflows with one or two LLM steps embedded, because workflows have bounded cost, bounded latency, deterministic shape, and a debuggable execution trace. This document is the depth on that claim.

The shape choice — agent, workflow, hybrid — is the single most consequential agent engineering decision. It determines the cost envelope (workflow: bounded by step count × per-step cost; agent: unbounded by loop). It determines the latency envelope (workflow: sum of step latencies; agent: unbounded by turn count). It determines the failure model (workflow: each step fails or succeeds; agent: dozens of trajectory-level failure modes). It determines the debugging surface (workflow: step-by-step trace; agent: trajectory reading). It determines the eval methodology (workflow: per-step + outcome; agent: trajectory + outcome). It determines the on-call burden (workflow: predictable; agent: cost incidents, loop incidents, runaway trajectories).

Teams default to "agent" because the framework demos are agent demos and the model marketing emphasises agentic capability. The framework demo is a happy-path demo on a curated input; production traffic is the long tail. The default is wrong for most features. The right shape is the simplest shape that can produce the outcome on the production traffic the feature will see.

This document is opinionated about three things:

1. **The shape is chosen explicitly with criteria.** Not "let's build an agent because the framework is fun." A decision document records the criteria considered, the shape chosen, and the conditions under which the shape would be re-examined.
2. **The default is workflow.** Workflow is the simpler shape; deviation from it requires justification. An agent earns its loop by demonstrating that the task genuinely requires LLM-controlled next-step decisions on each turn.
3. **Hybrid is often the right answer.** A workflow skeleton with a small bounded agent loop inside one or two steps captures the predictability of the workflow and the flexibility of the agent on the steps where it matters. Most "agent" features are better expressed as hybrids.

Structure: (2) the three shapes and their operational properties; (3) the decision tree; (4) workflow shape — when, how to engineer; (5) agent shape — when, how to engineer; (6) hybrid shape — workflow skeleton with contained loop; (7) re-evaluation cadence and shape migration; (8) boundaries with adjacent patterns; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The three shapes and their operational properties

A clean characterisation before the decision tree.

### 2.1 Workflow (deterministic plan, LLM steps embedded)

A workflow is a directed graph of steps. The graph is fixed at development time. Each step is either deterministic code (call an API, transform data, write to a store) or an LLM call (summarise, classify, extract). The next step is chosen by the workflow definition, not by the LLM.

```
input → classify (LLM) → if class=A → enrich → summarise (LLM) → write → return
                       → if class=B → fetch → answer (LLM) → return
                       → if class=C → escalate → return
```

LLM calls are *embedded steps*. They have inputs (the prompt + context) and outputs (the response, parsed). The workflow handles the routing.

**Operational properties.**

- **Cost is bounded.** Sum of per-step LLM costs over the longest path. Predictable.
- **Latency is bounded.** Sum of per-step latencies over the longest path. Predictable.
- **Failure model is per-step.** Each step succeeds, fails, or times out. Workflow handles step failure with deterministic logic.
- **Debugging is per-step trace.** Each step has inputs, outputs, duration, cost. The trace is a tree of completed steps.
- **Eval is per-step + outcome.** Eval the LLM steps individually (judge / golden set) and the workflow's outcome (correctness on input → output).
- **Shape changes are code changes.** Adding a branch, adding a step, changing the routing — all are pull requests with diffs.

### 2.2 Agent (LLM decides the next step)

An agent is a loop. Each turn, the LLM sees the current state (input + history + tool results so far) and emits a decision: call a tool with arguments, produce a final answer, or escalate. The runner executes the decision and loops with updated state.

```
loop:
  decision = LLM(state)
  if decision == final_answer: return
  if decision == escalate: hand off
  if decision == tool_call(t, args): result = dispatch(t, args); state.update(result)
  if budget_breach: terminate
```

The graph is *not* fixed; it is constructed by the model turn by turn. Tools are first-class entities the model chooses among.

**Operational properties.**

- **Cost is unbounded by the loop.** Bounded only by budget (which the runner enforces). A single loop can burn 50 turns × $0.20/turn = $10, or 500 turns if the budget allows.
- **Latency is unbounded by the loop.** Same. Bounded only by time budget.
- **Failure model is trajectory-level.** Wrong tool call, wrong arguments, infinite loop, tool failure, partial completion, lost context, hallucinated intermediate result. Dozens of modes.
- **Debugging is trajectory reading.** Each loop is unique; the same input produces a different trajectory the next run. Reading the trace is the debugging activity.
- **Eval is trajectory + outcome.** Eval the trajectory (was each tool call right?) and the outcome (did the agent achieve the goal?). Substantially harder than workflow eval.
- **Shape changes are prompt + tool changes.** Adding a tool, changing a tool description, changing the system prompt — all change the model's decisions.

### 2.3 Hybrid (workflow skeleton, agent within)

A workflow with one (or rarely two) of its steps implemented as a bounded agent loop. The agent loop runs inside the step; its result is the step's output.

```
input → classify (LLM) → if class=complex → AGENT_STEP → write → return
                       → if class=simple → answer (LLM) → write → return
                       → if class=escalate → escalate → return
```

The `AGENT_STEP` is a bounded loop with strict budgets (turn / cost / time / tool-call). It is contained: the workflow knows when it started, knows when it finished, has a cost budget for it, and treats its outcome as a step result. The agent doesn't choose the next workflow step; the workflow does.

**Operational properties.**

- **Cost is bounded.** Workflow sums to a bounded total; the agent step has a step-level cost budget.
- **Latency is bounded.** Same.
- **Failure model is step-level.** The agent step either produced a valid result, breached its budget, escalated, or failed. The workflow handles the four outcomes.
- **Debugging is per-step trace plus agent trajectory.** The workflow trace shows steps; the agent step's trajectory is the inner detail of one step.
- **Eval is per-step + outcome + trajectory of the agent step.** Workflow-eval the outer; agent-eval the inner.
- **Shape changes can be local.** Adding a new agent capability inside the agent step is a prompt/tool change. Changing the workflow's branching is a code change. The two surfaces are separable.

### 2.4 Summary comparison

| Property | Workflow | Agent | Hybrid |
| --- | --- | --- | --- |
| Cost envelope | Bounded by graph | Bounded only by budget | Bounded by graph + step budget |
| Latency envelope | Bounded by graph | Bounded only by budget | Bounded by graph + step budget |
| Failure modes | Per-step (few) | Trajectory-level (many) | Per-step + trajectory inside one step |
| Debugging surface | Step trace | Trajectory | Step trace + trajectory inside one step |
| Eval methodology | Per-step + outcome | Trajectory + outcome | Per-step + outcome + agent-step trajectory |
| Shape change | Code | Prompt + tools | Code (outer) + prompt + tools (inner) |
| Operational burden | Low | High | Medium |
| Suitability for novel inputs | Limited (within graph) | High (model decides) | High within agent step, bounded by outer |
| Determinism on same input | High (modulo LLM stochasticity per step) | Low | Medium |

The summary's bottom row is the most operationally important one: *workflow is predictable, agent is not, hybrid is predictable except inside the agent step*. Choose the shape whose predictability profile matches what the operations team can sustain.

---

## 3. The decision tree

The criteria, in priority order. Apply top-down.

### 3.1 Criterion 1 — Is the task decomposition known at development time?

If a tech lead can write down the steps the system will take to produce the outcome, the task decomposition is known. Examples: "classify the email, then route to the right department" — known. "Fetch the patient record, summarise the recent visit history, format the summary as a SOAP note" — known. "Given a question about a policy, retrieve relevant sections, answer with citations" — known.

If the task decomposition is known: **workflow.**

If the task decomposition is *partly* known — the outer shape is known but one or two steps require flexibility — **hybrid.**

If the task decomposition is genuinely unknown — the next step depends on what previous steps revealed, and there is no useful structure to capture in code — **agent.**

"Genuinely unknown" is rare. Most teams claim it on the first sketch and then discover, when forced to enumerate cases, that 90% of production traffic follows a predictable shape with a few branches. That 90% is workflow territory.

### 3.2 Criterion 2 — Is the cost envelope acceptable in the unbounded case?

Imagine the worst-case agent loop on the worst-case input: 50 turns, each $0.15. That's $7.50 per request. If the feature is invoked 100 times per day, that's $750/day, $22,500/month. If it's invoked 1000 times per day, that's $7,500/day, $225,000/month. If a misbehaving prompt causes 500-turn loops, multiply by 10. A single agent feature can produce a six-figure monthly bill in the wrong configuration.

If the worst-case cost is acceptable to the business and the engineering team has the on-call capacity to handle cost incidents — agent is on the table. Otherwise — workflow or hybrid.

The cost analysis must be honest. Teams often underweight the worst case ("we'll put a budget on it") and then discover the budget was wrong, the budget wasn't enforced, or a single tenant blew through the global budget in an afternoon. See [agent-cost-control.md](./agent-cost-control.md) for the engineering controls; the *decision* to use an agent is partly a decision to invest in those controls.

### 3.3 Criterion 3 — Is the latency envelope acceptable?

A 50-turn loop with 1.5s per turn is 75s. If the feature is user-facing in a chat interface, that's unacceptable. If the feature is a backend batch process — fine.

Workflows have a knowable latency envelope (sum of step latencies on the longest path). Agents do not.

If the feature requires sub-5s latency at p99: workflow strongly preferred. If sub-15s: workflow preferred; hybrid possible with strict budget on the agent step. If async / batch / non-interactive: agent's unbounded latency may be acceptable.

### 3.4 Criterion 4 — Is the failure model debuggable by the operations team?

A workflow's failure model is per-step: a step succeeded, failed, or timed out. The trace shows which step, the inputs, the outputs. An engineer can read the trace and identify the failure in minutes.

An agent's failure model is trajectory-level. The trace shows many turns; the model's chain of reasoning is implicit in the conversation log; the failure (wrong tool call, wrong arguments, missing context) requires reading the full trajectory and understanding why the model chose what it chose. Debugging is slower and requires specialised skill.

If the team has agent-debugging skill and an established trajectory-reading practice (per [agent-observability.md](./agent-observability.md) and [debugging-from-traces.md](../observability-and-telemetry/debugging-from-traces.md)) — agent is operationally feasible. If not — workflow.

### 3.5 Criterion 5 — Is the eval methodology feasible?

Workflow eval is per-step plus outcome. Each LLM step has a golden set (input → expected output). The workflow's outcome is evaluated end-to-end on a golden set of input → expected output. Standard eval engineering.

Agent eval is trajectory plus outcome. Trajectory eval requires expected-trajectory golden sets (which are expensive to build and brittle), or it requires sampling production trajectories for human or LLM-judge review. Outcome eval requires a way to score "did the agent achieve the goal" which is often subjective. See [agent-evals.md](./agent-evals.md).

If the team has the eval-engineering investment to support trajectory eval — agent is feasible. If the team's eval practice is just-getting-started — workflow keeps eval simple while the practice matures.

### 3.6 Criterion 6 — Will the production traffic surprise the team?

Workflows handle known cases well; they handle unknown cases by falling into a default branch or failing. If 95% of production traffic matches the workflow's branches and the remaining 5% is acceptable to degrade or escalate — workflow.

Agents handle novel cases by reasoning. If the production traffic is genuinely diverse and the long tail is large — and degrading the tail is unacceptable — agent's flexibility is worth its operational burden.

Honest analysis: most teams' production traffic is less diverse than the team expects. The tail is real but is often handleable with a workflow that escalates the un-handled cases to a human. The expensive agent approach is justified when escalation isn't acceptable — for example, when the volume is too high for human handling.

### 3.7 The decision tree

```
1. Task decomposition known at dev time?
   - Yes → workflow (go to engineering it)
   - Partly → hybrid (workflow skeleton; agent inside the flexible step)
   - No → continue

2. Cost envelope acceptable for unbounded loop?
   - No → workflow or hybrid with tight inner budget; do not deploy an unbounded agent
   - Yes → continue

3. Latency envelope acceptable for unbounded loop?
   - No (user-facing < 5s) → workflow or tightly-budgeted hybrid
   - Yes or async → continue

4. Team has agent-debugging skill + observability practice?
   - No → workflow; build the skill before attempting an agent
   - Yes → continue

5. Team has eval-engineering investment for trajectories?
   - No → workflow; build the eval practice before attempting an agent
   - Yes → continue

6. Production traffic genuinely diverse, escalation not acceptable?
   - No → workflow with escalation; pay no operational tax
   - Yes → agent
```

The tree pushes toward workflow at every gate. An agent is what you get when no earlier gate stops you. This is intentional: the operational tax of an agent is high, and the answer to most "should we build an agent?" questions on a first reading is "not yet."

### 3.8 A note on the model improvement axis

Frontier models in 2026-Q2 are markedly better at agent loops than they were even a year ago — fewer tool-call hallucinations, better turn-by-turn reasoning, better refusal handling. The threshold for "an agent is workable" has moved. But the operational reality has not moved: agents are still expensive, slow, hard to debug, and hard to eval. Better models reduce the *correctness* burden; they do not reduce the *operational* burden. The decision tree applies regardless of model.

---

## 4. Workflow shape — when, and how to engineer it

### 4.1 When workflow is right

- The task decomposition is known.
- The number of branches is manageable (typically < 20).
- LLM steps are clearly bounded — summarise this, classify this, extract this.
- Cost and latency need to be predictable.
- The team wants per-step debugging and per-step eval.
- The feature ships fast and iterates safely (each step is a small change).

Most production AI features land here.

### 4.2 How to engineer the workflow

Use a workflow engine — Temporal, Step Functions, Airflow, Prefect, a framework-specific orchestrator, or a homegrown step runner. The engine provides the durability, retry, and observability properties that make workflows operationally cheap.

Each step has:

- **A name** (used in traces and logs).
- **A typed input** (Pydantic / dataclass / TypedDict).
- **A typed output** (same).
- **A timeout** (per-step, not just whole-workflow).
- **A retry policy** (transient retry; permanent fail-fast).
- **An idempotency key** (so retries don't double-execute side effects).

LLM steps additionally have:

- **A prompt reference** (versioned per [prompt-versioning.md](../prompt-engineering/prompt-versioning.md)).
- **A model reference** (pinned per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md)).
- **A structured-output schema** (per [structured-output-engineering.md](../prompt-engineering/structured-output-engineering.md)).
- **A per-step cost budget** (workflow rejects the step if the LLM call exceeds the budget; sums into the per-workflow budget).

### 4.3 Branching

Branches are explicit. A branch is a step whose output determines the next step. The branch's logic lives in the workflow definition, not in the model.

```python
@step
def classify(input: Email) -> Classification:
    # LLM step; returns a class enum
    return llm.classify(input, schema=Classification)

@workflow
def email_handler(input: Email):
    classification = classify(input)
    if classification.category == "support":
        enriched = enrich_support(input)
        response = summarise_support(input, enriched)
        send(response)
    elif classification.category == "billing":
        record = fetch_billing(input.customer_id)
        response = answer_billing(input, record)
        send(response)
    else:
        escalate(input)
```

The model classifies; the workflow routes. The branching is debuggable: the trace shows which branch was taken and why.

### 4.4 Workflow-level guardrails

- **Per-workflow time budget.** Fail-fast if the workflow runs longer than the configured budget.
- **Per-workflow cost budget.** Fail-fast on cumulative LLM cost breach.
- **Per-step retry caps.** Bounded retries; no infinite retry loops.
- **Idempotency on side-effect steps.** Retries do not double-execute (per [error-and-partial-failure.md](./error-and-partial-failure.md)).
- **Escalation step.** A defined step for "we cannot handle this" — typically a queue for human review.

### 4.5 Workflow eval

- **Per-step golden sets.** Each LLM step has its own golden set; promoted to production only when the eval gate passes (per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md)).
- **End-to-end golden set.** Input → expected workflow outcome. Tests the full path.
- **Step-coverage check.** Verify every branch is exercised in the eval set; uncovered branches are silently broken.

### 4.6 Workflow observability

- **One trace per workflow execution.**
- **One span per step.**
- **LLM steps emit the rich LLM span** per [llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md).
- **Branch decisions are logged as span attributes** (not buried in step code).
- **Cost is summed into a workflow-level cost metric.**

The trace is the debugging surface. An engineer paged on a workflow failure opens the trace, sees the failed step, opens the step's inputs and outputs, and identifies the failure in minutes.

---

## 5. Agent shape — when, and how to engineer it

### 5.1 When agent is right

- The task decomposition is genuinely not known at development time.
- The action space is large enough that a workflow would have hundreds of branches.
- The model's flexibility (deciding next step based on prior tool results) is essential.
- The team has the operational investment to support the burden.
- Cost and latency are acceptable in the bounded-by-budget case.
- The eval-engineering practice supports trajectory eval.

Examples: a code-modification agent that reads files, runs tests, edits, re-runs tests, and decides the next action based on test output. A research agent that searches, reads, follows leads, and synthesises. A complex customer-support agent that handles multi-turn conversations with tool-mediated lookups and state transitions.

Counter-examples that look like agents but are workflows: "classify this email and route it" (workflow). "Summarise this article" (single LLM call). "Answer this RAG question" (workflow: retrieve → answer).

### 5.2 How to engineer the agent

The engineering is described in depth in [agent-engineering-playbook.md](./agent-engineering-playbook.md) and [agent-loop-design.md](./agent-loop-design.md). Summary requirements for the shape decision:

- **A runner with explicit budgets** — turn, cost, time, tool-call (per agent-loop-design section 5).
- **A tool registry with curated, well-described tools** (per [tool-architecture.md](./tool-architecture.md)).
- **A memory strategy** (per [memory-engineering.md](./memory-engineering.md)).
- **Error and partial-failure handling** (per [error-and-partial-failure.md](./error-and-partial-failure.md)).
- **Trajectory observability** (per [agent-observability.md](./agent-observability.md)).
- **Trajectory + outcome eval** (per [agent-evals.md](./agent-evals.md)).
- **Cost circuit-breaker** (per [agent-cost-control.md](./agent-cost-control.md) and [cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md)).
- **An escalation path** for the cases the agent shouldn't handle alone.

The investment is substantial. A team shipping its first agent should expect the operational scaffolding to take longer than the agent itself.

### 5.3 The agent's escalation path

An agent that runs unsupervised at scale needs an escalation tool — a tool the model can call to say "I cannot handle this; hand to human." The escalation tool is itself a finding on the eval set: did the agent escalate the cases it should have escalated, and not escalate the ones it could handle?

A common failure mode is the agent that never escalates — it loops and burns cost rather than giving up. The cure is in the prompt (instruct the agent to escalate when uncertain), the tool design (make escalation a first-class action with a low friction surface), and the budget breach behaviour (escalation is the natural action when budgets are tight).

### 5.4 The "agent loop wrapped in a workflow step" alternative

Before committing to a top-level agent, consider whether the loop could be wrapped inside a workflow step (the hybrid shape, section 6). This often gives most of the flexibility benefit while keeping the predictability of the outer workflow.

---

## 6. Hybrid shape — workflow skeleton, contained agent loop

### 6.1 When hybrid is right

- The outer shape is known but one (rarely two) inner steps require flexibility.
- The team wants the outer workflow's predictability and observability.
- The team can place a tight budget on the agent step.
- The agent step's outcome can be expressed as a step result the workflow consumes.

This is the right shape for most "agent" features. The outer workflow handles intake, classification, post-processing, side effects, and escalation. The inner agent handles the part of the work where the next-step decision genuinely belongs to the model.

### 6.2 Engineering the hybrid

```python
@step
def classify(input: Request) -> Classification:
    ...

@step
def simple_answer(input: Request) -> Answer:
    ...

@agent_step(
    budgets=AgentBudgets(turns=10, cost_usd=2.50, time_s=60, tool_calls=15),
    tools=[search_tool, fetch_tool, summarise_tool],
    escalation_tool=escalate_tool,
)
def complex_research(input: Request, context: Context) -> Answer | Escalation:
    """Agent loop runs inside this step. Returns Answer or Escalation."""
    ...

@workflow
def research_handler(input: Request):
    cls = classify(input)
    if cls.category == "simple":
        answer = simple_answer(input)
    elif cls.category == "complex":
        result = complex_research(input, context)
        if isinstance(result, Escalation):
            return escalate(input, reason=result.reason)
        answer = result
    else:
        return escalate(input, reason="unclassifiable")
    write(answer)
```

The agent step has a budget. The workflow handles the four step outcomes: answer, escalation, budget breach (treated as escalation), and step error. The outer trace shows the step ran; the inner trace shows the trajectory inside the step.

### 6.3 Budget envelopes for the agent step

The agent step's budgets are tighter than a top-level agent's:

- Turn budget: 5–15 (vs 30–50 for top-level).
- Cost budget: $1–5 (vs $10–50 for top-level).
- Time budget: 30–60s (vs 300s+ for top-level).
- Tool-call budget: 10–25.

Tight budgets keep the agent step bounded. When the agent step breaches a budget, the workflow handles it — typically by escalating or falling back to a simpler answer.

### 6.4 Eval for hybrids

- **Workflow eval on the outer.**
- **Agent eval on the inner step** (trajectory + outcome inside the step).
- **End-to-end eval on the full flow** (input → outcome, treating the agent step as a black box).

Three eval surfaces, not one. Worth the investment because the hybrid is the production shape most teams will live with.

### 6.5 Why hybrid is often the right call

The hybrid captures the predictability the operations team needs and the flexibility the model brings to the genuinely-flexible step. It avoids the workflow shape's brittleness on long-tail inputs and the agent shape's unboundedness on the bulk of the input distribution. The outer workflow handles 80% of the operational concerns; the inner agent handles the 20% that genuinely needs model judgement.

Most teams that ship a successful agent feature end up here, regardless of where they started. Starting here saves the months of rework that come from starting at top-level agent and discovering the operational burden was wrong-sized.

---

## 7. Re-evaluation cadence and shape migration

### 7.1 When to re-evaluate

The shape choice ages. Re-examine it when:

- **Production traffic shape changes.** A workflow that handled 95% of traffic now handles 70% — the long tail grew. Either a new branch or migration to hybrid.
- **A new capability is requested.** The new capability doesn't fit the existing branches. Workflow → hybrid (add an agent step) or extend the workflow.
- **A cost incident reveals the agent was wrong-sized.** Agent → hybrid (constrain inside a workflow) or agent → workflow (the agent's bounded behaviour was a workflow in disguise).
- **The eval methodology proves infeasible.** The agent's trajectory eval can't be built at acceptable quality — re-shape to a hybrid where the inner agent is small enough to eval, or to a workflow with branches.
- **A model upgrade changes feasibility.** A workflow that struggled with edge cases under an older model might now be handled by a hybrid step on the upgraded model.

Quarterly review of every AI feature's shape is the operational discipline. The shape choice is a *decision*, not a *fact*; it should be revisited as conditions change.

### 7.2 Workflow → hybrid migration

Pattern: identify the step or branch that handles the long tail badly. Replace it with an agent step with tight budgets. The outer workflow is unchanged.

Risk: the agent step's eval and observability investment is new for the team. Plan for the investment alongside the shape change.

### 7.3 Hybrid → workflow migration

Pattern: the agent step's trajectory analysis reveals the agent took the same N paths 95% of the time. Convert those N paths into workflow branches. The agent step shrinks (handles only the residual 5%) or is removed entirely.

This is the maturity arc most hybrids follow: the agent step grows in coverage early, then narrows as the team learns which inputs need flexibility and which can be handled deterministically.

### 7.4 Agent → hybrid migration

Pattern: the agent's behaviour reveals a predictable outer shape (intake, dispatch, post-process, escalate). Lift the outer shape into a workflow; constrain the agent to the flexible inner step.

This is one of the most operationally beneficial migrations a team can make. The cost envelope tightens; the trace becomes legible; the eval surface decomposes; on-call burden drops.

### 7.5 Migration is incremental

Shape migration does not require a big rewrite. The migration plan typically:

1. Add the new shape as a parallel path behind a feature flag.
2. Shadow-traffic both paths; compare outcomes.
3. Migrate a tenant at a time once shadow shows parity.
4. Decommission the old path.

The pattern is the same one used for model upgrades and prompt versions. Apply it here.

---

## 8. Boundaries with adjacent patterns

### 8.1 Workflow vs single LLM call

Workflow implies multiple steps. A single LLM call (input → prompt → response → return) is not a workflow; it's a *call*. The workflow framing applies when there are 2+ steps to orchestrate. Don't reach for the workflow engine for a single call.

### 8.2 Workflow vs pipeline

Pipelines (data pipelines, ML training pipelines) have a stable shape too, but their unit is "stage" (e.g., extract → transform → load) and their cadence is batch. Workflows here are *per-request*. Use the right framing for the right cadence.

### 8.3 Agent vs multi-agent

The decision tree in this document is for single-agent shapes. Multi-agent (multiple agents communicating) is its own shape with its own decision criteria; see [multi-agent-coordination.md](./multi-agent-coordination.md). The most common mistake is treating multi-agent as the natural next step from single-agent; it usually isn't.

### 8.4 Agent vs RAG

RAG (retrieve-augmented generation) is a workflow pattern: retrieve → answer. It is not an agent. Some RAG flows have an agent inner step (the model decides what to query next or what to follow up on); that's a hybrid. Don't conflate the two.

### 8.5 Shape choice vs framework choice

The shape choice (workflow / agent / hybrid) is independent of the framework choice (Temporal, LangGraph, CrewAI, Vercel AI SDK, raw SDK). The shape is *what* the system does; the framework is *how* you build it. Choose the shape first, then choose the framework that supports the shape best.

---

## 9. Worked Meridian example

Meridian Health's care-coordinator feature is the canonical example. Three subsystems, three different shape choices.

### 9.1 The care-coordinator chat experience

A clinician talks with the care coordinator about a patient: asks about recent labs, recent appointments, recommended follow-ups, gaps in preventative care. The clinician's questions are varied; the responses require retrieval, synthesis, and occasional escalation to a human care manager.

**Shape choice.** Hybrid.

- Outer workflow: intake → classify question type → dispatch to the appropriate inner handler → post-process → log → return.
- Inner handlers: most question types are workflow steps with a retrieval + answer pattern. The "complex multi-step research" handler is an agent step with budgets (turns=8, cost=$1.50, time=45s, tools=10).

**Why hybrid.** The outer shape is known — classification dispatches to a small set of handlers, each with a predictable pattern. But the "complex multi-step" handler genuinely needs LLM-controlled next-step decisions (which patient record section to look at next, whether to follow up on a thread, when to escalate to a human). Workflow would have hundreds of branches; pure agent would be operationally too heavy.

**Cost envelope.** Outer workflow: $0.05–0.20 per request typical. Agent step (when invoked): up to $1.50 budget. Total per-request budget: $1.70. Observed average: $0.18. Worst case at budget breach: $1.70 + escalation overhead.

### 9.2 The patient-summary generation

For each upcoming clinical visit, Meridian generates a one-page patient summary the clinician reads before the visit.

**Shape choice.** Workflow.

- Steps: fetch patient record → fetch recent labs → fetch recent encounters → summarise medical history (LLM) → summarise recent encounters (LLM) → format SOAP-note structure (LLM) → render PDF.
- LLM steps: 3, each with a tight per-step prompt and a 30-second timeout.

**Why workflow.** Task decomposition fully known. Cost predictable: 3 LLM calls × ~$0.04 = $0.12 per summary. Latency predictable: 8–12s. Per-step eval straightforward. No need for model-decided next steps.

**Cost envelope.** $0.12 per summary, deterministic.

### 9.3 The patient-API copilot

Developers integrate with Meridian's patient API. The copilot answers their API questions, generates request examples, and helps debug response payloads.

**Shape choice.** Workflow (specifically, RAG with structured response).

- Steps: classify intent (LLM) → retrieve from API docs and code samples (deterministic) → answer (LLM) → check answer for hallucinated endpoints (deterministic) → return.
- Optional: if classify=debug, also include the developer's pasted request/response in the retrieval context.

**Why workflow.** API documentation Q&A is a textbook RAG workflow. The flexibility an agent would offer (browsing the API surface, executing test calls) is not what developers want — they want fast, citation-backed answers.

**Cost envelope.** $0.03–0.08 per question.

### 9.4 The analytics-warehouse copilot

Analysts at Meridian's payer customers ask natural-language questions against the warehouse: "What was 30-day readmission rate by service line in Q1 by payer mix?"

**Shape choice.** Hybrid with agent step.

- Outer workflow: classify question → look up authorised data scopes → run SQL generation pipeline (LLM) → validate SQL against schema → execute SQL → format results (LLM) → return.
- Inner agent step: SQL generation. The model has access to schema-introspection tools and example-query tools; iterates on SQL until validation passes or budget exhausts.

**Why hybrid.** SQL generation is the part where the model's flexibility helps — it tries a query, validates, fixes column names, validates again. The rest of the flow is deterministic.

**Cost envelope.** Outer: $0.02. Agent step: budget $0.50, observed average $0.12. Total: $0.14 average, $0.52 worst-case.

### 9.5 What the four shapes have in common

- Every shape choice is explicit and recorded in the feature's design document.
- Every cost envelope is honest about the worst case, not just the average.
- Every agent step (in the two hybrids) has tight budgets.
- Every feature's shape is reviewed each quarter; the analytics copilot moved from pure agent (initial implementation) to hybrid in Q4-25 after a cost incident.

The right shape is the one whose operational properties match the feature's requirements and the team's investment.

---

## 10. Anti-patterns

### 10.1 "Default to agent because the framework is fun"

The team picks an agent framework, builds an agent, ships it. The first production incident reveals the cost envelope was wrong-sized. The team retrofits budgets and observability under pressure.

**Corrective.** The shape decision is explicit and uses the decision tree. The default is workflow.

### 10.2 "Workflow with hundreds of branches"

The team built a workflow but the branch explosion has overwhelmed it. Every new case adds a branch. The workflow definition is a thousand lines of if-else.

**Corrective.** When branch count exceeds ~20–30, examine whether the problem has shifted shape — a hybrid (workflow skeleton with agent step in the flexible region) may be the right migration.

### 10.3 "Agent because we couldn't enumerate the cases"

The team didn't do the case enumeration; concluded "the cases are too varied for a workflow"; built an agent. Production reveals 80% of traffic does fit a workflow.

**Corrective.** Enumerate the cases honestly using production traffic samples. Workflow + escalation for the 20% may be the right shape.

### 10.4 "Agent with no budgets"

The team built an agent and didn't engineer the budgets. The first runaway-cost incident is a real one with a real bill.

**Corrective.** Budgets are part of the shape choice. An agent without budgets is not a viable agent; it's an unbounded liability.

### 10.5 "Hybrid that's secretly agent-with-extra-steps"

The team wrote a "hybrid" but the outer workflow has one step (the agent step), no classification, no post-processing, no escalation. It's an agent with a workflow wrapper that does nothing.

**Corrective.** A hybrid earns the name by having meaningful outer structure. If the outer workflow is one step, the shape is agent — accept that and engineer accordingly.

### 10.6 "Workflow rewrite of an agent because of one incident"

A single cost incident triggers a rewrite from agent to workflow. The workflow can't handle the long-tail cases the agent did. The team rewrote what was the right shape into the wrong shape under pressure.

**Corrective.** Cost incidents prompt fixes to the agent's budgets and controls. Shape migration is a strategic decision made calmly, not a panicked reaction to a single incident.

### 10.7 "Shape decision unrecorded"

There is no document recording which shape was chosen and why. Future engineers wonder; quarterly review is impossible because there's no baseline to review against.

**Corrective.** Every AI feature has a one-pager recording shape, criteria, cost envelope, latency envelope, eval methodology, and re-evaluation triggers.

### 10.8 "Shape decision is final"

The team chose a shape on day one and never revisits. Three years later the conditions have changed but the shape is treated as immutable.

**Corrective.** Quarterly shape review. Migration is incremental.

---

## 11. Findings (sprint-assignable)

### AGT-SHAPE-001 — Severity: Critical
**Finding.** A production AI feature is implemented as a top-level agent with no recorded shape decision.
**Recommendation.** Author the shape-decision one-pager per section 4.3 and 9. Re-examine using the decision tree; if hybrid is right, migrate.
**Owner.** ai-platform-eng + tech-lead-of-feature, sprint N+1.

### AGT-SHAPE-002 — Severity: Critical
**Finding.** A top-level agent has no enforced cost or turn budgets.
**Recommendation.** Wire budgets per [agent-loop-design.md](./agent-loop-design.md) section 5 before any further roadmap work on the feature.
**Owner.** ai-platform-eng, sprint N+1.

### AGT-SHAPE-003 — Severity: High
**Finding.** A workflow has accumulated > 30 branches; readability and maintainability degrading.
**Recommendation.** Audit the long-tail branches; migrate the flexible region to an agent step per section 6.
**Owner.** feature-team, sprint N+2.

### AGT-SHAPE-004 — Severity: High
**Finding.** Agent feature lacks trajectory observability; debugging incidents takes hours.
**Recommendation.** Wire the trajectory observability per [agent-observability.md](./agent-observability.md) before any further capability work.
**Owner.** ai-platform-eng, sprint N+2.

### AGT-SHAPE-005 — Severity: High
**Finding.** Agent feature has no eval; production changes are made blind.
**Recommendation.** Build trajectory + outcome eval per [agent-evals.md](./agent-evals.md); gate prompt and tool changes on the eval.
**Owner.** ai-platform-eng + feature-team, sprint N+2.

### AGT-SHAPE-006 — Severity: High
**Finding.** Hybrid "agent step" has no per-step cost budget; only the global workflow budget gates it.
**Recommendation.** Per-step budgets per section 6.3; workflow handles step breach explicitly.
**Owner.** feature-team, sprint N+2.

### AGT-SHAPE-007 — Severity: Medium
**Finding.** Shape choice was made on day one and has not been reviewed; production traffic has shifted shape.
**Recommendation.** Quarterly shape review per section 7; record outcome in the feature's design document.
**Owner.** tech-lead-of-feature, sprint N+3.

### AGT-SHAPE-008 — Severity: Medium
**Finding.** Workflow branches are not covered by the eval set; uncovered branches silently regress.
**Recommendation.** Step-coverage check per section 4.5; eval set covers every branch.
**Owner.** ai-platform-eng + feature-team, sprint N+3.

### AGT-SHAPE-009 — Severity: Medium
**Finding.** Workflow has LLM steps without pinned model versions; model upgrades change behaviour silently.
**Recommendation.** Pin per [model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md); promote pin via eval gate.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-SHAPE-010 — Severity: Medium
**Finding.** Agent has no escalation tool; loops until budget breach on cases it should hand off.
**Recommendation.** Add escalation tool per section 5.3; eval against cases that should escalate.
**Owner.** feature-team, sprint N+3.

### AGT-SHAPE-011 — Severity: Medium
**Finding.** Workflow steps lack per-step timeouts; one slow step holds the workflow hostage.
**Recommendation.** Per-step timeouts per section 4.2; workflow handles timeout as a step failure.
**Owner.** ai-platform-eng, sprint N+3.

### AGT-SHAPE-012 — Severity: Medium
**Finding.** "Hybrid" feature is structurally an agent with a wrapper workflow that does nothing.
**Recommendation.** Either restructure the outer workflow to do meaningful work (classification, dispatch, post-processing) or accept the shape as agent and engineer accordingly.
**Owner.** feature-team, sprint N+4.

### AGT-SHAPE-013 — Severity: Medium
**Finding.** Workflow's idempotency keys are missing on side-effect steps; retries can double-execute.
**Recommendation.** Idempotency keys per section 4.2; verify retries are safe.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-SHAPE-014 — Severity: Low
**Finding.** Workflow's branching decisions are not surfaced as span attributes; trace requires reading step code.
**Recommendation.** Branch decision as span attribute per section 4.6.
**Owner.** ai-platform-eng, sprint N+4.

### AGT-SHAPE-015 — Severity: Low
**Finding.** Workflow cost is not summed into a workflow-level metric; cost analysis requires per-step aggregation.
**Recommendation.** Workflow-level cost metric per [cost-dashboards.md](../observability-and-telemetry/cost-dashboards.md).
**Owner.** ai-platform-eng, sprint N+4.

### AGT-SHAPE-016 — Severity: Low
**Finding.** Shape-decision documents exist for new features but are not linked from the feature's runbook.
**Recommendation.** Runbook links to shape-decision document; on-call can reference it during incidents.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-SHAPE-017 — Severity: Low
**Finding.** Migrations from one shape to another are done as big-bang rewrites; risk is concentrated.
**Recommendation.** Incremental migration per section 7.5 — parallel paths behind feature flag, shadow-traffic, tenant-at-a-time cutover.
**Owner.** ai-platform-eng, sprint N+5.

### AGT-SHAPE-018 — Severity: Low
**Finding.** Engineers refer to a workflow as an "agent" colloquially; the imprecise language confuses operational discussions.
**Recommendation.** Lexicon discipline in design documents and code comments; use the three shape names precisely.
**Owner.** ai-platform-eng, sprint N+5.

---

## 12. Adoption sequencing checklist

For a team building a new AI feature:

- [ ] **Sprint 0 — case enumeration.** Sample 100 production-realistic inputs. List the cases. Identify the long tail.
- [ ] **Sprint 0 — decision tree application.** Walk the tree (section 3); record the answer to each criterion.
- [ ] **Sprint 0 — shape-decision one-pager.** Shape chosen, criteria, cost envelope, latency envelope, eval methodology, re-evaluation triggers.
- [ ] **Sprint 1 — workflow or hybrid implementation.** Build the chosen shape. For hybrid, scope the inner agent step tightly.
- [ ] **Sprint 1 — budgets.** Wire per-step (workflow), per-agent-step (hybrid), or per-loop (agent) budgets.
- [ ] **Sprint 1 — observability.** Per-step traces (workflow); trajectory traces (agent or agent step of hybrid).
- [ ] **Sprint 2 — eval.** Per-step eval (workflow LLM steps); trajectory + outcome eval (agent).
- [ ] **Sprint 2 — escalation.** Workflow escalation step or agent escalation tool.
- [ ] **Sprint 2 — eval gate.** Promotion gated on eval pass (per [eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md)).
- [ ] **Sprint 3 — production gradual rollout.** Feature-flag gated; canary tenants first.
- [ ] **Sprint 3 — runbook.** On-call procedures; reference to shape-decision document.
- [ ] **Ongoing — quarterly review.** Re-examine the shape; record the conclusion.

For a team with an existing top-level agent considering migration:

- [ ] **Sprint 0 — production traffic analysis.** Sample 200 trajectories; identify the recurring patterns; identify the rare flexible cases.
- [ ] **Sprint 0 — hybrid shape design.** Outer workflow skeleton from the recurring patterns; inner agent step from the flexible cases.
- [ ] **Sprint 1 — parallel implementation.** Hybrid as a parallel path behind a feature flag.
- [ ] **Sprint 1 — shadow traffic.** Both paths receive production traffic; outcomes compared.
- [ ] **Sprint 2 — tenant-by-tenant cutover.** Migrate one tenant; observe a week; migrate next.
- [ ] **Sprint 3 — decommission.** Old path removed; budgets and observability updated.

A team that completes the checklist has shipped a feature whose shape matches its requirements and whose operational properties are known and bounded. A team that defers the shape decision builds the wrong thing and learns about it in production.

---

## 13. References

- [agent-engineering-playbook.md](./agent-engineering-playbook.md) — the broader practice; this document deepens section 2 (the shape decision).
- [agent-loop-design.md](./agent-loop-design.md) — the runner depth for when the shape is agent or hybrid.
- [tool-architecture.md](./tool-architecture.md) — tool surface for agents and agent steps.
- [memory-engineering.md](./memory-engineering.md) — memory strategy when memory is needed.
- [error-and-partial-failure.md](./error-and-partial-failure.md) — the failure-handling discipline that's heavier for agents.
- [agent-cost-control.md](./agent-cost-control.md) — the cost-control investments that change the cost-envelope criterion.
- [agent-observability.md](./agent-observability.md) — the trajectory observability that changes the debugging criterion.
- [agent-evals.md](./agent-evals.md) — the eval engineering that changes the eval feasibility criterion.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — agent-step spans for hybrids.
- [observability-and-telemetry/llm-call-instrumentation.md](../observability-and-telemetry/llm-call-instrumentation.md) — per-LLM-step instrumentation for workflows.
- [eval-engineering/eval-gate-architecture.md](../eval-engineering/eval-gate-architecture.md) — eval gate for promotion regardless of shape.
- [prompt-engineering/prompt-versioning.md](../prompt-engineering/prompt-versioning.md) — prompts as versioned artefacts for workflow LLM steps.
- [cicd-and-eval-gates/model-version-pinning.md](../cicd-and-eval-gates/model-version-pinning.md) — model pinning for predictability.
- [cost-and-finops/cost-budget-circuit-breaker.md](../cost-and-finops/cost-budget-circuit-breaker.md) — the gateway-side breaker that complements per-feature budgets.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/agent-topologies.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/agent-topologies.md) — architectural topology catalogue.
- Sibling repo: [ai-architecture-reference-architecture/reference-patterns/rag-architecture-decision-guide.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/rag-architecture-decision-guide.md) — the parallel RAG decision document.
- Temporal, AWS Step Functions, Inngest, Trigger.dev — workflow engines for the workflow shape.
- LangGraph, CrewAI, Vercel AI SDK, raw provider SDK — runtimes for the agent shape.
