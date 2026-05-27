# Eval of Agents

> **Audience.** Engineers writing the eval suite for an agent system (an LLM that reasons across multiple steps, calls tools, maintains state). Tech leads whose first agent eval was a copy-paste of their single-call eval pattern, and who learned at incident time that the patterns don't transfer. Anyone who has stared at a passing trajectory and a failing outcome on the same case and wondered which one to trust. **Scope.** The *engineering* practice of evaluating multi-step agents: trajectory eval (was the path right?), step-level eval (was each step right?), outcome-only eval (did the user-visible result work?), tool-call accuracy eval, and the cost-aware sampling strategies that keep the suite affordable when each case costs real money to run. Pair with [agent-engineering/agent-evals.md](../agent-engineering/agent-evals.md) (the agent-engineering-side practice this connects to) and [golden-set-design.md](./golden-set-design.md) (the underlying case-design discipline). Cross-link to [agent-engineering/agent-loop-design.md](../agent-engineering/agent-loop-design.md) (the loop shape the eval has to cover) and [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) (the trace data the eval reads). **Worked client.** Meridian Health.

---

## 1. Why this document exists

Eval of single LLM calls is a tractable engineering problem: pin a prompt, run it against a curated set of inputs, score the outputs against a rubric, aggregate to a pass rate. The patterns in [eval-engineering-playbook.md](./eval-engineering-playbook.md), [golden-set-design.md](./golden-set-design.md), and [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) cover it.

Eval of agents is harder. An agent is a loop: read state → decide → act (possibly call a tool) → read result → decide → act → ... → produce an outcome. Every dimension of "did this work?" multiplies. Was the *outcome* right? Was the *trajectory* through the loop reasonable? Was each *step* well-formed? Were the right *tools* called with the right arguments? Did the agent waste steps? Did it spend too much? Did it eventually succeed only after retrying past a sensible bound?

The most common failure mode I see is teams adopting one eval dimension and assuming it covers the others. Outcome-only eval misses agents that "got there" through a wildly wasteful path that will hurt cost and latency at scale. Trajectory eval misses agents whose perfect-looking steps produced a wrong answer because of a bad tool result. Step-level eval misses the agent that made locally-correct decisions that compose into a poor outcome. The patterns in this document are about choosing the right dimensions for the workload and combining them so the eval signal is real.

Agent evals are also expensive. A single agent run can be ten or more LLM calls, with tool calls in between. Running a 1000-case eval suite at agent-call cost is a real bill — often $100s per run. The cost-aware sampling and short-circuit-on-failure patterns are not optional for sustainable eval at production scale.

The honest framing: agent evals require three or four distinct eval dimensions running in concert, each catching what the others miss. Building this up takes investment — a team's first agent eval suite often takes a sprint to design and two sprints to populate. Done well, it catches a class of regressions that single-call evals never see: trajectory regressions that don't change the outcome but blow up cost; tool-call regressions that work today and break next month when the tool's response shape changes; step regressions that compose into outcome regressions only on edge cases.

This document is opinionated about four things:

1. **Outcome-only eval is insufficient for production agents.** It passes too often (the agent got lucky on a bad path) and fails too often (the outcome was fine but a stale rubric flagged it). Pair it with at least one of trajectory or step-level eval.
2. **Tool-call accuracy is its own dimension.** The wrong tool with the right arguments is wrong; the right tool with the wrong arguments is wrong; the right tool with the right arguments at the wrong time is wrong. Eval each separately.
3. **Cost-aware sampling is mandatory at scale.** Full agent eval on every PR is unaffordable. Stratified sampling, short-circuit-on-first-failure, and cached-trajectory replay are the cost levers.
4. **Trajectory eval reads traces, not outputs.** Without instrumentation ([observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md)), trajectory eval is impossible. Instrumentation is a prerequisite, not a nice-to-have.

Structure: (2) the four eval dimensions; (3) outcome-only eval; (4) trajectory eval; (5) step-level eval; (6) tool-call accuracy eval; (7) cost-aware sampling strategies; (8) combining the dimensions; (9) worked Meridian example; (10) anti-patterns; (11) findings; (12) adoption checklist; (13) references.

---

## 2. The four eval dimensions

Agent evals have four eval dimensions. Each catches what the others miss.

### 2.1 Outcome eval

The final user-visible result. Did the user get what they needed? Did the system produce a correct answer, complete the booking, draft the email, file the ticket?

