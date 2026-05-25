# Agent Engineering

## What this folder is

The engineering practice of shipping and operating multi-step AI agents — loop design, tool architecture, memory, error recovery, partial-failure handling, multi-agent coordination, agent-specific evals, and the on-call disciplines that keep an agent useful rather than expensive. The material here is what I put in front of a team when the question is: *the demo agent worked on three example tasks; production traffic shows it loops for 40 turns on edge cases, calls tools with wrong arguments, occasionally answers from before it had any retrieval — how do we engineer this responsibly?*

## The organizing principle

Agents are the most operationally demanding shape of AI system in 2026. The single LLM call has bounded cost, bounded latency, and one failure mode (the model returns something wrong). The agent has unbounded cost (loops), unbounded latency (slow tools, retries), and dozens of failure modes (wrong tool call, wrong arguments, infinite loop, tool failure, partial completion, lost context, hallucinated intermediate result). It also has the property that the *traces* of its behavior are the only debuggable surface — the same input can produce a different trajectory the next time, so debugging is reading-the-trace, not running-the-step.

So the patterns here treat the agent loop as *a distributed-systems engineering problem*: explicit budgets (turn budget, cost budget, time budget, tool-call budget), explicit failure handling at every boundary (tool failed / tool returned junk / tool returned partial / tool slow), explicit observability (every step, every tool call, every retry, every cost line item), and explicit human-in-the-loop boundaries (where the agent must escalate, not retry).

The folder is opinionated about three things specifically. First, that *most "agent" features should not be agents* — they should be deterministic workflows with one or two LLM steps embedded, because the workflow has bounded cost and is debuggable. Second, that when an agent is the right shape, the loop must have explicit budgets and escalation paths from day one; retrofitting them after a runaway-cost incident is much harder. Third, that the *tool* is the most leveraged piece of agent engineering — a poorly designed tool surface produces a confused agent no matter how good the model is, and a well-designed tool surface lets a smaller / cheaper model perform well above its weight.

## Planned documents

- **[agent-engineering-playbook.md](./agent-engineering-playbook.md)** — Soup-to-nuts playbook on shipping agents in production: the agent-vs-workflow decision, loop design with explicit budgets, tool architecture, memory engineering (short / long / episodic / semantic), error and partial-failure handling, observability requirements, eval strategy, cost-control patterns, on-call disciplines, and the sprint sequencing from prototype to GA. Includes the worked Meridian Care Coordinator example, eight anti-patterns, and sprint-assigned AGT- findings.
- **agent-vs-workflow-decision.md** *(coming)* — The decision tree: when an agent (LLM decides the next step) is the right shape, when a workflow (deterministic plan, LLM steps embedded) is the right shape, and the hybrid pattern (workflow as the skeleton, agent as a contained loop inside one workflow step). The single most consequential agent engineering decision.
- **agent-loop-design.md** *(coming)* — Loop variants (ReAct, plan-then-execute, reflection / self-critique, supervisor / worker), explicit budgets (turn / cost / time / tool-call), termination conditions, the loop-state-as-checkpoint pattern, and the debuggable-loop discipline (every state transition is observable, every decision is logged).
- **tool-architecture.md** *(coming)* — Tool design as a first-class engineering concern: tool naming (the model will pick the tool whose name and description match the intent), argument schemas, error-return shapes, idempotency, side-effect boundaries, the tool-registry pattern, MCP-vs-local-function decision, and the test-the-tool-with-the-agent eval pattern.
- **memory-engineering.md** *(coming)* — Memory taxonomy (short-term context, long-term store, episodic / per-conversation, semantic / cross-conversation), implementation patterns (running summary, structured-fact-extraction, vector-backed conversation index), the retention-and-forgetting policy, and the failure modes (memory drift, hallucinated memory, contradicting memory).
- **error-and-partial-failure.md** *(coming)* — Tool failed (transient vs permanent), tool returned junk (validate inputs, validate outputs, repair-loop, escalate), tool succeeded but the agent misinterpreted, the partial-success handling pattern for multi-step plans, and the rollback / compensating-action discipline for agents that take real-world side effects.
- **multi-agent-coordination.md** *(coming)* — When multiple agents are warranted (rare), supervisor / worker patterns, hand-off vs delegation, shared-state vs message-passing, and the over-engineering trap of building a multi-agent topology when a single-agent loop would have worked.
- **agent-cost-control.md** *(coming)* — Per-request cost budgets, per-tenant cost caps, cost-as-circuit-breaker, the model-tier routing pattern inside an agent loop (cheap orchestrator, expensive specialist), and the operational pattern for an in-progress cost incident — a single misbehaving agent can burn six figures in a weekend.
- **agent-observability.md** *(coming)* — Trace and span design for agents (one trace per top-level request, one span per loop turn, one sub-span per tool call), the LangSmith / Braintrust / Phoenix integration patterns, the alert design for "agent went into a loop" and "agent burning unexpected cost," and the runbook integration.
- **agent-evals.md** *(coming)* — Evaluating agents (much harder than evaluating single-call LLMs): trajectory eval, step-level eval, outcome-only eval, tool-call accuracy eval, cost-aware sampling, the "regression suite from real production traces" pattern, and the integration with the sibling `eval-engineering/` folder.
- **agent-anti-patterns.md** *(coming)* — The eight agent engineering anti-patterns I see most often: "agent for everything," no-turn-budget, no-cost-budget, tools-with-side-effects-without-idempotency, memory-that-is-actually-just-context-stuffing, retry-everything-on-error, no-trajectory-observability, and human-in-the-loop-as-rubber-stamp.

## How to use this section

**If you are about to ship your first agent**, read [agent-engineering-playbook.md](./agent-engineering-playbook.md) end-to-end and `agent-vs-workflow-decision.md` second. Many "agent" projects should not be agents; the decision deserves explicit examination before the engineering investment begins.

**If your agent is in production and occasionally burns unexpected cost**, `agent-cost-control.md` is the urgent reading and `agent-observability.md` is the prerequisite — you cannot put budgets on a system you cannot see.

**If your agent occasionally takes wrong actions that are hard to debug**, `tool-architecture.md` and `agent-observability.md` together describe the diagnostic surface. The fix is usually a tool-surface refactor and a better trace, not a smarter model.

## What this section is not

- **An agent-framework comparison.** LangGraph, CrewAI, AutoGen, Vercel AI SDK, raw SDK — each has its own opinionated take. The engineering patterns here apply across them. Where framework choice matters operationally (observability integration, prompt versioning), it is noted.
- **An agent-security playbook.** Prompt injection in agent loops, MCP server hardening, agent permission models, and agent-specific threat modeling live in the sibling [ai-security-reference-architecture](https://github.com/jeremiahredden/ai-security-reference-architecture)'s `agent-security/` folder.