- **Pros.** Closest to user-visible quality. Easiest to define. The metric the product owner cares about.
- **Cons.** Slow signal — only available at the end of a multi-step run. Doesn't decompose: an outcome failure could be from any step. Misses wasted-effort regressions where the outcome is fine but the path was 3x too long.

### 2.2 Trajectory eval

The sequence of steps the agent took. Was the path through the loop reasonable? Did it call the right tools in the right order? Did it converge in a sensible number of steps? Did it avoid known anti-patterns (infinite loops, redundant tool calls, premature termination)?

- **Pros.** Catches efficiency regressions. Decomposable — points at which steps deviated. Independent of outcome (a trajectory can be wrong even if the outcome happened to be right).
- **Cons.** Requires a notion of "correct trajectory" — sometimes ambiguous when many paths are valid. Reads from traces; requires instrumentation.

### 2.3 Step-level eval

Each individual step (each LLM call, each tool call, each decision). Was the step well-formed? Did it produce valid output? Did it make a reasonable decision given the state at that point?

- **Pros.** Fastest signal — fails before the full trajectory completes. Pinpoints the failing step. Catches subtle regressions in a single step that don't show up in trajectory or outcome eval.
- **Cons.** Locally-correct decisions can compose into poor outcomes. A step pass doesn't guarantee outcome pass.

### 2.4 Tool-call accuracy eval

Specifically the tool-calling behavior: was the right tool selected? Were the arguments correct? Did the agent handle the tool's response correctly?

- **Pros.** Catches the largest single class of agent regressions in production. Independent of LLM-generation quality.
- **Cons.** Requires per-tool eval cases. Tools change shape over time; eval has to keep up.

### 2.5 The composition

| Dimension | Catches | Misses |
|---|---|---|
| Outcome | User-visible failure | Wasted-effort regressions, path-was-wrong regressions |
| Trajectory | Wrong-path regressions, efficiency drift | Locally-broken steps that happen to converge |
| Step-level | Per-step quality drift | Composition failures across steps |
| Tool-call | Tool-selection/argument errors | Reasoning errors between tool calls |

A production agent suite has at least outcome + one of trajectory / step-level + tool-call. The choice between trajectory and step-level depends on the agent shape (see §8.4).

---

## 3. Outcome-only eval

The simplest agent eval. Run the agent end-to-end; score the final outcome.

### 3.1 The shape

For each eval case:

- Inputs: the user query, any seeded context, the initial state.
- The agent runs (multiple LLM calls, multiple tool calls).
- Output: whatever the agent's final action / response is.
- A judge (LLM or rule-based) scores the output against the expected outcome.

### 3.2 What outcome eval is good for

- **Bootstrapping** the eval suite. Outcome is the easiest dimension to define cases for.
- **Product-owner-readable** signal. "60% of cases produce a correct final answer" maps to product reasoning.
- **Cross-cutting regression catch.** If the outcome stops working, *something* regressed; it's a sentinel.

### 3.3 What outcome eval misses

- **Path quality.** An agent that takes 25 steps to do what should take 4 produces the right outcome at 6x the cost.
- **Trajectory regressions that haven't surfaced in outcome yet.** A new prompt may shift the agent toward a worse path that still works *today*; tomorrow, when a tool changes shape or a user input shifts, the path will break.
- **Partial credit.** An outcome that's *almost* right gets the same fail score as an outcome that's completely wrong. The diagnostic is poor.

### 3.4 The "always run outcome eval" rule

Even when other dimensions are the primary signal, run outcome eval as a sentinel. If outcome regresses, you know to investigate.

---

## 4. Trajectory eval

The path the agent took through the loop.

### 4.1 What trajectory means

A trajectory is the sequence of (state, decision, action, result) tuples that the agent traversed:

```
[
  {state: "fresh query", decision: "search clinical corpus", action: "tool:search", result: "5 docs"},
  {state: "5 docs retrieved", decision: "synthesize answer", action: "llm:draft", result: "draft text"},
  {state: "draft text", decision: "verify citations", action: "tool:verify_citations", result: "all valid"},
  {state: "verified", decision: "respond to user", action: "llm:final_response", result: "user response"}
]
```

The trajectory has length (number of steps), shape (which tools, in what order), and depth (any recursion, any retries).

### 4.2 Two patterns for trajectory eval

**Reference-trajectory eval.** For each case, the eval defines an "expected trajectory" — the canonical good path. The agent's actual trajectory is compared to it.

- Pros: precise comparison; easy to flag deviations.
- Cons: many cases have multiple valid trajectories; comparing exactly is over-restrictive; defining the reference for each case is work.

**Trajectory-rubric eval.** No reference; an LLM-judge evaluates the trajectory against a rubric (e.g., "Is the trajectory free of redundant tool calls? Did the agent converge in ≤ 6 steps? Did it use appropriate tools for the query type?").

- Pros: handles multiple valid trajectories naturally.
- Cons: judge variance; rubric design takes iteration.

Most production agent suites use trajectory-rubric eval for flexibility, with reference-trajectory comparison reserved for cases where the optimal path is well-defined.

### 4.3 What trajectory eval checks

- **Length.** Did the agent converge in a sensible number of steps?
- **Tool selection.** Did it call the right tools?
- **Tool order.** Did it call them in a sensible order? (Verify-before-respond rather than respond-then-verify.)
- **Loop avoidance.** Did it avoid calling the same tool repeatedly with the same arguments?
- **Convergence.** Did it terminate cleanly, or did it bounce around before terminating?
- **State coherence.** Did its decisions follow from the state at each step?

### 4.4 The instrumentation requirement

Trajectory eval reads traces, not just outputs. Without [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) capturing each step's `(state, decision, action, result)` tuple, trajectory eval is impossible.

The trace shape:

- Per-step span with `step_type`, `inputs`, `outputs`, `tool_name` (if applicable), `cost`, `latency`.
- Trace-level metadata: `total_steps`, `total_cost`, `total_latency`, `final_outcome`.

Traces from production are also reusable as eval data — the next section's pattern.

### 4.5 Trajectory replay from production

A powerful pattern: production traces become eval cases. The agent's production run on a known query is replayed offline against a candidate version; the candidate's trajectory is compared to the production trajectory.

- Cheap to collect (no manual case design).
- Reflects the production traffic distribution.
- Catches regressions in cases the curated eval suite doesn't cover.

The catch: production data has PII / PHI; the replay store needs the same access controls as production.

---

## 5. Step-level eval

Per-step evaluation. The fastest signal in agent eval.

### 5.1 The shape

For each step in a trajectory, score it independently:

- **LLM call step.** Did the LLM produce a well-formed decision? Did it select a sensible action? Did its reasoning chain make sense?
- **Tool call step.** Did the agent call the right tool? Right arguments? Right interpretation of the tool's result?
- **Decision step.** Was the agent's branch decision (retry vs proceed, escalate vs answer) correct given the state?

### 5.2 What step-level eval is good for

- **Fast feedback.** A step-level fail can stop the eval immediately; you don't run the full trajectory.
- **Precise diagnosis.** The failing step is pinpointed.
- **Per-prompt eval.** When the agent uses multiple prompts (supervisor, drafter, classifier), step-level eval can score each prompt independently.

### 5.3 What step-level eval misses

- **Composition failures.** All steps may pass individually but the trajectory is still poor.
- **Context-aware quality.** A step that is locally correct may be wrong given upstream decisions.

### 5.4 Step-level rubrics

Define a rubric per step type:

- **Reasoning step.** Is the chain-of-thought coherent? Does it use the available context? Does it select an action consistent with the reasoning?
- **Tool-call step.** Did it select an available tool? Did it format arguments correctly? Did it interpret the response correctly on the *next* step?
- **Termination step.** Did the agent correctly decide to stop?

Step-level rubrics are typically scored by LLM judge ([llm-as-judge-patterns.md](./llm-as-judge-patterns.md)).

### 5.5 Step-level eval cost

Step-level eval requires a judge call per step. For an agent that takes 10 steps per case across 1000 cases = 10,000 judge calls per eval run. The cost is meaningful; sample (see §7).

---

## 6. Tool-call accuracy eval

The dimension I see regress most often in production. Eval it separately.

### 6.1 The three sub-dimensions

**Tool selection.** Did the agent call the right tool for the situation?

- Eval case: given a state, what tool should be called?
- Score: did the agent's chosen tool match?

**Argument accuracy.** Did the agent pass the right arguments?

- Eval case: for the chosen tool, what arguments should be passed given the state?
- Score: do the arguments match (exact or schema-compatible)?

**Result interpretation.** Did the agent correctly interpret the tool's response?

- Eval case: given a tool result, how should the agent proceed?
- Score: did the agent's next action match the expected interpretation?

### 6.2 The pattern

Construct eval cases that target each sub-dimension specifically. The case includes:

- The state when the tool-call decision is made.
- The expected tool name.
- The expected arguments (schema-level if exact match is brittle).
- For result-interpretation: a stubbed tool result and the expected agent response to it.

### 6.3 The "isolate the step" technique

To eval tool-call decisions in isolation, the eval harness pre-loads the agent state and asks for *just* the tool-call decision. The agent doesn't run the full trajectory; it makes one decision; the eval scores it.

This is fast and cheap (one LLM call per case rather than ten). It catches tool-selection regressions specifically.

### 6.4 What changes when tools change

When a tool's API changes (new argument, removed argument, different response shape), the tool-call eval cases must update. The discipline:

- Tool definitions in a registry; tool versions tracked.
- Eval cases reference the tool version.
- When the tool changes, eval cases are reviewed and updated.

Without this discipline, tool-call eval drifts out of sync with reality and either fails on everything or passes on nothing.

### 6.5 The tool-stub vs real-tool decision

For tool-call eval, do you call the real tool or stub it?

- **Real tool:** the eval reflects production behavior including the tool's actual responses.
- **Stub:** the eval is deterministic; the agent's tool-call behavior is isolated from tool-quality variance.

For dev-loop eval, prefer stubs (fast, deterministic). For pre-production eval, run against real tools at least in a release-candidate pass.

---

## 7. Cost-aware sampling strategies

Agent eval is expensive. Cost-aware patterns are mandatory.

### 7.1 The cost shape

A single agent run cost example for Care Coordinator:

- 8 LLM calls average per trajectory.
- ~5K tokens per call average.
- $0.08 per call at frontier pricing.
- ~$0.64 per agent run.

For a 1000-case eval suite: ~$640 per run. Run nightly + per release candidate = ~$30K/month if uncontrolled.

### 7.2 Stratified sampling

Don't run the full suite on every PR. Sample:

- **Fast subset (50–100 cases):** runs per PR. Stratified across query types. ~$60 per run.
- **Full suite (1000+ cases):** runs nightly and per release candidate. ~$640 per run.
- **Extended suite (5000+ cases):** runs weekly. ~$3000 per run.

The fast subset is where engineers see the signal; the full suite catches what the subset missed.

### 7.3 Short-circuit on critical-case failure

If a critical case fails early in the run, abort the run. Don't burn through 900 more cases when one critical case has already failed.

- Critical cases run first.
- On failure: short-circuit; report the critical failure; don't run the rest.
- Non-critical cases continue if all critical pass.

### 7.4 Cached-trajectory replay

For deterministic-ish agents (low temperature, stable tools), the trajectory is reproducible. Cache trajectories from prior eval runs:

- For each case, the canonical trajectory is cached.
- On the next eval, if the candidate's behavior would produce the same trajectory (check via short-circuit on first divergence), short-circuit.
- Only run full trajectories on cases that diverge from cache.

The cache hit rate determines the savings; in practice 50–80% of cases trajectory-match prior runs and can be short-circuited.

### 7.5 Step-level eval with sampling

Step-level eval doesn't have to score every step. Sample:

- Score every step in critical cases.
- Score 10% of steps in non-critical cases.
- Score 100% of tool-call steps (cheap relative to LLM steps; high signal).

### 7.6 Cheap-judge / expensive-judge cascading

Use a cheap LLM judge for first-pass scoring. Escalate to a more expensive judge only on borderline cases.

- 95% of cases: cheap judge confirms pass or fail. Done.
- 5% of cases (borderline scores): escalate to expensive judge for second opinion.

The savings: ~80% reduction in judge cost with negligible quality loss.

### 7.7 Budget the suite

The eval suite has a budget. Per-PR fast subset, per-release full suite, weekly extended. Each is sized to fit the budget. Pruning is intentional; new cases displace old ones if the budget can't grow.

---

## 8. Combining the dimensions

Which dimensions to run, and how to combine their signals.

### 8.1 The minimum production set

For any production agent: at least *outcome + tool-call*. Outcome catches user-visible regressions; tool-call catches the most-common source of agent regressions.

### 8.2 Add trajectory or step-level when

- **Trajectory:** when the agent has multiple valid paths and you care about path quality / cost / latency.
- **Step-level:** when the agent has many independent prompts that need per-prompt eval, or when fast feedback is the primary need.

For most agents: outcome + tool-call + trajectory. Step-level is added when the agent's complexity warrants it.

### 8.3 The gating decision

A case passes overall if:

- Outcome judge: pass.
- Trajectory judge: pass.
- All tool-call sub-evals: pass.
- All sampled step-level evals: pass.

A case fails overall if any dimension fails. The diagnostic includes which dimension failed.

### 8.4 The diagnostic priority

When debugging a regression: tool-call first (catches the most common cause), then trajectory (catches efficiency regressions), then step-level (catches subtle per-prompt drift), then outcome (catches what the others missed).

### 8.5 Per-dimension thresholds

Each dimension has its own pass-rate threshold:

| Dimension | Threshold delta from baseline | Block-deploy |
|---|---|---|
| Outcome | -1pp | -3pp |
| Trajectory | -1pp | -5pp |
| Step-level (sampled) | -0.5pp | -2pp |
| Tool-call accuracy | 0pp | -1pp |

Tool-call gets the tightest threshold because regressions there propagate everywhere.

---

## 9. Worked Meridian example: Care Coordinator agent eval

The Care Coordinator's supervisor agent orchestrates multiple sub-agents: clinical-knowledge retrieval, drafting, refusal-detection, escalation-handling. Its trajectory is: classify → retrieve → draft → verify → respond (or escalate).

### 9.1 The eval suite shape

- **Outcome eval:** 800 curated cases, each with an expected user-visible response.
- **Trajectory eval:** trajectory rubric across all 800 cases. Reference trajectory for 50 critical cases.
- **Step-level eval:** sampled — 100% on critical cases, 10% on others.
- **Tool-call accuracy eval:** 200 dedicated cases targeting tool selection / arguments / result interpretation.

### 9.2 The fast subset (per-PR)

- 50 cases from the outcome suite, stratified by query type.
- 20 cases from the tool-call suite.
- ~7 minutes runtime, ~$15 cost.

### 9.3 The full suite (per release candidate, nightly)

- All 800 outcome cases.
- All 200 tool-call cases.
- Trajectory rubric on all 1000.
- Step-level on critical cases + 10% sample.
- ~3 hours runtime, ~$680 cost.

### 9.4 A regression caught by tool-call eval

A PR changed the supervisor's system prompt to be more concise. Outcome eval passed (the model still produced reasonable answers). Trajectory eval flagged: average trajectory length increased from 4.2 steps to 6.1 steps. Step-level eval flagged: the classifier step had become less specific, leading to broader retrievals.

Tool-call eval caught the root cause: the supervisor was calling the broader-scope retrieval tool more often instead of the targeted clinical-knowledge tool. The system prompt change had de-emphasized the tool-selection guidance.

Without tool-call eval, the team would have seen "trajectory got longer" but not why. With tool-call eval, the diagnostic was direct: tool-selection regressed; fix the system prompt.

### 9.5 Cost-aware execution in action

- Cached-trajectory replay short-circuited 60% of cases in the next eval run (the prompt fix returned trajectories to cached shape).
- Cheap-judge first pass eliminated ~85% of cases from expensive-judge scoring.
- Total cost reduction vs naive full-run: ~70%.

### 9.6 Findings closed

- **ARCH-CARE-094** (agent eval was outcome-only; trajectory regressions invisible).
- **ARCH-CARE-095** (no tool-call accuracy eval; tool-selection regressions caught only in production).
- **ARCH-CARE-096** (no cost-aware sampling; eval cost ran 4x what it could).
- **ARCH-CARE-097** (no per-dimension thresholds; eval signal was binary pass / fail).
- **ARCH-CARE-098** (cached-trajectory replay not implemented; full re-evaluation on every run).

---

## 10. Anti-patterns

### 10.1 The outcome-only eval

Team builds eval that scores only the final output. Production agent has a series of regressions that don't quite break outcome but blow up cost and latency. Finance flags the cost three months in; the team can't decompose because their eval was outcome-blind.

The fix: outcome + at least one of trajectory / tool-call.

### 10.2 The exact-trajectory rigidity

Eval defines a single expected trajectory per case and fails on any deviation. The agent has multiple valid paths; eval fails on every minor variation. Engineers learn to ignore the eval.

The fix: trajectory-rubric for most cases; reference-trajectory only where the canonical path is uniquely correct.

### 10.3 The "tools aren't called consistently" excuse

Team observes tool-call accuracy is noisy and decides not to gate on it. The "noise" is actually real regression that compounds over time.

The fix: tighten the prompt-side discipline so tool-calling is consistent. Tool-call eval is the forcing function.

### 10.4 The judge of the agent's judge

Step-level eval uses an LLM judge to score each step. The judge itself is an LLM that can drift. No one calibrates the judge.

The fix: judge calibration per [llm-as-judge-patterns.md §6](./llm-as-judge-patterns.md). Quarterly re-anchoring.

### 10.5 The full-suite-on-every-PR cost surprise

Team runs full agent eval on every PR. The quarterly bill is much larger than projected; finance flags; the eval gets reduced to "run when someone remembers."

The fix: fast subset per PR; full suite per release candidate. Cost-aware patterns.

### 10.6 The replay-without-PII-controls

Team builds production-trace replay for eval expansion. Production traces contain PHI; the replay store has weaker access controls than production. A breach happens.

The fix: replay store has the same access controls and audit logging as production.

### 10.7 The static tool eval

Tool-call eval was written six months ago. Three tools have changed their APIs since. The eval still passes (because the agent uses the new APIs and the eval cases reference the new APIs too — or doesn't, and fails on everything).

The fix: tool versions tracked; eval cases re-validated when tools change.

### 10.8 The all-dimensions-or-nothing rollout

Team decides agent eval needs all four dimensions. Six months later, they're still designing dimension three; meanwhile production has no eval coverage.

The fix: ship outcome + tool-call in week one. Add trajectory and step-level incrementally. Partial coverage beats no coverage.

---

## 11. Findings (sprint-assignable)

| ID | Finding | Severity | Recommendation | Owner |
|---|---|---|---|---|
| EVAL-AGT-001 | Agent eval is outcome-only; trajectory and tool-call dimensions absent | High | Add tool-call eval per §6; trajectory or step-level per §4–§5 | Eval Eng + AI Platform |
| EVAL-AGT-002 | No instrumentation for trajectory eval | High | Implement per [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) | Observability + AI Platform |
| EVAL-AGT-003 | Tool-call accuracy eval missing; tool-selection regressions reach production | High | Build per-tool eval cases per §6; cover selection, arguments, result-interpretation | Eval Eng |
| EVAL-AGT-004 | Cost-aware sampling not in use; eval cost runs 3–5x what it could | High | Stratified sampling per §7.2; cached-trajectory replay per §7.4 | Eval Eng + FinOps |
| EVAL-AGT-005 | Full agent eval runs per PR; cost prohibitive | High | Fast subset (50–100 cases) per PR; full per release candidate per §7.2 | Eval Eng |
| EVAL-AGT-006 | Reference-trajectory comparison used for cases with multiple valid paths; over-restrictive | Medium | Use trajectory-rubric default; reference only where uniquely correct | Eval Eng |
| EVAL-AGT-007 | Tool definitions not versioned; eval drifts from reality | Medium | Tool registry with versions; eval references tool version | AI Platform + Eval Eng |
| EVAL-AGT-008 | Judge calibration not run on agent step-level judges | Medium | Quarterly judge calibration per [llm-as-judge-patterns.md §6](./llm-as-judge-patterns.md) | Eval Eng |
| EVAL-AGT-009 | Production-trace replay store lacks PHI/PII access controls | High | Same controls and audit logging as production | Privacy + AI Platform |
| EVAL-AGT-010 | Critical cases don't run first; full run continues after critical failure | Medium | Short-circuit per §7.3; critical cases first | Eval Eng |
| EVAL-AGT-011 | Cheap-judge / expensive-judge cascading not implemented | Low | Cascading per §7.6 | Eval Eng |
| EVAL-AGT-012 | Per-dimension thresholds undefined; eval signal is binary | Medium | Per-dimension thresholds per §8.5 | Eval Eng + AI Platform |
| EVAL-AGT-013 | Step-level eval scores every step; cost runs 10x higher than needed | Low | Sampled step-level per §7.5 | Eval Eng |
| EVAL-AGT-014 | Trajectory length not tracked; efficiency regressions invisible | Medium | Trajectory length in trace; pass-rate threshold per §8.5 | Observability + Eval Eng |
| EVAL-AGT-015 | Tool-call eval cases stale after tool API changes; eval pass on everything | Medium | Re-validate when tools change per §6.4 | Eval Eng + Tool Owners |
| EVAL-AGT-016 | All-dimensions-required-before-launch pattern; partial coverage avoided | Medium | Ship outcome + tool-call first; add trajectory / step-level incrementally | Eval Eng + AI Platform |
| EVAL-AGT-017 | Diagnostic priority undefined; regression debugging takes too long | Low | Document priority per §8.4 in runbook | Eval Eng + SRE |
| EVAL-AGT-018 | Outcome eval cases re-used as trajectory eval cases without rubric design | Medium | Trajectory rubric distinct from outcome judge | Eval Eng |

---

## 12. Adoption checklist

- [ ] At least two eval dimensions in production: outcome + (trajectory or tool-call).
- [ ] Tool-call accuracy eval covers selection, arguments, result-interpretation per §6.
- [ ] Trajectory eval uses rubric pattern by default; reference-trajectory only where uniquely correct.
- [ ] Step-level eval (when used) samples appropriately; 100% on critical, 10% on others.
- [ ] Per-dimension pass-rate thresholds defined and gated in CI.
- [ ] Instrumentation captures per-step `(state, decision, action, result)` tuples.
- [ ] Fast subset (50–100 cases) per PR; full suite (1000+) per release candidate; weekly extended.
- [ ] Short-circuit on critical-case failure to bound cost on failing runs.
- [ ] Cached-trajectory replay reduces re-evaluation cost on stable cases.
- [ ] Cheap-judge / expensive-judge cascading active.
- [ ] Eval cost budgeted per quarter; fast / full / extended suites sized to budget.
- [ ] Production-trace replay store has same access controls as production.
- [ ] Tool versions tracked; eval cases re-validated on tool change.
- [ ] Judge calibration quarterly per [llm-as-judge-patterns.md §6](./llm-as-judge-patterns.md).
- [ ] Diagnostic priority documented: tool-call → trajectory → step-level → outcome.
- [ ] Partial coverage acceptable; incremental dimension addition over time.

---

## 13. References

**Internal:**

- [eval-engineering-playbook.md](./eval-engineering-playbook.md) — the eval discipline this builds on.
- [golden-set-design.md](./golden-set-design.md) — case-design discipline.
- [llm-as-judge-patterns.md](./llm-as-judge-patterns.md) — judge patterns for trajectory and step-level eval.
- [regression-eval-suites.md](./regression-eval-suites.md) — suite organization.
- [online-eval-and-feedback.md](./online-eval-and-feedback.md) — production signals that feed agent eval.
- [eval-gate-architecture.md](./eval-gate-architecture.md) — gate placement.
- [eval-of-rag.md](./eval-of-rag.md) — RAG-specific eval (agents often include RAG).
- [eval-anti-patterns.md](./eval-anti-patterns.md) — anti-patterns common across eval practices.
- [agent-engineering/agent-loop-design.md](../agent-engineering/agent-loop-design.md) — the agent loop shape eval has to cover.
- [agent-engineering/agent-evals.md](../agent-engineering/agent-evals.md) — agent-engineering-side practice.
- [agent-engineering/tool-architecture.md](../agent-engineering/tool-architecture.md) — tool patterns the eval verifies.
- [agent-engineering/agent-observability.md](../agent-engineering/agent-observability.md) — observability the eval reads.
- [observability-and-telemetry/agent-step-instrumentation.md](../observability-and-telemetry/agent-step-instrumentation.md) — instrumentation prerequisite.
- [observability-and-telemetry/trace-and-span-design.md](../observability-and-telemetry/trace-and-span-design.md) — trace shape.
- [cost-and-finops/cost-attribution.md](../cost-and-finops/cost-attribution.md) — cost signal for cost-aware eval.
- [cicd-and-eval-gates/eval-gate-design.md](../cicd-and-eval-gates/eval-gate-design.md) — the CI-side gate.

**Cross-repo (architecture sibling):**

- [reference-patterns/agent-topologies.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-patterns/agent-topologies.md) — agent topology choices the eval has to handle.
- [reference-systems/meridian-care-coordinator.md](https://github.com/jeremiahredden/ai-architecture-reference-architecture/blob/main/reference-systems/meridian-care-coordinator.md) — the worked-example system.
